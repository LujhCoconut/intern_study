#   Mooncake + SGLang 四大关键组件代码解析

本文档从源代码层面详细解析 Mooncake 与 SGLang 集成时的四个关键组件：`主服务（Master Service）`、`元数据服务（Metadata Service）`、`存储服务（Store Service）` 和 `SGLang 服务器`。

**基于源码版本：**
- SGLang: `/home/ljh/sglang/python/sglang/srt/mem_cache/storage/mooncake_store/`
- Mooncake Store: `/home/ljh/Mooncake/mooncake-store/`

---

## 目录

1. [架构概览](#架构概览)
2. [核心数据结构与类型](#核心数据结构与类型)
3. [主服务（Master Service）](#主服务master-service)
4. [元数据服务（Metadata Service）](#元数据服务metadata-service)
5. [存储服务（Store Service）](#存储服务store-service)
6. [SGLang 服务器中的 Mooncake 集成](#sglang-服务器中的-mooncake-集成)
7. [组件间交互流程](#组件间交互流程)
8. [关键配置参数](#关键配置参数)

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SGLang Server                                  │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐  │
│  │  HiRadixCache   │───▶│ HiCacheController│───▶│    MooncakeStore        │  │
│  │  (radix_cache)  │    │(cache_controller)│    │  (mooncake_store.py)    │  │
│  └─────────────────┘    └─────────────────┘    └─────────────────────────┘  │
│           │                      │                       │                  │
│           ▼                      ▼                       ▼                  │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐  │
│  │   HostKVCache   │◄───│  Memory Pool    │◄───│    PyClient (C++)       │  │
│  │ (memory_pool_host)   │  Management     │    │  (mooncake-store lib)   │  │
│  └─────────────────┘    └─────────────────┘    └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
           │                                               │
           │ RDMA/TCP                                      │ RDMA/TCP (Zero-Copy)
           ▼                                               ▼
┌─────────────────────┐                         ┌─────────────────────────────┐
│   Metadata Service  │◄───────────────────────▶│      Master Service         │
│ (http_metadata_server)                        │  (WrappedMasterService)     │
│                     │                         │  - PutStart/PutEnd          │
│   - 服务发现         │                         │  - GetReplicaList           │
│   - 连接管理         │                         │  - MountSegment             │
│   - P2P握手         │                         │  - 副本管理/驱逐策略          │
└─────────────────────┘                         └─────────────────────────────┘
                                                           │
                                                           │ RPC (gRPC/coro_rpc)
                                                           ▼
                                                ┌─────────────────────────────┐
│                                               │      Store Service          │
│                                               │  (mooncake store client)    │
│                                               │  ┌─────────────────────┐    │
│                                               │  │   ClientService     │    │
│                                               │  │  - 全局内存段注册      │    │
│                                               │  │  - RDMA传输引擎       │    │
│                                               │  │  - 本地热缓存         │    │
│                                               │  └─────────────────────┘    │
│                                               └─────────────────────────────┘
```

---

## 核心数据结构与类型

### 1. 基础类型定义 (`types.h`)

```cpp
namespace mooncake {

// 对象标识
using ObjectKey = std::string;
using UUID = std::pair<uint64_t, uint64_t>;
using Version = uint64_t;
using SegmentId = int64_t;

// 内存切片
struct Slice {
    void* ptr{nullptr};
    size_t size{0};
};

// 内存段 (存储节点贡献的内存)
struct Segment {
    UUID id{0, 0};
    std::string name{};        // 逻辑段名称
    uintptr_t base{0};         // 内存基地址
    size_t size{0};            // 内存大小
    std::string te_endpoint{}; // Transfer Engine 端点 (ip:port)
    std::string protocol;      // 传输协议 (rdma/tcp)
};

// 副本描述符
struct Replica::Descriptor {
    std::string segment_name;  // 所在段名
    uintptr_t address;         // 段内偏移地址
    size_t size;               // 数据大小
    ReplicaType type;          // 副本类型 (MEMORY/DISK/LOCAL_DISK)
    std::string endpoint;      // 网络端点
    // ...
};

// 错误码定义
enum class ErrorCode : int32_t {
    OK = 0,
    INTERNAL_ERROR = -1,
    BUFFER_OVERFLOW = -10,
    SEGMENT_NOT_FOUND = -101,
    OBJECT_NOT_FOUND = -704,
    OBJECT_ALREADY_EXISTS = -705,
    TRANSFER_FAIL = -800,
    RPC_FAIL = -900,
    // ...
};
}
```

### 2. RPC 类型定义 (`rpc_types.h`)

```cpp
// Master 服务响应结构
struct PingResponse {
    ViewVersionId view_version_id;  // 集群视图版本
    ClientStatus client_status;     // 客户端状态 (OK/NEED_REMOUNT)
};

struct GetReplicaListResponse {
    std::vector<Replica::Descriptor> replicas;  // 副本列表
    uint64_t lease_ttl_ms;                      // 租约过期时间
};

struct BatchGetOffloadObjectResponse {
    uint64_t batch_id;              // 批量传输ID
    std::vector<uint64_t> pointers; // 远程内存指针
    std::string transfer_engine_addr;
    uint64_t gc_ttl_ms;
};

// 任务相关
struct TaskAssignment {
    UUID id;
    TaskType type;          // COPY/MOVE
    std::string payload;
    int64_t created_at_ms_epoch;
    uint32_t max_retry_attempts;
};
```

---

## 主服务（Master Service）

### 1. 功能概述

主服务是 Mooncake 分布式系统的中心协调器，核心功能：

| 功能 | 说明 |
|------|------|
| **元数据管理** | 维护对象到副本列表的映射 |
| **段管理** | 管理存储节点注册的内存/磁盘段 |
| **副本放置** | 决定新对象的副本位置 |
| **租约管理** | 为读取操作授予租约，保证数据一致性 |
| **驱逐策略** | 当存储空间不足时触发对象驱逐 |
| **任务调度** | 管理副本复制/迁移任务 |
| **高可用** | 通过 etcd 实现主服务 HA |

### 2. 核心 RPC 接口 (`rpc_service.h`)

```cpp
class WrappedMasterService {
public:
    // 对象生命周期管理
    tl::expected<std::vector<Replica::Descriptor>, ErrorCode> 
        PutStart(const UUID& client_id, const std::string& key, 
                 const uint64_t slice_length, const ReplicateConfig& config);
    tl::expected<void, ErrorCode> PutEnd(const UUID& client_id, 
                                         const std::string& key, 
                                         ReplicaType replica_type);
    tl::expected<void, ErrorCode> PutRevoke(const UUID& client_id, 
                                            const std::string& key);
    
    // 批量操作
    std::vector<tl::expected<std::vector<Replica::Descriptor>, ErrorCode>> 
        BatchPutStart(...);
    std::vector<tl::expected<void, ErrorCode>> 
        BatchPutEnd(...);
    
    // 查询接口
    tl::expected<bool, ErrorCode> ExistKey(const std::string& key);
    std::vector<tl::expected<bool, ErrorCode>> BatchExistKey(
        const std::vector<std::string>& keys);
    tl::expected<GetReplicaListResponse, ErrorCode> GetReplicaList(
        const std::string& key);
    
    // 段管理
    tl::expected<void, ErrorCode> MountSegment(const Segment& segment, 
                                               const UUID& client_id);
    tl::expected<void, ErrorCode> UnmountSegment(const UUID& segment_id, 
                                                 const UUID& client_id);
    
    // 副本复制/迁移任务
    tl::expected<UUID, ErrorCode> CreateCopyTask(
        const std::string& key, const std::vector<std::string>& targets);
    tl::expected<UUID, ErrorCode> CreateMoveTask(
        const std::string& key, const std::string& source, 
        const std::string& target);
    tl::expected<QueryTaskResponse, ErrorCode> QueryTask(const UUID& task_id);
    
    // 心跳检测
    tl::expected<PingResponse, ErrorCode> Ping(const UUID& client_id);
};
```

### 3. Put 操作流程

```
Client (SGLang)                          Master Service
     │                                         │
     │ 1. PutStart(key, size, replica_config)  │
     │ ───────────────────────────────────────▶│
     │                                         │
     │     2. 选择副本位置 (副本放置策略)       │
     │     3. 返回 Replica::Descriptor 列表    │
     │◄────────────────────────────────────────│
     │                                         │
     │ 4. 使用 TransferEngine 写入数据         │
     │    (RDMA 零拷贝传输到存储节点)           │
     │                                         │
     │ 5. PutEnd(key)                          │
     │ ───────────────────────────────────────▶│
     │                                         │
     │     6. 确认写入完成，更新元数据           │
     │◄────────────────────────────────────────│
```

### 4. 副本放置策略

```cpp
// MasterService 内部的副本选择逻辑
class MasterService {
    // 分配策略类型
    AllocationStrategyType allocation_strategy_;
    
    // 选择最佳段来存储副本
    ErrorCode AllocateReplica(const std::string& key, size_t size, 
                              const ReplicateConfig& config,
                              std::vector<Replica::Descriptor>& replicas);
    
    // 驱逐策略
    EvictionStrategy eviction_strategy_;
    ErrorCode EvictObjects(double eviction_ratio);  // 驱逐比例，默认 5%
};

// 配置参数 (来自 types.h)
constexpr double DEFAULT_EVICTION_HIGH_WATERMARK_RATIO = 0.95;  // 95% 触发驱逐
constexpr double DEFAULT_EVICTION_RATIO = 0.05;                 // 每次驱逐 5%
```

### 5. 租约机制

```cpp
struct GetReplicaListResponse {
    std::vector<Replica::Descriptor> replicas;
    uint64_t lease_ttl_ms;  // 租约过期时间 (默认 5 秒)
};

// 客户端必须在租约到期前完成数据传输
// 否则副本可能被标记为无效，触发重新分配
```

### 6. SGLang 中的配置

```python
# python/sglang/srt/mem_cache/storage/mooncake_store/mooncake_store.py

@dataclass
class MooncakeStoreConfig:
    master_server_address: str   # Master RPC 地址 (默认: 127.0.0.1:50051)
    master_metrics_port: int     # Master HTTP 指标端口 (默认: 9003)
    check_server: bool           # 是否检查 Master 健康状态
```

```bash
# 启动 Master 服务
mooncake_master \
    --eviction_high_watermark_ratio=0.95 \
    --enable_http_metadata_server=true \
    --http_metadata_server_port=8080
```

---

## 元数据服务（Metadata Service）

### 1. 功能概述

元数据服务 (`http_metadata_server`) 负责：
- 客户端发现：新节点加入时获取集群信息
- 连接管理：维护客户端与 Transfer Engine 的连接状态
- P2P 握手：支持点对点握手模式 (可选)

### 2. P2P 握手机制

```cpp
// types.h
constexpr const char* DEFAULT_MASTER_SERVER_ADDR = "127.0.0.1:50051";

// ClientService 初始化时的元数据连接
class Client {
    static std::optional<std::shared_ptr<Client>> Create(
        const std::string& local_hostname,
        const std::string& metadata_connstring,  // "P2PHANDSHAKE" 或 URL
        const std::string& protocol,
        const std::optional<std::string>& device_names = std::nullopt,
        const std::string& master_server_entry = kDefaultMasterAddress,
        ...
    );
};
```

当 `metadata_connstring` 设置为 `"P2PHANDSHAKE"` 时：
- 跳过集中式元数据服务
- 客户端直接与目标节点进行 P2P 握手交换地址信息
- 适合小规模部署或网络受限环境

### 3. SGLang 中的配置

```python
# MooncakeStoreConfig
metadata_server: str  # "http://127.0.0.1:8080/metadata" 或 "P2PHANDSHAKE"

# MooncakeTransferEngine
ret_value = self.engine.initialize(
    hostname,
    "P2PHANDSHAKE",  # 使用 P2P 握手
    "rdma",
    device_name if device_name is not None else "",
)
```

---

## 存储服务（Store Service）

### 1. 核心组件：Client (`client_service.h`)

`Client` 是存储服务的核心 C++ 类，负责：
- 全局内存段注册 (`MountSegment`)
- 数据传输 (`TransferEngine`)
- 本地热缓存 (`LocalHotCache`)
- 副本管理

```cpp
class Client {
public:
    // 创建客户端实例
    static std::optional<std::shared_ptr<Client>> Create(
        const std::string& local_hostname,
        const std::string& metadata_connstring,
        const std::string& protocol,
        const std::optional<std::string>& device_names = std::nullopt,
        const std::string& master_server_entry = kDefaultMasterAddress,
        const std::shared_ptr<TransferEngine>& transfer_engine = nullptr,
        std::map<std::string, std::string> labels = {});
    
    // 数据传输接口
    tl::expected<void, ErrorCode> Get(const std::string& object_key, 
                                      std::vector<Slice>& slices);
    tl::expected<void, ErrorCode> Put(const ObjectKey& key, 
                                      std::vector<Slice>& slices, 
                                      const ReplicateConfig& config);
    
    // 批量操作
    std::vector<tl::expected<void, ErrorCode>> BatchGet(...);
    std::vector<tl::expected<void, ErrorCode>> BatchPut(...);
    
    // 段管理
    tl::expected<void, ErrorCode> MountSegment(const void* buffer, size_t size,
                                               const std::string& protocol = "tcp",
                                               const std::string& location = kWildcardLocation);
    tl::expected<void, ErrorCode> UnmountSegment(const void* buffer, size_t size);
    
    // 本地内存注册 (用于 RDMA)
    tl::expected<void, ErrorCode> RegisterLocalMemory(
        void* addr, size_t length, const std::string& location,
        bool remote_accessible = true, bool update_metadata = true);
    
    // 本地热缓存
    ErrorCode InitLocalHotCache();
    bool IsHotCacheEnabled() const { return hot_cache_ != nullptr; }
    bool ShouldAdmitToHotCache(const std::string& key, bool cache_used);
    
private:
    UUID client_id_;                                    // 客户端唯一ID
    std::shared_ptr<TransferEngine> transfer_engine_;   // 传输引擎
    MasterClient master_client_;                        // Master 客户端
    std::shared_ptr<LocalHotCache> hot_cache_;          // 本地热缓存
    std::unique_ptr<CountMinSketch> admission_sketch_;  // 准入计数器
};
```

### 2. Python 客户端接口 (`pyclient.h`)

SGLang 通过 `PyClient` Python 绑定调用 C++ Client：

```cpp
class PyClient {
public:
    // 设置模式
    virtual int setup_real(
        const std::string& local_hostname, 
        const std::string& metadata_server,
        size_t global_segment_size,           // 贡献给集群的内存大小
        size_t local_buffer_size,             // 本地缓冲区大小
        const std::string& protocol,          // "rdma" 或 "tcp"
        const std::string& rdma_devices,      // RDMA 设备名
        const std::string& master_server_addr,
        const std::shared_ptr<TransferEngine>& transfer_engine,
        const std::string& ipc_socket_path) = 0;
    
    // 虚拟客户端模式 (Dummy Client)
    virtual int setup_dummy(size_t mem_pool_size, 
                            size_t local_buffer_size,
                            const std::string& server_address,  // Real Client 地址
                            const std::string& ipc_socket_path) = 0;
    
    // 数据传输 (零拷贝)
    virtual std::vector<int64_t> batch_get_into(
        const std::vector<std::string>& keys,
        const std::vector<void*>& buffers,
        const std::vector<size_t>& sizes) = 0;
    
    virtual std::vector<int> batch_put_from(
        const std::vector<std::string>& keys,
        const std::vector<void*>& buffers,
        const std::vector<size_t>& sizes,
        const ReplicateConfig& config = ReplicateConfig{}) = 0;
    
    // 主机内存池注册 (SGLang HostKVCache 使用)
    virtual int register_buffer(void* buffer, size_t size) = 0;
    virtual int unregister_buffer(void* buffer) = 0;
    
    // 存在性检查
    virtual std::vector<int> batchIsExist(const std::vector<std::string>& keys) = 0;
};
```

### 3. 本地热缓存 (`local_hot_cache`)

```cpp
// Client 中的热缓存成员
std::shared_ptr<LocalHotCache> hot_cache_;
std::unique_ptr<CountMinSketch> admission_sketch_;  // Count-Min Sketch 频率统计
uint8_t admission_threshold_ = 2;  // 准入阈值

// 热缓存逻辑
bool Client::ShouldAdmitToHotCache(const std::string& key, bool cache_used) {
    if (!(hot_cache_ && !cache_used)) return false;
    if (admission_sketch_ == nullptr) return true;  // 无准入控制，直接允许
    
    // 频率 >= 阈值时才准入
    return admission_sketch_->increment(key) >= admission_threshold_;
}

// 读取时优先检查热缓存
bool Client::RedirectToHotCache(const std::string& key, 
                                Replica::Descriptor& replica);
```

### 4. Dummy Client 模式 (虚拟客户端)

用于将 SGLang 与 RDMA/内存管理解耦：

```cpp
// IPC 请求类型
enum IpcRequestType : uint32_t {
    IPC_SHM_REGISTER = 0,    // 注册共享内存
    IPC_SHM_FD_REQUEST = 1,  // 请求共享内存 fd
};

// Dummy Client (SGLang) ◄──IPC/Unix Domain Socket──► Real Client (Store Service)
// SGLang 通过 IPC 将内存注册请求转发给本地 mooncake_client 进程
```

---

## SGLang 服务器中的 Mooncake 集成

### 1. MooncakeStore 类

```python
# python/sglang/srt/mem_cache/storage/mooncake_store/mooncake_store.py

class MooncakeStore(HiCacheStorage):
    def __init__(self, storage_config: HiCacheStorageConfig = None, 
                 mem_pool: HostKVCache = None):
        # 导入 Mooncake Python 库
        from mooncake.store import MooncakeDistributedStore
        self.store = MooncakeDistributedStore()
        
        # 配置加载（优先级从高到低）
        if extra_config is not None:
            self.config = MooncakeStoreConfig.load_from_extra_config(extra_config)
        elif envs.SGLANG_HICACHE_MOONCAKE_CONFIG_PATH.is_set():
            self.config = MooncakeStoreConfig.from_file()
        else:
            self.config = MooncakeStoreConfig.load_from_env()
        
        # 检查 Master 服务健康状态
        if self.config.check_server:
            self.check_server()
        
        # 计算每个 TP rank 贡献的内存
        per_tp_global_segment_size = (
            self.config.global_segment_size // tp_scale_factor
        )
        
        # 初始化 Mooncake 连接
        if self.config.standalone_storage:
            # 虚拟客户端模式 - 连接到本地存储服务
            ret_code = self.store.setup_dummy(
                mem_pool.size * mem_pool.size_per_token,
                DEFAULT_LOCAL_BUFFER_SIZE,
                self.config.client_server_address,
            )
        else:
            # 完整客户端模式 - 直接连接 Master
            ret_code = self.store.setup(
                client_hostname,
                self.config.metadata_server,
                per_tp_global_segment_size,
                DEFAULT_LOCAL_BUFFER_SIZE,
                self.config.protocol,
                device_name,
                self.config.master_server_address,
                transfer_engine,  # 可选：重用已有的 TransferEngine
            )
        
        # 注册主机内存池以启用零拷贝
        if mem_pool_host is not None:
            self.register_mem_pool_host(mem_pool_host)
        
        # 预热
        self.warmup()
```

### 2. 零拷贝传输实现

```python
def register_mem_pool_host(self, mem_pool_host: HostKVCache):
    """
    注册 SGLang 的 HostKVCache 到 Mooncake，启用零拷贝传输。
    这是性能关键路径。
    """
    buffer = self.mem_pool_host.kv_buffer
    buffer_ptr = buffer.data_ptr()
    buffer_size = buffer.numel() * buffer.element_size()
    
    # 调用 Mooncake C++ 接口注册内存
    ret_code = self.store.register_buffer(buffer_ptr, buffer_size)
    if ret_code:
        raise RuntimeError(f"Failed to register buffer to Mooncake Store")

def _put_batch_zero_copy_impl(self, key_strs: List[str], 
                              buffer_ptrs: List[int], 
                              buffer_sizes: List[int]) -> List[int]:
    """
    从 HostKVCache 直接写入 Mooncake，无需中间拷贝。
    
    参数:
        key_strs: 对象键列表
        buffer_ptrs: 主机内存指针列表
        buffer_sizes: 数据大小列表
    
    返回:
        每个键的写入结果 (0 = 成功)
    """
    return self.store.batch_put_from(key_strs, buffer_ptrs, buffer_sizes)

def _get_batch_zero_copy_impl(self, key_strs: List[str], 
                              buffer_ptrs: List[int], 
                              buffer_sizes: List[int]) -> List[int]:
    """
    从 Mooncake 直接读取到 HostKVCache，无需中间拷贝。
    
    返回:
        每个键读取的字节数（负值表示错误）
    """
    return self.store.batch_get_into(key_strs, buffer_ptrs, buffer_sizes)
```

### 3. 键生成逻辑 (MHA vs MLA)

```python
def _get_mha_buffer_meta(self, keys, indices):
    """MHA 模型：每个键生成 _k 和 _v 两个后缀的键"""
    ptr_list, element_size_list = self.mem_pool_host.get_page_buffer_meta(indices)
    key_list = []
    for key_ in keys:
        key_list.append(f"{key_}_{self.mha_suffix}_k")
        key_list.append(f"{key_}_{self.mha_suffix}_v")
    return key_list, ptr_list, element_size_list

def _get_mla_buffer_meta(self, keys, indices):
    """MLA 模型：每个键只生成一个键（KV 合并存储）"""
    ptr_list, element_size_list = self.mem_pool_host.get_page_buffer_meta(indices)
    key_list = []
    for key_ in keys:
        key_list.append(f"{key_}_{self.mla_suffix}_k")
    return key_list, ptr_list, element_size_list
```

### 4. HiCacheController 中的存储线程

```python
# python/sglang/srt/managers/cache_controller.py

class HiCacheController:
    def _start_storage_threads(self):
        """启动存储预取和备份的后台线程"""
        self.prefetch_thread = threading.Thread(
            target=self.prefetch_thread_func, daemon=True
        )
        self.backup_thread = threading.Thread(
            target=self.backup_thread_func, daemon=True
        )
        
        # 通信队列
        self.prefetch_queue = Queue()        # 预取请求队列
        self.backup_queue = Queue()          # 备份请求队列
        self.prefetch_revoke_queue = Queue() # 预取撤销队列
        self.ack_backup_queue = Queue()      # 备份确认队列
        
        self.prefetch_thread.start()
        self.backup_thread.start()
    
    def prefetch_thread_func(self):
        """后台线程：从 L3 存储预取数据到 L2 主机内存"""
        while not self.storage_stop_event.is_set():
            operation = self.prefetch_queue.get(block=True, timeout=1)
            
            # 1. 查询存储命中率
            hash_value, storage_hit_count = self._storage_hit_query(operation)
            
            # 2. TP 同步：取各 rank 最小值
            if self.tp_world_size > 1:
                torch.distributed.all_reduce(
                    storage_hit_count_tensor,
                    op=torch.distributed.ReduceOp.MIN,
                    group=self.prefetch_tp_group,
                )
            
            # 3. 判断是否值得预取
            if storage_hit_count < self.prefetch_threshold:
                self.prefetch_revoke_queue.put(operation.request_id)
                continue
            
            # 4. 执行预取
            self.page_get_func(operation, batch_hashes, batch_host_indices)
    
    def backup_thread_func(self):
        """后台线程：将 L2 主机内存数据备份到 L3 存储"""
        while not self.storage_stop_event.is_set():
            operation = self.backup_queue.get(block=True, timeout=1)
            
            # 批量写入存储
            for batch in chunks(operation.hash_value, self.storage_batch_size):
                success = self.page_set_func(batch_hashes, batch_host_indices)
                if not success:
                    logger.warning("Backup batch failed")
```

---

## 组件间交互流程

### 1. 系统初始化流程

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Master     │    │   Metadata   │    │    Store     │
│  Service     │    │   Service    │    │   Service    │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       │ 1. 启动           │                   │
       │──────────────────▶│ 2. 启动           │
       │                   │──────────────────▶│ 3. 启动
       │                   │                   │
       │                   │                   │ 4. MountSegment()
       │                   │◄─────────────────▶│ 5. 注册内存段
       │◄─────────────────▶│ 6. 连接 Master    │
       │                   │                   │
       ▼                   ▼                   ▼
    [就绪]              [就绪]              [就绪]

SGLang Server 启动:
    │
    ├─ 7. 加载配置 (extra_config / 文件 / 环境变量)
    │
    ├─ 8. 检查 Master 健康状态 (HTTP GET /get_all_segments)
    │
    ├─ 9. MooncakeDistributedStore.setup()
    │      ├─ 连接 Metadata Service (或 P2PHANDSHAKE)
    │      ├─ 注册全局内存段 (global_segment_size)
    │      └─ 建立到 Master 的 RPC 连接
    │
    ├─ 10. 注册 HostKVCache (register_buffer)
    │       └─ RDMA 注册主机内存以启用零拷贝
    │
    └─ 11. 预热 (warmup)
            └─ 测试性写入/读取 4KB 数据
```

### 2. KV 缓存写入流程（L2 → L3）

```
HiRadixCache (SGLang)
    │
    ├─ 1. 驱逐决策 (eviction policy)
    │       └─ 选择被淘汰的节点
    │
    └─ 2. write_backup_storage(node)
            │
            ▼
HiCacheController
    │
    ├─ 3. write_storage(host_indices, token_ids, hash_value)
    │       │
    │       └─ backup_queue.put(operation)
    │
    └─ 4. backup_thread (后台)
            │
            └─ 5. _page_backup(operation)
                    │
                    ▼
MooncakeStore
    │
    ├─ 6. batch_set_v1(keys, host_indices)
    │       │
    │       └─ 7. _batch_preprocess(keys, host_indices)
    │               ├─ 生成带后缀的键 (e.g., "hash_{tp_rank}_{pp_rank}_k")
    │               └─ 计算缓冲区指针和大小
    │
    └─ 8. _put_batch_zero_copy_impl(keys, ptrs, sizes)
            │
            ▼
MooncakeDistributedStore (C++)
    │
    ├─ 9. batch_put_from()
    │       ├─ 调用 Master.PutStart() 获取 Replica::Descriptor
    │       │       └─ Master 选择存储节点和位置
    │       ├─ 使用 TransferEngine 执行 RDMA 写入
    │       │       └─ batch_transfer_sync_write()
    │       └─ 调用 Master.PutEnd() 确认写入
    │
    └─ 10. 返回结果 (0 = 成功)
```

### 3. KV 缓存读取流程（L3 → L2）

```
Request arrives
    │
    ▼
HiRadixCache (SGLang)
    │
    ├─ 1. 匹配前缀缓存
    │       └─ 发现部分命中，需要从存储加载
    │
    └─ 2. prefetch(request_id, host_indices, token_ids)
            │
            ▼
HiCacheController
    │
    ├─ 3. prefetch_queue.put(operation)
    │
    └─ 4. prefetch_thread (后台)
            │
            ├─ 5. _storage_hit_query(operation)
            │       │
            │       └─ storage_backend.batch_exists(batch_hashes)
            │               └─ 调用 Master.BatchExistKey()
            │
            ├─ 6. TP 同步 (all_reduce MIN)
            │       └─ 确保所有 rank 看到相同的命中情况
            │
            ├─ 7. 判断是否值得预取
            │       if storage_hit_count < prefetch_threshold:
            │           revoke prefetch
            │
            └─ 8. _page_transfer(operation)
                    │
                    ▼
MooncakeStore
    │
    ├─ 9. batch_get_v1(keys, host_indices)
    │       │
    │       └─ 10. _get_batch_zero_copy_impl(keys, ptrs, sizes)
    │               │
    │               ▼
    │       MooncakeDistributedStore (C++)
    │               │
    │               ├─ 11. batch_get_into()
    │               │       ├─ 调用 Master.BatchQuery() 获取副本位置
    │               │       ├─ 使用 TransferEngine 执行 RDMA 读取
    │               │       │       └─ batch_transfer_sync_write()
    │               │       └─ 可选：更新本地热缓存
    │               │
    │               └─ 12. 返回读取的字节数
    │
    └─ 13. 通知 SGLang 预取完成
```

### 4. 高可用 (HA) 流程

```
Client                              etcd Cluster                    Master (HA)
  │                                       │                              │
  │ 1. Create()                           │                              │
  │─────────────────────────────────────────────────────────────────────▶│
  │                                       │                              │
  │                                       │◄─────────────────────────────│
  │                                       │ 2. Leader 选举/注册           │
  │                                       │                              │
  │◄─────────────────────────────────────────────────────────────────────│
  │ 3. 获取当前 Leader 地址                │                              │
  │
  │ 4. 建立到 Leader 的连接
  │
  │ [正常运行]
  │
  │ 5. 心跳线程 (Ping)
  │─────────────────────────────────────────────────────────────────────▶│
  │                                       │                              │
  │◄─────────────────────────────────────────────────────────────────────│
  │ 6. view_version_id 变更检测            │                              │
  │
  │ [Leader 故障]
  │
  │ 7. Ping 超时/失败
  │                                       │                              │
  │                                       │ 8. etcd 选举新 Leader          │
  │                                       │                              │
  │◄─────────────────────────────────────────────────────────────────────│
  │ 9. SwitchLeader()                      │                              │
  │    ├─ 重新连接到新 Leader
  │    └─ 可能需要重新挂载段 (NEED_REMOUNT)
```

---

## 关键配置参数

### 1. SGLang 环境变量

```python
# python/sglang/srt/environ.py

class Envs:
    # Mooncake Store 配置路径 (JSON 文件)
    SGLANG_HICACHE_MOONCAKE_CONFIG_PATH = EnvStr(None)
    
    # 主服务地址 (IP:Port)
    MOONCAKE_MASTER = EnvStr(None)
    
    # 虚拟客户端模式下存储服务地址
    MOONCAKE_CLIENT = EnvStr(None)
    
    # 本地主机名
    MOONCAKE_LOCAL_HOSTNAME = EnvStr("localhost")
    
    # 元数据服务地址 (URL 或 "P2PHANDSHAKE")
    MOONCAKE_TE_META_DATA_SERVER = EnvStr("P2PHANDSHAKE")
    
    # 贡献给 Mooncake 的内存大小
    MOONCAKE_GLOBAL_SEGMENT_SIZE = EnvStr("4gb")
    
    # 传输协议
    MOONCAKE_PROTOCOL = EnvStr("tcp")  # 可选: "rdma"
    
    # RDMA 设备名称
    MOONCAKE_DEVICE = EnvStr("")
    
    # Master 指标 HTTP 端口
    MOONCAKE_MASTER_METRICS_PORT = EnvInt(9003)
    
    # 是否检查 Master 服务状态
    MOONCAKE_CHECK_SERVER = EnvBool(False)
    
    # 是否使用独立存储（Dummy Client 模式）
    MOONCAKE_STANDALONE_STORAGE = EnvBool(False)
```

### 2. Mooncake Store 默认配置 (`types.h`)

```cpp
namespace mooncake {

// 段大小
static constexpr size_t DEFAULT_GLOBAL_SEGMENT_SIZE = 1024 * 1024 * 16;  // 16MB
static constexpr size_t DEFAULT_LOCAL_BUFFER_SIZE = 1024 * 1024 * 16;    // 16MB

// 段大小限制
static constexpr size_t MIN_SEGMENT_SIZE = 1024;                          // 1KB
static constexpr size_t MAX_SEGMENT_SIZE = 1024ULL * 1024 * 1024 * 1024;  // 1TB

// 驱逐策略
constexpr double DEFAULT_EVICTION_HIGH_WATERMARK_RATIO = 0.95;  // 95% 触发驱逐
constexpr double DEFAULT_EVICTION_RATIO = 0.05;                 // 每次驱逐 5%

// 租约和 TTL
static constexpr uint64_t DEFAULT_DEFAULT_KV_LEASE_TTL = 5000;   // 5 秒 (毫秒)
static constexpr uint64_t DEFAULT_KV_SOFT_PIN_TTL_MS = 30 * 60 * 1000;  // 30 分钟
static constexpr int64_t DEFAULT_MASTER_VIEW_LEASE_TTL_SEC = 5;  // 5 秒
static constexpr int64_t DEFAULT_CLIENT_LIVE_TTL_SEC = 10;       // 10 秒

// 任务管理
static constexpr uint32_t DEFAULT_MAX_TOTAL_PENDING_TASKS = 10000;
static constexpr uint32_t DEFAULT_MAX_TOTAL_PROCESSING_TASKS = 10000;
static constexpr uint32_t DEFAULT_MAX_RETRY_ATTEMPTS = 10;

// 默认地址
constexpr const char* DEFAULT_MASTER_SERVER_ADDR = "127.0.0.1:50051";
constexpr const char* DEFAULT_PROTOCOL = "tcp";

}
```

### 3. SGLang 启动参数示例

```bash
# 完整部署模式 (SGLang 作为 Mooncake Client)
python -m sglang.launch_server \
    --model-path /path/to/model \
    --enable-hierarchical-cache \
    --hicache-storage-backend mooncake \
    --hicache-mem-layout page_first \
    --hicache-storage-prefetch-policy timeout \
    --hicache-storage-backend-extra-config '{
        "master_server_address": "127.0.0.1:50051",
        "local_hostname": "localhost",
        "metadata_server": "http://127.0.0.1:8080/metadata",
        "global_segment_size": "4gb",
        "protocol": "rdma",
        "device_name": "mlx5_0,mlx5_1",
        "check_server": true
    }'

# 虚拟客户端模式 (SGLang 通过 RPC 连接到本地存储服务)
python -m sglang.launch_server \
    --model-path /path/to/model \
    --enable-hierarchical-cache \
    --hicache-storage-backend mooncake \
    --hicache-storage-backend-extra-config '{
        "standalone_storage": true,
        "client_server_address": "127.0.0.1:50052"
    }'
```

### 4. Mooncake 服务启动命令

```bash
# 1. 启动元数据服务 (可选，如果使用 P2PHANDSHAKE 可跳过)
python -m mooncake.http_metadata_server --port=8080

# 2. 启动 Master 服务
mooncake_master \
    --eviction_high_watermark_ratio=0.95 \
    --enable_http_metadata_server=true \
    --http_metadata_server_port=8080 \
    --port=50051

# 3. 启动存储服务 (Standalone 模式)
python -m mooncake.mooncake_store_service \
    --config=/path/to/config.json \
    --port=8081

# 或者使用 mooncake_client (Dummy Client 模式下的 Real Client)
mooncake_client \
    --global_segment_size=4GB \
    --master_server_address=127.0.0.1:50051 \
    --port=50052
```

---

## 总结

| 组件 | 核心类/文件 | 主要功能 | 关键接口 |
|------|------------|---------|---------|
| **Master Service** | `WrappedMasterService`<br>`rpc_service.h` | 元数据管理<br>副本放置<br>租约管理 | `PutStart/PutEnd`<br>`GetReplicaList`<br>`MountSegment` |
| **Metadata Service** | `http_metadata_server` | 服务发现<br>P2P 握手 | HTTP 端点<br>`P2PHANDSHAKE` |
| **Store Service** | `Client`<br>`client_service.h` | 内存段管理<br>数据传输<br>本地热缓存 | `MountSegment`<br>`Get/Put`<br>`BatchGet/BatchPut` |
| **Python Client** | `PyClient`<br>`pyclient.h` | Python 绑定<br>零拷贝接口 | `setup_real/setup_dummy`<br>`batch_get_into`<br>`batch_put_from` |
| **SGLang 集成** | `MooncakeStore`<br>`mooncake_store.py` | 配置管理<br>键生成<br>存储线程 | `register_mem_pool_host`<br>`batch_get_v1`<br>`batch_set_v1` |

**核心技术特点：**
1. **零拷贝传输**：通过 `register_buffer` 将 HostKVCache 注册到 RDMA，避免数据拷贝
2. **分层存储**：L1 (GPU) → L2 (Host Memory) → L3 (Mooncake Store)
3. **异步预取**：后台线程预取 L3 数据到 L2，隐藏延迟
4. **高可用**：通过 etcd 实现 Master 故障切换
5. **灵活部署**：支持完整客户端和 Dummy Client 两种模式
