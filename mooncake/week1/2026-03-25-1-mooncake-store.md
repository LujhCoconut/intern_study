# Mooncake-Store 完全入门指南

> 本文档创建时间：2026-03-25  
> 适用版本：Mooncake 最新主干版本  
> 目标读者：初次接触 Mooncake-Store 的开发者

---

## 目录

1. [什么是 Mooncake-Store？](#一什么是-mooncake-store)
2. [快速开始：最简单的使用方式](#二快速开始最简单的使用方式)
3. [核心概念与架构](#三核心概念与架构)
4. [代码流程详解](#四代码流程详解)
5. [与 SGLang HiCache 集成](#五与-sglang-hicache-集成)
6. [高级特性](#六高级特性)
7. [故障排查](#七故障排查)

---

## 一、什么是 Mooncake-Store？

### 1.1 一句话解释

**Mooncake-Store** 是一个高性能的分布式键值存储系统，专门为大规模 AI 推理场景（如 vLLM、SGLang）设计，支持 RDMA/TCP 传输、多副本冗余、SSD 卸载等高级特性。

### 1.2 核心特性

| 特性 | 说明 |
|------|------|
| **高性能传输** | 支持 RDMA、NVLink、TCP 等多种传输协议 |
| **分层存储** | 内存 + 本地磁盘 + 分布式存储多级架构 |
| **多副本机制** | 支持配置多副本数，提高数据可靠性 |
| **Tensor 原生支持** | 内置 PyTorch Tensor 序列化/反序列化 |
| **零拷贝传输** | 支持预注册缓冲区，避免数据拷贝 |
| **高可用** | 支持 Master 主备切换、数据持久化 |

### 1.3 典型应用场景

```
┌─────────────────────────────────────────────────────────────────┐
│                      AI 推理集群                                  │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐              │
│  │ Prefill  │──────│  Decode  │      │  Decode  │              │
│  │  Node    │  KV  │  Node 1  │      │  Node 2  │              │
│  │          │ Cache│          │      │          │              │
│  └────┬─────┘      └────┬─────┘      └────┬─────┘              │
│       │                 │                 │                     │
│       └─────────────────┴─────────────────┘                     │
│                         │                                        │
│                    ┌────┴────┐                                   │
│                    │Mooncake │  ←  KV Cache 存储层                │
│                    │ Store   │                                   │
│                    └────┬────┘                                   │
│                         │                                        │
│              ┌──────────┼──────────┐                            │
│              ▼          ▼          ▼                            │
│         ┌──────┐   ┌──────┐   ┌──────┐                         │
│         │Memory│   │ Local│   │  SSD │                         │
│         │  RAM │   │ Disk │   │      │                         │
│         └──────┘   └──────┘   └──────┘                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、快速开始：最简单的使用方式

### 2.1 安装 Mooncake

```bash
# 从源码构建
git clone https://github.com/kvcache-ai/Mooncake.git
cd Mooncake
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
make install

# 安装 Python 包
cd ../mooncake-wheel
pip install -e .
```

### 2.2 启动基础服务

Mooncake-Store 需要两个基础服务：

```bash
# 1. 启动 Metadata 服务（用于服务发现）
python -m mooncake.http_metadata_server --port 8080

# 2. 启动 Master 服务（用于元数据管理）
mooncake_master --port 50051
```

### 2.3 Python 客户端基础用法

```python
import torch
from mooncake import store

# 1. 创建客户端实例
client = store.MooncakeDistributedStore()

# 2. 初始化（连接到 Master）
client.setup(
    local_hostname="192.168.1.100",           # 本机 IP
    metadata_server="http://192.168.1.1:8080/metadata",  # Metadata 服务地址
    global_segment_size=4*1024*1024*1024,     # 4GB 全局内存段
    local_buffer_size=1*1024*1024*1024,       # 1GB 本地缓冲区
    protocol="tcp",                           # 传输协议: tcp/rdma/nvlink
    rdma_devices="",                          # RDMA 设备名（使用 RDMA 时）
    master_server_addr="192.168.1.1:50051"    # Master 服务地址
)

# 3. 初始化传输层
client.init_all(protocol="tcp", device_name="", mount_segment_size=16*1024*1024)

# 4. 存储 PyTorch Tensor
tensor = torch.randn(1024, 1024, dtype=torch.float32)
client.put_tensor("my_key", tensor)

# 5. 读取 Tensor
retrieved = client.get_tensor("my_key")
print(retrieved.shape)  # torch.Size([1024, 1024])

# 6. 关闭连接
client.close()
```

### 2.4 关键配置参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `local_hostname` | 本机 IP 地址，用于其他节点连接 | 必填 |
| `metadata_server` | HTTP Metadata 服务地址 | 必填 |
| `global_segment_size` | 向 Master 注册的内存总量 | 16MB |
| `local_buffer_size` | 本地缓冲区大小 | 16MB |
| `protocol` | 传输协议: tcp/rdma/nvlink/ascend | tcp |
| `master_server_addr` | Master RPC 服务地址 | 127.0.0.1:50051 |

---

## 三、核心概念与架构

### 3.1 系统架构总览

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Mooncake-Store 架构                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐  │
│  │   Python 应用    │    │   Python 应用    │    │   Python 应用    │  │
│  │  (vLLM/SGLang)  │    │  (vLLM/SGLang)  │    │  (vLLM/SGLang)  │  │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘  │
│           │                      │                      │           │
│           ▼                      ▼                      ▼           │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐  │
│  │  Pybind11 封装   │    │  Pybind11 封装   │    │  Pybind11 封装   │  │
│  │  (store_py.cpp) │    │  (store_py.cpp) │    │  (store_py.cpp) │  │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘  │
│           │                      │                      │           │
│           ▼                      ▼                      ▼           │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐  │
│  │   RealClient    │    │   RealClient    │    │   DummyClient   │  │
│  │  (真实客户端)   │◄───►│  (真实客户端)   │    │  (轻量客户端)   │  │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘  │
│           │                      │                      │           │
│           │    ┌─────────────────┘                      │           │
│           │    │                                        │           │
│           ▼    ▼                                        ▼           │
│  ┌─────────────────┐                           ┌─────────────────┐  │
│  │  TransferEngine │                           │   IPC/RPC 调用   │  │
│  │  (RDMA/TCP传输) │                           │   (通过 Socket) │  │
│  └────────┬────────┘                           └────────┬────────┘  │
│           │                                              │           │
└───────────┼──────────────────────────────────────────────┼───────────┘
            │                                              │
            ▼                                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         Master Service 层                             │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │  │
│  │  │  MountSegment│  │   PutStart   │  │   GetReplica │        │  │
│  │  │   注册内存段  │  │   开始写入   │  │   查询副本   │        │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘        │  │
│  │                                                               │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │              Metadata Shards (1024 分片)                 │ │  │
│  │  │  ┌─────────┐ ┌─────────┐ ┌─────────┐     ┌─────────┐   │ │  │
│  │  │  │Shard 0  │ │Shard 1  │ │Shard 2  │ ... │Shard N  │   │ │  │
│  │  │  │(Key->   │ │(Key->   │ │(Key->   │     │(Key->   │   │ │  │
│  │  │  │Metadata)│ │Metadata)│ │Metadata)│     │Metadata)│   │ │  │
│  │  │  └─────────┘ └─────────┘ └─────────┘     └─────────┘   │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │  │
│  │  │Task Manager  │  │  Eviction    │  │  Snapshot    │        │  │
│  │  │  (任务管理)   │  │   (驱逐策略)  │  │  (数据快照)   │        │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘        │  │
│  └───────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.2 核心组件详解

#### 3.2.1 RealClient vs DummyClient

| 特性 | RealClient | DummyClient |
|------|------------|-------------|
| 内存管理 | 独立管理 | 依赖 RealClient |
| 适用场景 | 单进程高性能 | 多进程共享 |
| 初始化速度 | 较慢（需注册内存） | 快速 |
| 数据传输 | 直接传输 | 通过 IPC |

**使用建议：**
- **单进程应用**（如简单的推理服务）：直接使用 RealClient
- **多进程应用**（如 vLLM 的 TP/PP 并行）：一个进程用 RealClient，其他用 DummyClient

#### 3.2.2 Master Service

Master 是中央元数据管理服务，主要职责：

1. **元数据管理**：记录每个 Key 的副本位置、状态
2. **内存段管理**：跟踪所有客户端的可用内存
3. **副本管理**：协调副本复制、迁移
4. **生命周期管理**：处理客户端心跳、故障检测

#### 3.2.3 Replica（副本）

```cpp
// 三种副本类型
enum class ReplicaType {
    MEMORY,      // 内存副本（其他节点内存）
    DISK,        // 分布式磁盘（如 S3、HDFS）
    LOCAL_DISK   // 本地磁盘（SSD 卸载）
};

// 副本状态机
enum class ReplicaStatus {
    INITIALIZED,  // 已分配空间
    PROCESSING,   // 写入中
    COMPLETE,     // 写入完成，可用
    REMOVED,      // 已删除
    FAILED        // 写入失败
};
```

---

## 四、代码流程详解

### 4.1 写入流程（Put）

```
Client                          Master                         Target
  │                               │                              │
  │  1. PutStart(key, size)       │                              │
  │ ─────────────────────────────>│                              │
  │                               │  2. 选择目标 Segment          │
  │                               │  (分配策略：随机/剩余空间优先)  │
  │                               │                              │
  │  3. 返回 ReplicaDescriptor    │                              │
  │ <─────────────────────────────│                              │
  │                               │                              │
  │  4. 通过 TransferEngine       │                              │
  │     直接传输数据               │                              │
  │ ─────────────────────────────────────────────────────────────>│
  │                               │                              │
  │  5. PutEnd(key)               │                              │
  │ ─────────────────────────────>│                              │
  │                               │  6. 标记副本为 COMPLETE       │
  │                               │                              │
```

**代码路径：**
```cpp
// 1. Python 入口
store_py.cpp: put_tensor() -> put_tensor_impl()

// 2. RealClient 处理  
real_client.cpp: put_internal()
  ├── master_client.cpp: PutStart()  // 请求 Master 分配
  └── TransferEngine 传输数据
  
// 3. Master 分配
master_service.cpp: PutStart()
  ├── 选择 AllocationStrategy
  ├── 在目标 Segment 分配 Buffer
  └── 返回 Replica::Descriptor

// 4. 完成写入
real_client.cpp: put_internal() 继续
  └── master_client.cpp: PutEnd()  // 通知 Master 完成
```

### 4.2 读取流程（Get）

```
Client                          Master
  │                               │
  │  1. GetReplicaList(key)       │
  │ ─────────────────────────────>│
  │                               │  2. 查询 Metadata
  │                               │
  │  3. 返回 Replica 列表         │
  │ <─────────────────────────────│
  │                               │
  │  4. 选择最优副本（本地>同节点>其他）│
  │                               │
  │  5. 通过 TransferEngine       │
  │     从目标节点读取数据          │
  │ <─────────────────────────────│
```

**代码路径：**
```cpp
// 1. Python 入口
store_py.cpp: get_tensor() / get_buffer()

// 2. RealClient 处理
real_client.cpp: get_buffer_internal()
  ├── master_client.cpp: GetReplicaList()  // 查询副本位置
  ├── 选择最佳副本（本地优先）
  └── TransferEngine 读取数据
```

### 4.3 关键类关系图

```
PyClient (抽象基类)
    ├── RealClient
    │     ├── MasterClient        // RPC 客户端
    │     ├── ClientRequester     // HTTP 客户端
    │     ├── FileStorage         // 本地存储管理
    │     └── TransferEngine      // 数据传输引擎
    │
    └── DummyClient
          └── 通过 IPC 调用 RealClient

MasterService
    ├── MetadataShard[1024]       // 分片元数据存储
    │     └── map<key, ObjectMetadata>
    ├── AllocationStrategy        // 分配策略
    │     ├── RandomStrategy
    │     ├── FreeRatioFirstStrategy
    │     └── CXLStrategy
    ├── EvictionStrategy          // 驱逐策略
    └── TaskManager               // 任务管理
```

---

## 五、与 SGLang HiCache 集成

### 5.1 SGLang HiCache 简介

**HiCache** 是 SGLang 推理框架的 KV Cache 管理组件，支持：
- 分层存储（GPU HBM -> CPU DRAM -> SSD）
- 动态缓存驱逐
- 多种存储后端（File、Mooncake 等）

### 5.2 集成架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SGLang 推理进程                              │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                     HiCache 管理器                             │ │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐     │ │
│  │  │   GPU 缓存    │  │   CPU 缓存    │  │   存储后端     │     │ │
│  │  │  (HBM)        │  │  (DRAM)       │  │               │     │ │
│  │  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘     │ │
│  │          │                  │                  │              │ │
│  │          └──────────────────┴──────────────────┘              │ │
│  │                             │                                 │ │
│  │                             ▼                                 │ │
│  │                    ┌─────────────────┐                        │ │
│  │                    │  Storage Backend│                        │ │
│  │                    │  Interface      │                        │ │
│  │                    └────────┬────────┘                        │ │
│  │                             │                                 │ │
│  └─────────────────────────────┼─────────────────────────────────┘ │
│                                │                                    │
│  ┌─────────────────────────────┼─────────────────────────────────┐ │
│  │         Mooncake Store      │     (mooncake-integration)       │ │
│  │  ┌──────────────────────────┴──────────────────────────────┐  │ │
│  │  │              MooncakeStorePyWrapper                      │  │ │
│  │  │  - put_tensor()    - get_tensor()    - batch_put/get    │  │ │
│  │  └──────────────────────────┬──────────────────────────────┘  │ │
│  │                             │                                  │ │
│  │  ┌──────────────────────────┴──────────────────────────────┐  │ │
│  │  │                   RealClient                              │  │ │
│  │  └──────────────────────────┬──────────────────────────────┘  │ │
│  │                             │                                  │ │
│  │  ┌──────────────────────────┴──────────────────────────────┐  │ │
│  │  │                  TransferEngine                           │  │ │
│  │  └──────────────────────────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.3 配置方法

#### 启动 Mooncake 服务

```bash
# 1. 启动 Metadata Server
python -m mooncake.http_metadata_server --port 8080

# 2. 启动 Master Server
mooncake_master --port 50051
```

#### 启动 SGLang（带 HiCache）

```bash
# 设置 Mooncake 环境变量
export MOONCAKE_MASTER="127.0.0.1:50051"
export MOONCAKE_PROTOCOL="tcp"
export MOONCAKE_TE_META_DATA_SERVER="http://127.0.0.1:8080/metadata"
export MOONCAKE_GLOBAL_SEGMENT_SIZE="4294967296"  # 4GB

# 启动 SGLang Server
python -m sglang.launch_server \
    --model meta-llama/Llama-2-7b-hf \
    --tp-size 2 \
    --hicache-ratio 2 \
    --hicache-storage-backend mooncake \
    --mem-fraction-static 0.8
```

### 5.4 关键参数说明

| SGLang 参数 | 说明 | 推荐值 |
|------------|------|--------|
| `--hicache-ratio` | CPU/GPU 显存比例 | 2-4（根据内存大小） |
| `--hicache-storage-backend` | 存储后端类型 | `mooncake` |
| `--hicache-mem-layout` | 内存布局方式 | `page_first` 或 `layer_first` |
| `--mem-fraction-static` | 静态内存分配比例 | 0.6-0.8 |

| 环境变量 | 说明 | 示例 |
|----------|------|------|
| `MOONCAKE_MASTER` | Master 服务地址 | `127.0.0.1:50051` |
| `MOONCAKE_PROTOCOL` | 传输协议 | `tcp` / `rdma` |
| `MOONCAKE_TE_META_DATA_SERVER` | Metadata 服务 URL | `http://127.0.0.1:8080/metadata` |
| `MOONCAKE_GLOBAL_SEGMENT_SIZE` | 全局内存段大小 | `4294967296` (4GB) |

### 5.5 代码集成示例

```python
# SGLang HiCache 使用 Mooncake 后端的配置示例
# 文件：test_hicache_storage_mooncake_backend.py

class HiCacheStorageMooncakeBackendBaseMixin:
    @classmethod
    def setUpClass(cls):
        # 1. 启动 Mooncake 服务
        cls._start_mooncake_services()
        super().setUpClass()
    
    @classmethod
    def _start_mooncake_services(cls):
        # 启动 metadata 服务
        cls.metadata_service_process = subprocess.Popen([
            "python3", "-m", "mooncake.http_metadata_server",
            "--port", str(cls.mooncake_metadata_port)
        ])
        
        # 启动 master 服务
        cls.master_service_process = subprocess.Popen([
            "mooncake_master", "--port", str(cls.mooncake_master_port)
        ])
    
    @classmethod
    def _get_additional_server_args_and_env(cls):
        server_args = {
            "--tp-size": 2,
            "--hicache-ratio": 2,
            "--hicache-storage-backend": "mooncake",
            "--mem-fraction-static": 0.8,
        }
        
        env_vars = {
            "MOONCAKE_MASTER": f"127.0.0.1:{cls.mooncake_master_port}",
            "MOONCAKE_PROTOCOL": "tcp",
            "MOONCAKE_TE_META_DATA_SERVER": f"http://127.0.0.1:{cls.mooncake_metadata_port}/metadata",
            "MOONCAKE_GLOBAL_SEGMENT_SIZE": "4294967296",
        }
        
        return server_args, env_vars
```

### 5.6 性能优化建议

1. **使用 RDMA 协议**：如果硬件支持，将 `MOONCAKE_PROTOCOL` 设为 `rdma` 可大幅提升传输性能
2. **合理设置内存比例**：`--hicache-ratio` 建议根据实际 KV Cache 大小调整
3. **启用多副本**：关键数据可配置多副本提高可靠性
4. **NUMA 绑定**：使用 `bind_to_numa_node()` 减少跨 NUMA 访问

---

## 六、高级特性

### 6.1 零拷贝传输

```python
from mooncake import store

client = store.MooncakeDistributedStore()
client.setup(...)

# 1. 预注册缓冲区
buffer_ptr = client.register_buffer(buffer_size)

# 2. 使用预注册缓冲区写入
tensor = torch.randn(1024, 1024)
# 将 tensor 数据拷贝到预注册缓冲区
# ...
client.put_tensor_from("key", buffer_ptr, buffer_size)

# 3. 使用预注册缓冲区读取
client.get_tensor_into("key", buffer_ptr, buffer_size)

# 4. 注销缓冲区
client.unregister_buffer(buffer_ptr)
```

### 6.2 批量操作

```python
# 批量写入
tensors = [torch.randn(1024, 1024) for _ in range(10)]
keys = [f"key_{i}" for i in range(10)]
results = client.batch_put_tensor(keys, tensors)

# 批量读取
retrieved = client.batch_get_tensor(keys)

# 带 Tensor Parallelism 的批量操作
tp_size = 4
tp_rank = 0  # 当前 rank
results = client.batch_put_tensor_with_tp(
    keys, tensors, tp_rank=tp_rank, tp_size=tp_size, split_dim=0
)
```

### 6.3 多副本配置

```python
from mooncake.store import ReplicateConfig

# 创建配置：3 副本，软固定
config = ReplicateConfig()
config.replica_num = 3
config.with_soft_pin = True
config.preferred_segments = ["node1", "node2", "node3"]

# 使用配置写入
client.pub_tensor("important_key", tensor, config)
```

### 6.4 NVLink 支持（SGLang 定制）

```python
# allocator.py - 用于检测和使用 NVLink
from mooncake.allocator import NVLinkAllocator

# 检测是否支持 Fabric Memory
backend = NVLinkAllocator.detect_mem_backend()
if backend == MemoryBackend.USE_CUMEMCREATE:
    print("支持 Fabric Memory，可使用 NVLink 直接传输")
    allocator = NVLinkAllocator.get_allocator(torch_device)
else:
    print("使用 CudaMalloc 回退方案")
```

---

## 七、故障排查

### 7.1 常见问题

#### Q1: 连接 Master 失败
```
Error: RPC_FAIL (-900)
```
**解决方案：**
- 检查 Master 服务是否启动：`ps aux | grep mooncake_master`
- 检查防火墙设置：`iptables -L | grep 50051`
- 检查地址配置是否正确

#### Q2: 内存分配失败
```
Error: NO_AVAILABLE_HANDLE (-200)
```
**解决方案：**
- 增大 `global_segment_size`
- 检查是否有足够的系统内存
- 考虑启用 SSD 卸载

#### Q3: 传输超时
```
Error: TRANSFER_FAIL (-800)
```
**解决方案：**
- 检查网络连通性：`ping <remote_host>`
- 尝试切换协议：`protocol="tcp"`
- 检查 RDMA 配置（如使用 RDMA）

### 7.2 调试技巧

```python
# 启用详细日志
import logging
logging.basicConfig(level=logging.DEBUG)

# 健康检查
status = client.health_check()
if status == 0:
    print("连接正常")
elif status == 1:
    print("未初始化")
elif status == 2:
    print("Master 不可达")

# 获取副本信息
replica_desc = client.get_replica_desc("key")
for desc in replica_desc:
    print(f"Status: {desc.status}")
    if desc.is_memory_replica():
        mem = desc.get_memory_descriptor()
        print(f"Buffer address: {mem.buffer_address}")
```

### 7.3 性能监控

```python
# Master 指标（在 Master 节点）
# 可通过 HTTP 接口获取：http://master:8080/metrics

# Client 指标
# 查看 Replica 分布
all_segments = client.get_all_segments()
for seg in all_segments:
    capacity, used = client.query_segments(seg)
    print(f"Segment {seg}: {used}/{capacity} bytes")
```

---

## 附录：代码文件索引

| 文件路径 | 说明 |
|----------|------|
| `mooncake-store/include/real_client.h` | RealClient 头文件 |
| `mooncake-store/include/dummy_client.h` | DummyClient 头文件 |
| `mooncake-store/include/master_service.h` | Master 服务接口 |
| `mooncake-store/include/replica.h` | 副本定义 |
| `mooncake-store/include/types.h` | 基础类型定义 |
| `mooncake-integration/store/store_py.cpp` | Python 绑定实现 |
| `mooncake-wheel/mooncake/mooncake_connector_v1.py` | vLLM V1 连接器 |
| `mooncake-integration/allocator.py` | NVLink 分配器 |

---

> **文档结束**  
> 如有问题，请参考 [Mooncake GitHub](https://github.com/kvcache-ai/Mooncake) 或提交 Issue。
