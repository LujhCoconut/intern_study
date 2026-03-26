# Mooncake Client-Master 协同机制详解

**日期**: 2026-03-26  
**分析对象**: Mooncake Store 客户端与主控服务的交互机制

---

## 目录
1. [架构概览](#1-架构概览)
2. [Client 如何发现 Master](#2-client-如何发现-master)
3. [连接建立流程](#3-连接建立流程)
4. [核心 RPC 交互](#4-核心-rpc-交互)
5. [高可用（HA）机制](#5-高可用ha机制)
6. [故障切换流程](#6-故障切换流程)

---

## 1. 架构概览

### 1.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Mooncake 集群架构                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Master 层（控制平面）                          │   │
│  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │   │
│  │  │   Master A   │◄──►│   Master B   │◄──►│   Master C   │          │   │
│  │  │   (Leader)   │    │  (Follower)  │    │  (Follower)  │          │   │
│  │  └──────┬───────┘    └──────────────┘    └──────────────┘          │   │
│  │         │                                                          │   │
│  │         │ 通过 etcd 协调 Leader 选举                                  │   │
│  │         ▼                                                          │   │
│  │  ┌──────────────┐                                                  │   │
│  │  │  etcd 集群    │                                                  │   │
│  │  │ (元数据存储)  │                                                  │   │
│  │  └──────────────┘                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    ▲                                        │
│                                    │ gRPC/RPC                                │
│                                    │                                        │
│  ┌─────────────────────────────────┼─────────────────────────────────────┐ │
│  │                      Client 层（数据平面）                             │ │
│  │                                 │                                     │ │
│  │  ┌─────────────┐   ┌────────────┴────────────┐   ┌─────────────┐    │ │
│  │  │  Client 1   │   │      MasterClient        │   │  Client N   │    │ │
│  │  │ (App Node)  │◄──┤  - 连接管理              ├──►│ (App Node)  │    │ │
│  │  │             │   │  - 请求路由              │   │             │    │ │
│  │  │             │   │  - 故障切换              │   │             │    │ │
│  │  └──────┬──────┘   │  - Leader 发现           │   └──────┬──────┘    │ │
│  │         │          └──────────────────────────┘          │          │ │
│  │         │                    │                           │          │ │
│  │         ▼                    ▼                           ▼          │ │
│  │  ┌─────────────────────────────────────────────────────────────┐   │ │
│  │  │              TransferEngine（RDMA/TCP/CXL）                  │   │ │
│  │  │         实际数据传输（绕过 Master，点对点）                   │   │ │
│  │  └─────────────────────────────────────────────────────────────┘   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 职责分离

| 组件 | 职责 | 数据量 | 延迟要求 |
|------|------|--------|---------|
| **Master** | 元数据管理、资源分配、调度决策 | 小（KB 级） | 中（ms 级） |
| **Client** | 请求发起、状态维护、故障处理 | 中 | - |
| **MasterClient** | RPC 封装、连接管理、Leader 发现 | 小 | - |
| **TransferEngine** | 实际数据传输 | 大（GB 级） | 低（us/ms 级） |

---

## 2. Client 如何发现 Master

### 2.1 创建 Client 时的参数

```cpp
// include/client_service.h:78-84
static std::optional<std::shared_ptr<Client>> Create(
    const std::string& local_hostname,           // 本机地址
    const std::string& metadata_connstring,      // 元数据服务地址
    const std::string& protocol,                 // 传输协议
    const std::optional<std::string>& device_names = std::nullopt,
    const std::string& master_server_entry = kDefaultMasterAddress,  // Master 地址
    const std::shared_ptr<TransferEngine>& transfer_engine = nullptr,
    std::map<std::string, std::string> labels = {}
);
```

### 2.2 Master 地址格式

| 模式 | 地址格式 | 示例 | 说明 |
|------|---------|------|------|
| **单点模式** | `ip:port` | `127.0.0.1:50051` | 直接连接指定 Master |
| **HA 模式** | `etcd://host:port;...` | `etcd://192.168.1.1:2379;192.168.1.2:2379` | 通过 etcd 发现 Leader |
| **默认模式** | 空 | `kDefaultMasterAddress` | 使用内置默认值 |

### 2.3 默认地址定义

```cpp
// 默认 Master 地址
const std::string kDefaultMasterAddress = "127.0.0.1:50051";

// RealClient 初始化时的处理
// real_client.cpp:195-282
tl::expected<void, ErrorCode> RealClient::setup_internal(
    const std::string& local_hostname, 
    const std::string& metadata_server,
    // ... 其他参数
    const std::string& master_server_addr) {  // 接收 Master 地址
    
    // 创建底层 Client，传入 Master 地址
    auto client_opt = mooncake::Client::Create(
        this->local_hostname, 
        metadata_server, 
        protocol, 
        device_name,
        master_server_addr,  // 传给 Client
        transfer_engine
    );
}
```

---

## 3. 连接建立流程

### 3.1 完整初始化序列

```cpp
// Client 创建和连接流程

// 步骤 1: 创建 Client 实例（静态工厂方法）
auto client_opt = Client::Create(
    "localhost:12345",           // local_hostname
    "P2PHANDSHAKE",              // metadata_connstring
    "rdma",                      // protocol
    std::nullopt,                // device_names
    "192.168.1.100:50051"        // master_server_entry - 指定 Master
);

// 步骤 2: 构造函数内部初始化
Client::Client(...) {
    // 初始化成员变量
    local_hostname_ = local_hostname;
    metadata_connstring_ = metadata_connstring;
    protocol_ = protocol;
    // MasterClient 默认构造，未连接
}

// 步骤 3: Create 内部调用 ConnectToMaster
auto client = *client_opt;
ErrorCode ec = client->ConnectToMaster(master_server_entry);
if (ec != ErrorCode::OK) {
    return std::nullopt;  // 连接失败
}
```

### 3.2 ConnectToMaster 实现

```cpp
// client_service.cpp (简化示意)
ErrorCode Client::ConnectToMaster(const std::string& master_server_entry) {
    // 检查是否是 HA 模式
    if (master_server_entry.starts_with("etcd://")) {
        // ========== HA 模式 ==========
        // 启动 LeaderCoordinator 后台线程
        leader_coordinator_ = std::make_unique<ha::LeaderCoordinator>(
            master_server_entry);
        
        // 从 etcd 获取当前 Leader
        auto master_view = leader_coordinator_->GetLeaderView();
        if (!master_view) {
            return ErrorCode::INTERNAL_ERROR;
        }
        
        // 连接到 Leader
        current_master_view_ = master_view;
        return master_client_.Connect(master_view->endpoint);
        
    } else {
        // ========== 单点模式 ==========
        direct_master_address_ = master_server_entry;
        return master_client_.Connect(master_server_entry);
    }
}
```

### 3.3 MasterClient 连接

```cpp
// master_client.cpp (简化示意)
class MasterClient {
public:
    ErrorCode Connect(const std::string& address) {
        // 解析地址 ip:port
        auto [ip, port] = parseAddress(address);
        
        // 创建 gRPC 通道（或 coro_rpc 连接）
        channel_ = createChannel(ip, port);
        
        // 测试连接（发送 Ping）
        auto ping_result = Ping();
        if (!ping_result) {
            return ping_result.error();
        }
        
        return ErrorCode::OK;
    }
    
private:
    std::shared_ptr<Channel> channel_;  // RPC 通道
    std::string current_endpoint_;      // 当前连接的 Master
};
```

---

## 4. 核心 RPC 交互

### 4.1 写入流程（两阶段提交）

```
┌──────────┐                              ┌──────────┐
│  Client  │                              │  Master  │
└────┬─────┘                              └────┬─────┘
     │                                         │
     │  1. StartPut(key, config)              │
     │ ──────────────────────────────────────►│
     │     "申请写入 key，需要 2 个副本"         │
     │                                         │
     │     2. 分配副本位置                      │
     │     - 检查权限                          │
     │     - 选择存储节点                       │
     │     - 预留空间                          │
     │                                         │
     │  3. 返回 ReplicaList                   │
     │ ◄──────────────────────────────────────│
     │     [Replica_A, Replica_B]              │
     │                                         │
     │  4. TransferEngine 传输数据              │
     │     - 通过 RDMA 写入 Replica_A          │
     │     - 通过 RDMA 写入 Replica_B          │
     │     （不经过 Master）                    │
     │                                         │
     │  5. EndPut(key, replica_type)          │
     │ ──────────────────────────────────────►│
     │     "Replica_A 写入完成"                 │
     │                                         │
     │  6. 标记副本完成                         │
     │     - 更新元数据状态                      │
     │     - 授予租约                          │
     │                                         │
     │  7. 返回 OK                            │
     │ ◄──────────────────────────────────────│
     │                                         │
```

### 4.2 StartPut（申请阶段）

```cpp
// client_service.cpp
ErrorCode Client::StartBatchPut(std::vector<PutOperation>& ops, 
                                 const ReplicateConfig& config) {
    for (auto& op : ops) {
        // 构造请求
        StartPutRequest request;
        request.key = op.key;
        request.config = config;
        request.client_id = client_id_;
        
        // 调用 Master RPC
        auto result = master_client_.StartPut(request);
        
        if (result) {
            // 保存分配的副本位置
            op.replicas = result->replica_descriptors;
            op.lease_timeout = result->lease_timeout;
        } else {
            // 分配失败
            op.SetError(result.error());
        }
    }
}
```

### 4.3 Master 端的 PutStart

```cpp
// master_service.cpp
auto MasterService::PutStart(const StartPutRequest& request)
    -> tl::expected<StartPutResponse, ErrorCode> {
    
    const std::string& key = request.key;
    const ReplicateConfig& config = request.config;
    
    // 1. 检查是否已存在
    if (metadata_store_.Exists(key)) {
        return tl::unexpected(ErrorCode::OBJECT_ALREADY_EXISTS);
    }
    
    // 2. 分配副本位置
    std::vector<Replica::Descriptor> replicas;
    for (size_t i = 0; i < config.replica_num; ++i) {
        // 选择存储节点（根据 preferred_segments 或负载均衡）
        auto segment = allocator_.SelectSegment(config);
        
        // 创建副本描述符
        Replica::Descriptor desc;
        desc.segment_id = segment.id;
        desc.buffer_address = segment.allocate(size);
        desc.transport_endpoint = segment.endpoint;
        replicas.push_back(desc);
    }
    
    // 3. 创建元数据
    ObjectMetadata metadata;
    metadata.key = key;
    metadata.client_id = request.client_id;
    metadata.replicas = replicas;
    metadata.status = ObjectStatus::PROCESSING;  // 标记为处理中
    metadata.create_time = Now();
    
    // 4. 保存到元数据存储
    metadata_store_.Put(key, metadata);
    
    // 5. 加入处理中集合（用于租约管理）
    processing_objects_.insert(key);
    
    // 6. 返回响应
    StartPutResponse response;
    response.replica_descriptors = replicas;
    response.lease_timeout = Now() + lease_duration_;
    return response;
}
```

### 4.4 EndPut（确认阶段）

```cpp
// master_service.cpp
auto MasterService::PutEnd(const UUID& client_id, 
                           const std::string& key,
                           ReplicaType replica_type)
    -> tl::expected<void, ErrorCode> {
    
    // 1. 获取元数据（读锁保护）
    std::shared_lock<std::shared_mutex> shared_lock(snapshot_mutex_);
    MetadataAccessorRW accessor(this, key);
    
    if (!accessor.Exists()) {
        return tl::make_unexpected(ErrorCode::OBJECT_NOT_FOUND);
    }
    
    auto& metadata = accessor.Get();
    
    // 2. 权限检查（只能结束自己开始的写入）
    if (client_id != metadata.client_id) {
        LOG(ERROR) << "Illegal client " << client_id << " to PutEnd key " << key
                   << ", was PutStart-ed by " << metadata.client_id;
        return tl::make_unexpected(ErrorCode::ILLEGAL_CLIENT);
    }
    
    // 3. 标记指定类型的副本完成
    metadata.VisitReplicas(
        [replica_type](const Replica& replica) {
            return replica.type() == replica_type;
        },
        [](Replica& replica) { 
            replica.mark_complete();  // 标记为 COMPLETED
        });
    
    // 4. 检查是否所有副本都完成
    if (metadata.AllReplicas(&Replica::fn_is_completed)) {
        // 从处理中集合移除
        accessor.EraseFromProcessing();
        
        // 更新状态为可用
        metadata.status = ObjectStatus::AVAILABLE;
    }
    
    // 5. 授予租约（对象现在可被读取）
    metadata.GrantLease(0, default_kv_soft_pin_ttl_);
    
    return {};
}
```

### 4.5 读取流程

```
┌──────────┐                              ┌──────────┐
│  Client  │                              │  Master  │
└────┬─────┘                              └────┬─────┘
     │                                         │
     │  1. Query(key)                         │
     │ ──────────────────────────────────────►│
     │     "查询 key 的位置"                    │
     │                                         │
     │  2. 返回 ReplicaList                   │
     │ ◄──────────────────────────────────────│
     │     [Replica_A(primary), Replica_B]     │
     │                                         │
     │  3. TransferEngine 读取数据              │
     │     - 首选 Replica_A（低延迟）           │
     │     - 失败时 fallback 到 Replica_B      │
     │     （不经过 Master）                    │
     │                                         │
```

---

## 5. 高可用（HA）机制

### 5.1 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         HA 架构                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      etcd 集群                           │   │
│  │  - 存储 Master 元数据                                     │   │
│  │  - Leader 选举协调                                        │   │
│  │  - 服务发现                                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ▲                                  │
│           ┌──────────────────┼──────────────────┐              │
│           │                  │                  │              │
│           ▼                  ▼                  ▼              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  │  Master A   │◄──►│  Master B   │◄──►│  Master C   │       │
│  │  (Leader)   │    │  (Follower) │    │  (Follower) │       │
│  │   写请求    │    │   同步数据   │    │   同步数据   │       │
│  └──────┬──────┘    └─────────────┘    └─────────────┘       │
│         │                                                      │
│         │  gRPC                                                │
│         ▼                                                      │
│  ┌─────────────┐                                               │
│  │   Client    │◄────── 发现 Leader 变更 ─────► 自动切换      │
│  │             │                                               │
│  └─────────────┘                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 组件职责

| 组件 | HA 角色 | 职责 |
|------|---------|------|
| **etcd** | 协调服务 | 存储集群状态、Leader 选举、故障检测 |
| **Master A (Leader)** | 主节点 | 处理所有写请求、元数据变更、调度决策 |
| **Master B/C (Follower)** | 备节点 | 同步 Leader 数据、准备接管、提供读服务 |
| **Client** | 客户端 | 通过 etcd 发现 Leader、自动切换连接 |

### 5.3 Leader 发现流程

```cpp
// client_service.cpp - HA 模式初始化
ErrorCode Client::ConnectToMaster(const std::string& master_server_entry) {
    if (master_server_entry.starts_with("etcd://")) {
        // ===== HA 模式 =====
        
        // 1. 创建 LeaderCoordinator
        leader_coordinator_ = std::make_unique<ha::LeaderCoordinator>(
            master_server_entry);
        
        // 2. 启动 Leader 监控线程
        leader_monitor_running_ = true;
        leader_monitor_thread_ = std::thread(
            &Client::LeaderMonitorThreadMain, this);
        
        // 3. 获取当前 Leader
        auto master_view = leader_coordinator_->GetLeaderView();
        if (!master_view) {
            LOG(ERROR) << "Failed to get initial master view";
            return ErrorCode::INTERNAL_ERROR;
        }
        
        // 4. 连接到 Leader
        current_master_view_ = master_view;
        return master_client_.Connect(master_view->endpoint);
    }
    // ... 单点模式
}
```

---

## 6. 故障切换流程

### 6.1 切换时序图

```
时间 ──────────────────────────────────────────────────────────────►

Client           Master A (Leader)    Master B    etcd    Master C
  │                   │                  │          │         │
  │◄─────────────────正常通信────────────►│          │         │
  │                   │                  │          │         │
  │                   │    宕机           │          │         │
  │                   X                  │          │         │
  │                   │                  │          │         │
  │  RPC 失败 ────────┤                  │          │         │
  │                   │                  │          │         │
  │  查询 etcd Leader │                  │          │◄────────┤
  │  ───────────────────────────────────────────────►│         │
  │                   │                  │          │         │
  │  返回新 Leader ───┤                  │          │         │
  │  ◄───────────────────────────────────────────────│         │
  │                   │                  │          │         │
  │  连接到 Master B ─►│                  │◄─────────┤         │
  │                   │                  │          │         │
  │◄─────────────────恢复通信────────────►│          │         │
  │                   │                  │          │         │
```

### 6.2 Client 端自动切换代码

```cpp
// client_service.cpp
void Client::LeaderMonitorThreadMain() {
    while (leader_monitor_running_) {
        // 1. 等待 Leader 变更（阻塞，有事件才唤醒）
        auto new_view = leader_coordinator_->WaitForLeaderChange(
            *current_master_view_, 
            std::chrono::seconds(5));
        
        if (!new_view) {
            continue;  // 超时，继续监听
        }
        
        // 2. 检查是否真的变更
        if (new_view->endpoint == current_master_view_->endpoint) {
            continue;  // 相同，忽略
        }
        
        LOG(INFO) << "Leader changed from " << current_master_view_->endpoint
                  << " to " << new_view->endpoint;
        
        // 3. 执行切换
        ErrorCode ec = SwitchLeader(*new_view);
        if (ec != ErrorCode::OK) {
            LOG(ERROR) << "Failed to switch leader";
        }
    }
}

ErrorCode Client::SwitchLeader(const ha::MasterView& target_view) {
    std::lock_guard<std::mutex> lock(leader_switch_mutex_);
    
    // 1. 断开旧连接
    master_client_.Disconnect();
    
    // 2. 连接到新 Leader
    ErrorCode ec = master_client_.Connect(target_view.endpoint);
    if (ec != ErrorCode::OK) {
        LOG(ERROR) << "Failed to connect to new leader";
        return ec;
    }
    
    // 3. 更新当前视图
    current_master_view_ = target_view;
    
    // 4. 重新挂载 Segment（如果有必要）
    RemountSegments();
    
    return ErrorCode::OK;
}
```

### 6.3 写入过程中的故障处理

```cpp
// Client 代码中的容错处理
ErrorCode Client::PutWithRetry(const std::string& key, 
                                const std::vector<Slice>& slices) {
    int retry_count = 0;
    const int max_retries = 3;
    
    while (retry_count < max_retries) {
        // 尝试写入
        auto result = PutInternal(key, slices);
        
        if (result) {
            return ErrorCode::OK;  // 成功
        }
        
        if (result.error() == ErrorCode::RPC_FAIL) {
            // Master 可能宕机，尝试切换
            LOG(WARNING) << "RPC failed, attempting leader switch";
            
            // 强制刷新 Leader 信息
            auto new_leader = leader_coordinator_->ForceRefreshLeader();
            if (new_leader && new_leader->endpoint != current_master_view_->endpoint) {
                SwitchLeader(*new_leader);
            }
            
            retry_count++;
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        } else {
            // 其他错误，不重试
            return result.error();
        }
    }
    
    return ErrorCode::RPC_FAIL;
}
```

---

## 7. 总结

| 机制 | 说明 | 关键代码位置 |
|------|------|-------------|
| **Master 发现** | 创建时传入地址，支持单点和 etcd HA 模式 | `Client::Create()`, `ConnectToMaster()` |
| **连接管理** | MasterClient 封装 RPC 连接和重连逻辑 | `master_client.h/cpp` |
| **两阶段写入** | StartPut 申请资源 → 数据传输 → EndPut 确认 | `StartBatchPut()`, `FinalizeBatchPut()` |
| **HA 切换** | etcd 监听 Leader 变更，Client 自动切换 | `LeaderMonitorThreadMain()`, `SwitchLeader()` |
| **租约管理** | Master 授予租约，Client 定期续租防止驱逐 | `GrantLease()`, `RenewLease()` |

**核心设计原则**:
1. **控制平面与数据平面分离**：Master 只处理元数据，实际数据传输 bypass Master
2. **无单点故障**：HA 模式下 Master 可故障切换，数据不丢失
3. **客户端无状态**：Client 可以在任意时刻重连，不影响数据一致性

---

*文档生成时间: 2026-03-26*
*版本: Mooncake v1.x*
