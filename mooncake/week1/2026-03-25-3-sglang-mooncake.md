# SGLang + Mooncake HiCache 集成详解

**日期**: 2026-03-25  
**作者**: AI Assistant  
**文件**: `plan-1.sh`

---

## 目录

1. [执行摘要](#1-执行摘要)
2. [plan-1.sh 做了什么](#2-plan-1sh-做了什么)
3. [核心原理详解](#3-核心原理详解)
4. [关键代码分析](#4-关键代码分析)
5. [架构流程图](#5-架构流程图)
6. [环境变量与配置](#6-环境变量与配置)
7. [端口与服务说明](#7-端口与服务说明)
8. [故障排查](#8-故障排查)

---

## 1. 执行摘要

`plan-1.sh` 是一个自动化测试脚本，实现了 **SGLang + Mooncake HiCache** 的集成部署。它成功将 SGLang 的 KV Cache 分层存储功能与 Mooncake 的分布式存储后端对接，实现了：

- **L1 Cache**: GPU 显存中的 KV Cache
- **L2 Cache**: CPU 内存中的 KV Cache (Host KV Cache)
- **L3 Cache**: Mooncake 分布式存储后端

---

## 2. plan-1.sh 做了什么

### 2.1 主要步骤

```bash
# 1. 环境检查
- 检查 Python 3.10+ 环境
- 检查 GPU 可用性 (NVIDIA A100)
- 检查端口占用情况 (55051, 18888, 30001)

# 2. Python 环境准备
- 创建 venv 虚拟环境
- 安装 SGLang (0.5.9)
- 安装 Mooncake Python 包 (mooncake-wheel)
- 安装 sglang-router

# 3. 启动 Mooncake Master 服务
mooncake_master \
    --rpc_port 55051 \
    --http_metadata_server_port 18888 \
    --enable_http_metadata_server

# 4. 启动 SGLang 服务 (带 HiCache)
MOONCAKE_MASTER=127.0.0.1:55051 \
MOONCAKE_PROTOCOL=tcp \
MOONCAKE_TE_META_DATA_SERVER=http://127.0.0.1:18888/metadata \
python -m sglang.launch_server \
    --model-path /home/ljh/model/Qwen2.5-0.5B-Instruct \
    --enable-hierarchical-cache \
    --hicache-storage-backend mooncake \
    --hicache-ratio 2 \
    --port 30001

# 5. 执行测试
- 发送 generate 请求
- 验证文本生成功能
- 显示统计信息
```

### 2.2 核心配置参数

| 参数 | 值 | 说明 |
|------|-----|------|
| MOONCAKE_MASTER_PORT | 55051 | Master gRPC 端口 |
| MOONCAKE_METADATA_PORT | 18888 | Metadata HTTP 端口 |
| SGLANG_PORT | 30001 | SGLang API 端口 |
| hicache-ratio | 2 | Host Cache 是 Device Cache 的 2 倍 |
| hicache-storage-backend | mooncake | L3 存储后端类型 |

---

## 3. 核心原理详解

### 3.1 HiCache 三层缓存架构

```
┌─────────────────────────────────────────────────────────────┐
│                        请求处理流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 请求接入                                                │
│     └── 新请求: "Tell me a story about AI"                  │
│                                                             │
│  2. Radix Cache 前缀匹配                                     │
│     └── 检查是否有缓存的 KV 可以复用                         │
│                                                             │
│  3. 三级缓存查找                                             │
│     ┌───────────────────────────────────────────────┐       │
│     │ L1: GPU Device Cache                          │       │
│     │    └── 命中? 直接返回 ✓                       │       │
│     │    └── 未命中 → 查询 L2                       │       │
│     ├───────────────────────────────────────────────┤       │
│     │ L2: Host Memory Cache                         │       │
│     │    └── 命中? 拷贝到 GPU ✓                     │       │
│     │    └── 未命中 → 查询 L3 (Mooncake)            │       │
│     ├───────────────────────────────────────────────┤       │
│     │ L3: Mooncake Distributed Storage              │       │
│     │    └── 命中? 加载到 Host → GPU ✓              │       │
│     │    └── 未命中 → 计算新的 KV                   │       │
│     └───────────────────────────────────────────────┘       │
│                                                             │
│  4. 写入策略 (Write-Through)                                │
│     └── 新计算的 KV 同时写入 L1 + L2 + L3                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 为什么 SGLang 能"接上" Mooncake?

#### 3.2.1 插件化存储后端架构

SGLang 通过 **Storage Backend Factory** 模式实现了存储后端的插件化：

```python
# sglang/srt/mem_cache/storage/backend_factory.py

class StorageBackendFactory:
    _registry: Dict[str, Dict[str, Any]] = {}

    @classmethod
    def register_backend(cls, name: str, module_path: str, class_name: str):
        """注册存储后端"""
        cls._registry[name] = {
            "loader": loader,
            "module_path": module_path,
            "class_name": class_name,
        }

# 注册 Mooncake 后端
StorageBackendFactory.register_backend(
    "mooncake",
    "sglang.srt.mem_cache.storage.mooncake_store.mooncake_store",
    "MooncakeStore",
)
```

**关键点**: SGLang 不需要硬编码 Mooncake 的实现，只需通过配置 `--hicache-storage-backend mooncake` 即可动态加载。

#### 3.2.2 统一接口: HiCacheStorage

所有存储后端必须继承 `HiCacheStorage` 基类：

```python
# sglang/srt/mem_cache/hicache_storage.py

class HiCacheStorage(ABC):
    @abstractmethod
    def batch_get(self, keys, target_locations, target_sizes) -> int:
        """从存储批量读取 KV 到 Host Memory"""
        pass

    @abstractmethod
    def batch_set(self, keys, target_locations, target_sizes) -> bool:
        """将 KV 从 Host Memory 批量写入存储"""
        pass

    @abstractmethod
    def batch_exists(self, keys) -> int:
        """检查 keys 是否存在"""
        pass
```

**MooncakeStore 实现**:

```python
# sglang/srt/mem_cache/storage/mooncake_store/mooncake_store.py

class MooncakeStore(HiCacheStorage):
    def __init__(self, storage_config, mem_pool):
        # 1. 导入 Mooncake Python 绑定
        from mooncake.store import MooncakeDistributedStore
        self.store = MooncakeDistributedStore()

        # 2. 从环境变量加载配置
        self.config = MooncakeStoreConfig.load_from_env()

        # 3. 初始化 Mooncake Store
        self.store.setup(
            local_hostname=self.config.local_hostname,
            metadata_server=self.config.metadata_server,
            global_segment_size=self.config.global_segment_size,
            protocol=self.config.protocol,  # "tcp" 或 "rdma"
            device_name=self.config.device_name,
            master_server_address=self.config.master_server_address,
        )
```

#### 3.2.3 零拷贝 (Zero-Copy) 传输

Mooncake 通过 **Zero-Copy** 技术直接从 Host KV Cache 内存缓冲区读写数据，无需额外的数据拷贝：

```python
def register_mem_pool_host(self, mem_pool_host: HostKVCache):
    """注册 Host KV Cache 缓冲区到 Mooncake"""
    buffer = self.mem_pool_host.kv_buffer
    buffer_ptr = buffer.data_ptr()  # 获取 PyTorch Tensor 内存地址
    buffer_size = buffer.numel() * buffer.element_size()

    # 将缓冲区注册到 Mooncake，实现零拷贝访问
    self.store.register_buffer(buffer_ptr, buffer_size)

def batch_get(self, keys, target_locations, target_sizes):
    """零拷贝读取: 数据直接从 Mooncake 写入 Host Memory"""
    return self.store.batch_get_into(keys, target_locations, target_sizes)

def batch_set(self, keys, target_locations, target_sizes):
    """零拷贝写入: 数据直接从 Host Memory 读取到 Mooncake"""
    return self.store.batch_put_from(keys, target_locations, target_sizes)
```

### 3.3 Mooncake 服务组件

```
┌─────────────────────────────────────────────────────────────┐
│                    Mooncake 架构                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐         ┌──────────────────────────────┐  │
│  │   Master     │◄────────│   Metadata Service (HTTP)    │  │
│  │   Service    │  gRPC   │   - 管理 segment 元数据       │  │
│  │   :55051     │         │   - 提供节点发现服务          │  │
│  └──────┬───────┘         └──────────────────────────────┘  │
│         │                                                   │
│         │  管理全局存储资源                                   │
│         │                                                   │
│  ┌──────▼───────┐         ┌──────────────────────────────┐  │
│  │   Global     │         │      Mooncake Store          │  │
│  │   Segment    │◄────────│      (SGLang 内部)            │  │
│  │   (共享内存)  │  RDMA/TCP│      - put/get/exists        │  │
│  └──────────────┘         │      - batch 操作            │  │
│                           └──────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**服务说明**:

1. **Master Service** (端口 55051): 全局协调者，管理 segment 分配
2. **Metadata Service** (端口 18888): HTTP 服务，提供节点发现和元数据查询
3. **Mooncake Store**: SGLang 内部的客户端库，通过 TCP/RDMA 与 Master 通信

---

## 4. 关键代码分析

### 4.1 SGLang 启动流程中的 HiCache 初始化

```python
# sglang/srt/model_executor/model_runner.py

class ModelRunner:
    def __init__(self, server_args):
        # ... 其他初始化 ...

        # 1. 根据 server_args 决定是否启用 HiCache
        if server_args.enable_hierarchical_cache:
            # 创建 HiRadixCache (支持 L2/L3 的分层缓存)
            self.cache = HiRadixCache(cache_params, server_args)
        else:
            # 普通 RadixCache (仅 L1 GPU 缓存)
            self.cache = RadixCache(cache_params)
```

### 4.2 HiRadixCache 初始化

```python
# sglang/srt/mem_cache/hiradix_cache.py

class HiRadixCache(RadixCache):
    def __init__(self, params, server_args):
        # 1. 创建 Host KV Cache (L2)
        self.token_to_kv_pool_host = MHATokenToKVPoolHost(
            self.kv_cache,
            server_args.hicache_ratio,        # 2 (Host 是 Device 的 2 倍)
            server_args.hicache_size,         # 0 (自动计算)
            self.page_size,
            server_args.hicache_mem_layout,   # "page_first"
            allocator_type=server_args.hicache_storage_backend,  # "mooncake"
        )
```

### 4.3 Host KV Cache 与存储后端绑定

```python
# sglang/srt/mem_cache/memory_pool_host.py

class MHATokenToKVPoolHost:
    def __init__(self, ..., allocator_type="mooncake"):
        # 1. 分配 Host 内存
        self.allocator = self._create_allocator(allocator_type)
        self.kv_buffer = self.allocator.allocate(...)

        # 2. 创建存储后端 (Mooncake)
        self.storage = self._create_storage_backend(allocator_type)

    def _create_storage_backend(self, backend_name):
        # 通过工厂创建 MooncakeStore
        return StorageBackendFactory.create_backend(
            backend_name="mooncake",
            storage_config=storage_config,
            mem_pool_host=self,
        )
```

### 4.4 缓存查找流程

```python
# sglang/srt/mem_cache/hiradix_cache.py

class HiRadixCache:
    def match_prefix(self, rid, input_ids):
        """查找缓存的 KV"""
        # 1. 先在 L1 (GPU) 中查找
        gpu_match = self._match_in_device_cache(rid, input_ids)
        if gpu_match.hit:
            return gpu_match

        # 2. 在 L2 (Host) 中查找
        host_match = self._match_in_host_cache(rid, input_ids)
        if host_match.hit:
            return host_match

        # 3. 在 L3 (Mooncake) 中查找
        mooncake_match = self._match_in_storage(rid, input_ids)
        if mooncake_match.hit:
            # 从 Mooncake 加载到 Host，再拷贝到 GPU
            self._load_from_storage_to_host(mooncake_match)
            return mooncake_match

        return None
```

### 4.5 Mooncake 配置加载

```python
# sglang/srt/mem_cache/storage/mooncake_store/mooncake_store.py

class MooncakeStoreConfig:
    @staticmethod
    def load_from_env() -> "MooncakeStoreConfig":
        """从环境变量加载配置"""
        return MooncakeStoreConfig(
            local_hostname=os.getenv("MOONCAKE_LOCAL_HOSTNAME", "localhost"),
            metadata_server=os.getenv("MOONCAKE_TE_META_DATA_SERVER", "P2PHANDSHAKE"),
            global_segment_size=os.getenv("MOONCAKE_GLOBAL_SEGMENT_SIZE", "4gb"),
            protocol=os.getenv("MOONCAKE_PROTOCOL", "tcp"),
            device_name=os.getenv("MOONCAKE_DEVICE", ""),
            master_server_address=os.getenv("MOONCAKE_MASTER"),
            master_metrics_port=int(os.getenv("MOONCAKE_MASTER_METRICS_PORT", "9003")),
        )
```

---

## 5. 架构流程图

### 5.1 完整数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              客户端请求                                       │
│                    "Tell me a story about AI"                                 │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SGLang HTTP Server                                  │
│                              (Port 30001)                                     │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Scheduler/Tokenizer                                │
│                      - Tokenize input                                         │
│                      - Build token_ids                                        │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          HiRadixCache.match_prefix()                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ Input: token_ids = [1, 15043, 29892, 590, 11203, 338]                   │ │
│  │ ("Tell me a story about AI")                                            │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                  │                                          │
│          ┌───────────────────────┼───────────────────────┐                  │
│          │                       │                       │                  │
│          ▼                       ▼                       ▼                  │
│  ┌───────────────┐      ┌───────────────┐      ┌───────────────────┐       │
│  │  L1 Cache     │      │  L2 Cache     │      │  L3 Cache         │       │
│  │  (GPU)        │      │  (Host RAM)   │      │  (Mooncake)       │       │
│  │               │      │               │      │                   │       │
│  │ token_tree    │      │ token_tree    │      │ token_tree        │       │
│  │ radix_match   │      │ radix_match   │      │ key -> segment_id │       │
│  │               │      │               │      │                   │       │
│  │ 部分命中?     │      │ 部分命中?     │      │ batch_exists()    │       │
│  │ [1,15043]     │      │ [1,15043]     │      │ [1,15043]         │       │
│  └───────┬───────┘      └───────┬───────┘      └─────────┬─────────┘       │
│          │                      │                       │                  │
│          ▼                      ▼                       ▼                  │
│     [未命中]              [未命中]                [命中 90%]                │
│          │                      │                       │                  │
│          └──────────────────────┴───────────────────────┘                  │
│                                  │                                          │
│                                  ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ Action: 从 Mooncake L3 加载 KV 到 Host L2，再拷贝到 GPU L1             │ │
│  │                                                                        │ │
│  │ 1. storage.batch_get(keys, host_buffer_ptr, sizes)                     │ │
│  │    → Mooncake 通过 RDMA/TCP 将数据写入 Host Memory                     │ │
│  │                                                                        │ │
│  │ 2. host_to_device_copy(host_buffer, device_buffer)                     │ │
│  │    → 异步 DMA 将数据从 Host 拷贝到 GPU                                 │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Model Inference                                       │
│                   - 复用缓存的 KV (前缀)                                       │
│                   - 计算新的 KV (后缀)                                         │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Cache Update (Write-Through)                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ 新生成的 KV 同时写入:                                                   │ │
│  │   L1: device_cache.extend(new_kv)                                       │ │
│  │   L2: host_cache.extend(new_kv)                                         │ │
│  │   L3: storage.batch_set(keys, host_ptr, sizes)                          │ │
│  │       → Mooncake 将数据持久化到 Global Segment                          │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 环境变量与配置

### 6.1 Mooncake 环境变量

| 环境变量 | 必需 | 默认值 | 说明 |
|----------|------|--------|------|
| `MOONCAKE_MASTER` | 是 | - | Master 服务地址，如 `127.0.0.1:55051` |
| `MOONCAKE_PROTOCOL` | 否 | `tcp` | 传输协议: `tcp` 或 `rdma` |
| `MOONCAKE_TE_META_DATA_SERVER` | 否 | `P2PHANDSHAKE` | Metadata 服务 URL |
| `MOONCAKE_GLOBAL_SEGMENT_SIZE` | 否 | `4gb` | 每个 TP rank 的存储配额 |
| `MOONCAKE_LOCAL_HOSTNAME` | 否 | `localhost` | 本地主机名 |
| `MOONCAKE_DEVICE` | 否 | `""` | RDMA 设备名 (如 `mlx5_0`) |

### 6.2 SGLang HiCache 参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--enable-hierarchical-cache` | flag | false | 启用分层缓存 |
| `--hicache-ratio` | float | 2.0 | Host/Device 缓存大小比 |
| `--hicache-storage-backend` | str | `file` | L3 后端: `mooncake`, `file`, `nixl` |
| `--hicache-storage-prefetch-policy` | str | `timeout` | 预取策略 |
| `--hicache-write-policy` | str | `write_through` | 写入策略 |
| `--hicache-mem-layout` | str | `page_first` | 内存布局 |

---

## 7. 端口与服务说明

```
┌─────────────────────────────────────────────────────────────┐
│                      服务端口映射                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Mooncake Master                                    │   │
│  │  ├─ gRPC Port: 55051 (原 50051)                     │   │
│  │  └─ HTTP Metadata: 18888 (原 8888)                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          │ gRPC 协议                        │
│                          │ TCP/RDMA 传输                    │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  SGLang Server                                      │   │
│  │  ├─ HTTP API: 30001                                 │   │
│  │  ├─ HiCache L2 (Host Memory)                        │   │
│  │  └─ HiCache L3 (Mooncake Store)                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          │ HTTP REST API                    │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Client (curl/python)                               │   │
│  │  POST /generate                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7.1 为什么修改默认端口?

原默认端口与系统其他服务冲突：

| 服务 | 默认端口 | 修改后 | 原因 |
|------|----------|--------|------|
| Mooncake Master | 50051 | 55051 | 50051 常被其他 gRPC 服务使用 |
| Mooncake Metadata | 8888 | 18888 | 8888 被其他用户的 mooncake_master 占用 |
| SGLang | 30000 | 30001 | 避免与其他 SGLang 实例冲突 |

---

## 8. 故障排查

### 8.1 常见问题

#### Q1: SGLang 启动时提示 `ImportError: mooncake`

**原因**: Mooncake Python 模块未正确安装

**解决**:
```bash
# 检查模块是否存在
ls /home/ljh/Mooncake/mooncake-wheel/mooncake/

# 创建符号链接
ln -s /home/ljh/Mooncake/build/mooncake-integration/store*.so \
  /home/ljh/Mooncake/mooncake-wheel/mooncake/store.so
ln -s /home/ljh/Mooncake/build/mooncake-integration/engine*.so \
  /home/ljh/Mooncake/mooncake-wheel/mooncake/engine.so
```

#### Q2: Mooncake Master 启动失败

**原因**: 端口被占用或二进制文件不存在

**解决**:
```bash
# 检查端口占用
lsof -i :55051

# 使用 build 目录的 mooncake_master
export PATH="/home/ljh/Mooncake/build/mooncake-store/src:$PATH"
```

#### Q3: Metadata 服务返回 404

**原因**: 参数格式错误

**解决**:
```bash
# 正确用法 (不带 =true)
mooncake_master --enable_http_metadata_server --rpc_port 55051

# 错误用法
mooncake_master --enable_http_metadata_server=true  # 不要这样写
```

### 8.2 日志位置

```bash
# SGLang 日志
tail -f /home/ljh/learn_mooncake/test/logs/sglang_*.log

# Mooncake 日志
tail -f /home/ljh/learn_mooncake/test/logs/mooncake_master_*.log

# 测试响应
cat /home/ljh/learn_mooncake/test/logs/test_response_*.json
```

---

## 总结

`plan-1.sh` 成功实现了 SGLang + Mooncake 的集成，核心原理：

1. **分层缓存**: L1 (GPU) → L2 (Host) → L3 (Mooncake)
2. **插件化后端**: SGLang 通过 `StorageBackendFactory` 动态加载 Mooncake
3. **零拷贝传输**: Mooncake 直接操作 Host KV Cache 内存，避免数据拷贝
4. **统一接口**: `HiCacheStorage` 抽象层使得后端切换透明

通过环境变量配置，SGLang 在启动时自动连接到 Mooncake Master 服务，实现了 KV Cache 的分布式存储和跨请求复用。
