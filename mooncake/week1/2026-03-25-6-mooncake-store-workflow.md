# Mooncake-Store 作为 HiCache L3 后端的工作流程与架构意义

**日期**: 2026-03-25  
**分析对象**: Mooncake-store 在 SGLang HiCache 中的角色

---

## 1. 架构定位

### 1.1 三层缓存中的位置

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SGLang HiCache                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  L1 (GPU)                    L2 (Host)                     L3 (Storage)     │
│  ┌──────────┐               ┌──────────┐                ┌──────────────┐   │
│  │ GPU HBM  │ ←──load────   │ CPU DRAM │  ←──prefetch── │ Mooncake     │   │
│  │ Device   │    back       │ Host     │     from       │ Store        │   │
│  │ Pool     │ ──write──→    │ Pool     │ ──backup──→    │ (Distributed)│   │
│  └──────────┘    backup      └──────────┘                └──────────────┘   │
│       ↑                           ↑                            ↑            │
│       │                           │                            │            │
│   hit_count                   protect_host               batch_get_into    │
│   evicted                     host_ref_counter           batch_put_from    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Mooncake-Store 组件                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐      │
│  │  Master Service  │    │ Metadata Service │    │  Global Segment  │      │
│  │  (gRPC :55051)   │    │  (HTTP :18888)   │    │  (Shared Storage)│      │
│  │                  │    │                  │    │                  │      │
│  │ - 全局协调        │    │ - 节点发现        │    │ - 对象存储        │      │
│  │ - 资源分配        │    │ - 元数据查询      │    │ - 多副本管理      │      │
│  │ - 副本管理        │    │ - 健康检查        │    │ - 负载均衡        │      │
│  └──────────────────┘    └──────────────────┘    └──────────────────┘      │
│           │                       │                       │                 │
│           └───────────────────────┴───────────────────────┘                 │
│                               │                                             │
│                               ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Mooncake Python Client                            │   │
│  │  (mooncake.store.MooncakeDistributedStore)                           │   │
│  │                                                                      │   │
│  │  - batch_get_into(keys, buffer_ptrs)  # 从Storage读入Host           │   │
│  │  - batch_put_from(keys, buffer_ptrs)  # 从Host写入Storage           │   │
│  │  - register_buffer(ptr, size)         # 注册零拷贝内存              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心组件职责

| 组件 | 职责 | 对应代码/进程 |
|------|------|--------------|
| **Master Service** | 全局协调中心，管理segment分配和副本放置 | `mooncake_master` (C++ 进程) |
| **Metadata Service** | HTTP服务，提供节点发现和元数据查询 | Master的子服务 (端口18888) |
| **Global Segment** | 分布式存储空间，存储实际的KV数据 | Master管理的共享内存/SSD/远程节点 |
| **Python Client** | SGLang的接口，通过pybind11调用C++ API | `MooncakeStore` 类 |
| **Transfer Engine** | 底层数据传输引擎，支持RDMA/TCP | `TransferEnginePy` (C++) |

---

## 2. 完整工作流程

### 2.1 系统启动阶段

```
Step 1: 启动 Mooncake Master
────────────────────────────────────────────────────────
$ mooncake_master \
    --rpc_port 55051 \
    --enable_http_metadata_server \
    --http_metadata_server_port 18888

Master 启动流程:
1. 初始化 gRPC Server (端口 55051)
2. 启动 HTTP Metadata Server (端口 18888)
3. 初始化 Global Segment 管理器
4. 等待客户端连接

内存中的数据结构:
- segment_map: <segment_id> → <segment_location, size, replicas>
- object_map: <object_key> → <segment_id, offset, size>
- client_map: <client_id> → <address, protocol, capacity>
```

```
Step 2: SGLang 初始化 MooncakeStore
────────────────────────────────────────────────────────
# Python 代码
from mooncake.store import MooncakeDistributedStore

store = MooncakeDistributedStore()
store.setup(
    local_hostname="127.0.0.1",
    metadata_server="http://127.0.0.1:18888/metadata",
    global_segment_size=4GB,
    protocol="tcp",
    master_server_address="127.0.0.1:55051"
)

C++ 层初始化流程:
1. 创建 TransferEngine 实例
2. 连接 Master (gRPC Channel)
3. 通过 Metadata Service 获取集群拓扑
4. 注册本地内存 (registerLocalMemory)
5. 创建本地 Segment (MountSegment)
```

```
Step 3: 注册 Host KV Cache 缓冲区
────────────────────────────────────────────────────────
# SGLang 调用
self.store.register_buffer(buffer_ptr, buffer_size)

作用:
- 锁定内存页 (mlock) - 防止被swap出去
- RDMA 注册 (ibv_reg_mr) - 创建内存区域
- 建立虚拟地址到物理地址的映射

结果:
- buffer_ptr 指向的内存可以直接被 RDMA 访问
- 实现真正的零拷贝传输
```

### 2.2 运行时工作流程

#### 场景 1: L1/L2 Miss → L3 Prefetch

```
触发条件:
- SGLang 处理新请求
- match_prefix() 发现部分 token 在 L1 和 L2 都不存在
- 但这些 token 可能之前被其他请求存储在 L3

Workflow:
┌──────────────────────────────────────────────────────────────────────────┐
│ 1. HiRadixCache 触发 Prefetch                                            │
└──────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 2. 计算需要 Prefetch 的 tokens                                           │
│                                                                            │
│    new_input_tokens = [5, 6, 7, 8, 9]  # 未在 L1/L2 命中                │
│    hash_values = [sha256([5,6]), sha256([7,8]), ...]  # 每 page 的 hash  │
└──────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 3. 分配 Host 内存 (L2)                                                   │
│                                                                            │
│    host_indices = mem_pool_host.alloc(len(new_input_tokens))             │
│    # 返回 Host KV Cache 中的空闲位置                                      │
│    # 如: host_indices = [100, 101, 102, 103, 104]                        │
└──────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 4. 调用 MooncakeStore.batch_get_into()                                   │
│                                                                            │
│    keys = ["hash_5_6", "hash_7_8", ...]  # L3 的 key                      │
│    buffer_ptrs = [                                                        │
│        host_buffer.data_ptr() + 100 * page_size,  # index 100             │
│        host_buffer.data_ptr() + 101 * page_size,  # index 101             │
│        ...                                                                │
│    ]                                                                      │
│    sizes = [page_size, page_size, ...]                                    │
└──────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 5. C++ 层: PyClient::batch_get_into()                                    │
│                                                                            │
│    for each key:                                                          │
│        a. 查询 Master 获取副本位置 (ReplicaDesc)                          │
│           - Master 返回哪些节点有此数据                                   │
│           - 选择最近的副本 (同机架/同节点)                                │
│                                                                            │
│        b. 使用 TransferEngine 读取数据                                    │
│           - 如果是 RDMA: 直接 RDMA READ 到 buffer_ptr                     │
│           - 如果是 TCP:  通过 socket 接收写入 buffer_ptr                  │
│                                                                            │
│    关键点: 数据直接写入 SGLang 的 Host KV Cache，无中间拷贝              │
└──────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 6. 异步完成通知                                                           │
│                                                                            │
│    - 操作提交后立即返回 (非阻塞)                                          │
│    - SGLang 继续处理其他请求                                              │
│    - 定期检查完成状态 (check_prefetch_progress)                           │
│    - 完成后将数据标记为可用，插入 Radix Tree                              │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 场景 2: Write-Through → L3 Backup

```
触发条件:
- 新计算的 KV Cache 在 L1 (GPU)
- hit_count 达到阈值，触发 write_backup()
- Node 被标记为 backuped，需要持久化到 L3

Workflow:
┌──────────────────────────────────────────────────────────────────────────┐
│ 1. HiRadixCache 触发 Write Backup                                        │
└──────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 2. 数据已在 L2 (Host)                                                    │
│                                                                            │
│    node.host_value = [20, 21, 22]  # Host 中的 indices                   │
│    node.hash_value = ["hash_a", "hash_b", "hash_c"]  # L3 keys           │
└──────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 3. 调用 MooncakeStore.batch_put_from()                                   │
│                                                                            │
│    keys = ["hash_a_0_0_k", "hash_a_0_0_v",  # K/V 分开存储               │
│             "hash_b_0_0_k", "hash_b_0_0_v",                               │
│             ...]                                                          │
│    buffer_ptrs = [                                                        │
│        host_buffer.data_ptr() + 20 * bytes_per_page,                      │
│        host_buffer.data_ptr() + 21 * bytes_per_page,                      │
│        ...                                                                │
│    ]                                                                      │
└──────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 4. C++ 层: PyClient::batch_put_from()                                    │
│                                                                            │
│    a. 向 Master 请求写入位置                                              │
│       - Master 选择最佳的 Segment (负载均衡)                              │
│       - 决定副本数量 (通常 1-3 副本)                                      │
│                                                                            │
│    b. 并行写入多副本                                                      │
│       - Primary: 本地 Segment 或最近的远程 Segment                        │
│       - Replicas: 其他节点的 Segment (容灾)                               │
│                                                                            │
│    c. 使用 TransferEngine 传输数据                                        │
│       - RDMA WRITE 或 TCP send                                            │
│       - 直接从 Host Buffer 读取 (零拷贝)                                  │
│                                                                            │
│    d. 等待写入确认                                                        │
│       - 所有副本确认后返回成功                                            │
└──────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 5. 更新元数据                                                             │
│                                                                            │
│    Master 更新:                                                           │
│    - object_map: <hash_a> → <segment_id, offset, size>                    │
│    - segment_map: 更新使用统计                                            │
│                                                                            │
│    SGLang 更新:                                                           │
│    - node.backuped = True  # 标记为已备份                                 │
│    - 可以安全地从 Host 驱逐 (如果需要)                                    │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 场景 3: 跨请求 KV Cache 复用

```
场景描述:
- 请求 A 计算了前缀 "你好，请问" 的 KV，存储到 L3
- 请求 B 来了，有相同前缀 "你好，请问机器学习..."

传统方式 (无 L3):
├── 请求 A: 计算 "你好，请问" → 存储在 GPU/Host → 请求结束 → 驱逐
├── 请求 B: 重新计算 "你好，请问" → 重复工作
└── 效率低下

使用 Mooncake L3:
├── 请求 A: 
│   ├── 计算 "你好，请问" 的 KV
│   ├── 存储在 L1 (GPU)
│   ├── write_backup() → L2 (Host)
│   └── write_backup_storage() → L3 (Mooncake)
│
├── 请求 B (间隔一段时间后):
│   ├── match_prefix() → L1 Miss, L2 Miss
│   ├── 检查 L3: batch_exists(["hash_你好", "hash_请问"]) → 命中!
│   ├── prefetch_from_storage() → 从 L3 加载到 L2
│   ├── load_back() → 从 L2 加载到 L1
│   └── 复用 80% 的 KV，只需计算新部分
│
└── 效率提升: 减少 80% 计算量

Workflow 细节:
1. 请求 A 结束时，KV 被备份到 Mooncake
   - key: "model_name_tp_0_hash_你好请问"
   - location: Global Segment (可能分布在多个节点)

2. 请求 B 启动时，检查 L3
   - HiRadixCache.prefetch_from_storage()
   - MooncakeStore.batch_exists(keys)
   - Master 返回: key 存在，位于 node-X, node-Y

3. 数据从 Mooncake 流回 SGLang
   - L3 (Mooncake Segment) → L2 (Host KV Cache)
   - L2 (Host) → L1 (GPU KV Cache)
   - 全程零拷贝
```

---

## 3. Mooncake-Store 的核心机制

### 3.1 零拷贝传输机制

```
传统拷贝方式 (4次拷贝):
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Storage  │ → │ Kernel   │ → │ User     │ → │ User     │ → │ GPU      │
│ (Disk)   │   │ Buffer   │   │ Buffer   │   │ Buffer   │   │ HBM      │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
                    ↑              ↑              ↑
               DMA拷贝         内核-用户      用户-驱动
                               空间拷贝       拷贝

Mooncake 零拷贝方式 (1-2次拷贝):
┌──────────┐                              ┌──────────┐   ┌──────────┐
│ Storage  │ ─────── RDMA/TCP ──────────→ │ Host     │ → │ GPU      │
│ (Remote  │        直接写入              │ KV Cache │   │ HBM      │
│  Segment)│                              │ (Pinned) │   │          │
└──────────┘                              └──────────┘   └──────────┘
                                                ↑              ↑
                                          预注册内存        DMA拷贝
                                          (mlock+ibv)     (GPU Direct)

关键点:
1. Host KV Cache 使用 pinned memory (cudaHostRegister)
2. RDMA 可以直接读写 pinned memory，无需内核介入
3. 数据从网络直接流入应用内存，无中间缓冲
```

### 3.2 分布式存储架构

```
单机部署:
┌─────────────────────────────────────────────────────┐
│  Node A                                             │
│  ┌──────────────┐    ┌──────────────────────────┐  │
│  │ SGLang       │    │ Mooncake Master          │  │
│  │              │    │ ┌──────────────────────┐ │  │
│  │ ┌──────────┐ │    │ │ Global Segment       │ │  │
│  │ │ GPU      │ │    │ │ (本地 SSD/内存)      │ │  │
│  │ │ L1 Cache │ │    │ └──────────────────────┘ │  │
│  │ └──────────┘ │    └──────────────────────────┘  │
│  │      ↓       │                                   │
│  │ ┌──────────┐ │    ┌──────────────────────────┐  │
│  │ │ Host     │ │    │ Metadata Service         │  │
│  │ │ L2 Cache │ │    │ (:18888)                 │  │
│  │ └──────────┘ │    └──────────────────────────┘  │
│  │      ↓       │                                   │
│  │ ┌──────────┐ │                                   │
│  │ │ Mooncake │ │                                   │
│  │ │ Client   │ │                                   │
│  │ └──────────┘ │                                   │
│  └──────────────┘                                   │
└─────────────────────────────────────────────────────┘

多机部署 (集群):
┌──────────────────┐         ┌──────────────────┐         ┌──────────────────┐
│   Node A         │         │   Node B         │         │   Node C         │
│   (SGLang)       │         │   (SGLang)       │         │   (Storage Only) │
│                  │         │                  │         │                  │
│ ┌──────────────┐ │         │ ┌──────────────┐ │         │ ┌──────────────┐ │
│ │ L1 GPU       │ │         │ │ L1 GPU       │ │         │ │              │ │
│ │ L2 Host      │ │←──RDMA─→│ │ L2 Host      │ │←──RDMA─→│ │ L3 Global    │ │
│ │ L3 Client    │ │         │ │ L3 Client    │ │         │ │   Segment    │ │
│ └──────────────┘ │         │ └──────────────┘ │         │ └──────────────┘ │
└────────┬─────────┘         └────────┬─────────┘         └────────┬─────────┘
         │                            │                            │
         └────────────────────────────┼────────────────────────────┘
                                      │
                           ┌──────────▼──────────┐
                           │  Mooncake Master    │
                           │  (集中式元数据)      │
                           │  - 管理所有 Segment │
                           │  - 协调跨节点传输   │
                           └─────────────────────┘

优势:
1. Node A 存储的 KV，Node B 可以直接复用
2. 热点数据可以分布在多个节点，提高吞吐量
3. 节点故障时，可以从其他副本恢复
```

### 3.3 数据放置策略

```
Master 的决策逻辑:

1. 写入时选择 Segment:
   - 本地优先: 如果本地有 Segment，优先写入本地
   - 负载均衡: 选择剩余空间最多的 Segment
   - 拓扑感知: 选择同机架/同交换机的节点

2. 读取时选择 Replica:
   - 最近读取: 选择网络延迟最低的副本
   - 负载均衡: 避免热点 Segment
   - 故障转移: 主副本不可用时，自动切换到备用副本

3. 副本策略:
   - 默认 1 副本 (性能优先)
   - 可配置 2-3 副本 (可靠性优先)
   - 跨机架副本 (容灾)
```

---

## 4. Mooncake-Store 的架构意义

### 4.1 解决的问题

| 问题 | 传统方案 | Mooncake 方案 |
|------|----------|---------------|
| **显存不足** | 买更多 GPU | 将冷数据 offload 到 Host/Storage |
| **重复计算** | 每个请求重新计算前缀 | 跨请求共享 KV Cache |
| **长尾延迟** | 大模型推理固定延迟 | Prefix 命中后延迟大幅降低 |
| **多机协作** | 每个节点独立缓存 | 集群级共享缓存 |
| **容错恢复** | 服务重启丢失缓存 | 持久化到 L3，重启后恢复 |

### 4.2 核心优势

#### 1. 跨请求 KV Cache 复用

```
场景: 客服对话系统

请求 1: "你好，请问怎么退款？"
请求 2: "你好，请问怎么退货？"
请求 3: "你好，请问订单在哪里看？"

传统方式:
- 每个请求都重新计算 "你好，请问" (4 tokens)
- 对于 70B 模型，这可能是 20-30ms 的重复计算

Mooncake L3:
- 请求 1 将 "你好，请问" 存储到 L3
- 请求 2/3 直接从 L3 加载，跳过前缀计算
- 延迟从 100ms → 30ms (3x 加速)
```

#### 2. 跨节点数据共享

```
场景: 4 节点推理集群

节点 1 处理: "讲一个关于猫的故事..."
节点 2 处理: "讲一个关于狗的故事..."

传统方式:
- 两个节点各自独立计算 "讲一个关于" 的 KV
- 浪费 2x 计算资源

Mooncake 集群:
- 节点 1 计算后将前缀存储到 L3
- 节点 2 直接从 L3 加载，无需计算
- 节省 50% 计算资源
```

#### 3. 大模型推理成本优化

```
成本构成 (以 70B 模型为例):
- GPU 计算: $2-3/小时
- 显存容量: 瓶颈 (40GB/80GB HBM)

优化效果:
- 缓存命中率 80% → 减少 80% 计算量
- 显存需求降低 50% (冷数据 offload 到 Host)
- 整体成本降低 60-70%
```

### 4.3 与 HiCache 的协同

```
Mooncake-Store 不是替代 HiCache，而是增强:

HiCache (L1+L2):
├── 管理 GPU/Host 内存池
├── Radix Tree 索引
├── 细粒度的缓存策略 (LRU, Write-Through)
└── 本地高性能访问

Mooncake (L3):
├── 分布式持久化存储
├── 跨节点共享
├── 大容量 (TB 级)
└── 相对较慢但可扩展

协同工作:
1. Hot data (最近使用): L1 (GPU)
2. Warm data (偶尔使用): L2 (Host)
3. Cold data (历史数据): L3 (Mooncake)

数据流动:
   ┌──────────────────────────────────────────┐
   │ Compute  │  Access  │  Write  │  Read    │
   ├──────────────────────────────────────────┤
   │ L1       │  Hot     │  Auto   │  Instant │
   │ L2       │  Warm    │  Async  │  Fast    │
   │ L3       │  Cold    │  Async  │  Slow    │
   └──────────────────────────────────────────┘
```

---

## 5. 实际部署架构

### 5.1 生产环境配置

```yaml
# 典型生产部署

sglang_servers:
  - node: sglang-0
    gpus: 8x A100
    hicache:
      l1_size: "80GB"        # GPU HBM
      l2_size: "512GB"       # Host DDR (pinned)
      l3_backend: "mooncake"
  - node: sglang-1
    gpus: 8x A100
    hicache:
      l1_size: "80GB"
      l2_size: "512GB"
      l3_backend: "mooncake"

mooncake_cluster:
  master:
    node: mooncake-master-0
    rpc_port: 55051
    metadata_port: 18888
    global_segment_size: "4TB"
  
  storage_nodes:
    - node: storage-0
      capacity: "2TB"
      network: "100Gbps RDMA"
    - node: storage-1
      capacity: "2TB"
      network: "100Gbps RDMA"
```

### 5.2 性能指标

| 指标 | L1 (GPU) | L2 (Host) | L3 (Mooncake) |
|------|----------|-----------|---------------|
| 访问延迟 | 10-100 μs | 1-10 ms | 10-100 ms |
| 带宽 | 1-2 TB/s | 10-50 GB/s | 1-10 GB/s |
| 容量 | 40-80 GB | 256-512 GB | 1-10 TB |
| 成本/GB | $20-30 | $3-5 | $0.1-0.5 |

---

## 6. 总结

### Mooncake-Store 的核心价值

1. **分层存储的 L3 层**: 提供大容量、分布式、持久化的 KV Cache 存储

2. **零拷贝高性能**: 通过 RDMA/pinned memory 实现无拷贝数据传输

3. **跨节点共享**: 打破单机限制，实现集群级 KV Cache 复用

4. **成本优化**: 通过缓存复用减少 60-70% 的计算成本

5. **容错与恢复**: 服务重启后可以从 L3 恢复缓存，无需重新预热

### Workflow 的本质

```
不是简单的 "读取/写入"，而是:

1. 计算 → 存储 → 复用 的闭环
   - 计算出的 KV 不只是临时使用
   - 而是存储到分布式存储，供未来复用

2. 内存层级之间的智能流动
   - Hot (L1) → Warm (L2) → Cold (L3)
   - 根据访问模式自动调整数据位置

3. 分布式协作
   - 多个 SGLang 实例共享同一个 Mooncake 集群
   - 一个实例存储的数据，其他实例可以复用

最终目标:
让大模型推理像 CPU Cache 一样高效，
通过分层存储最大化缓存命中率，
通过分布式共享最大化资源利用率。
