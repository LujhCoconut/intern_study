# SGLang + Mooncake HiCache 深度集成分析

**日期**: 2026-03-25  
**文档版本**: v2 - 深度技术分析  
**目标**: 理解每一层代码如何实现 SGLang 与 Mooncake 的交互

---

## 目录

1. [架构总览](#1-架构总览)
2. [工厂模式：动态加载 Mooncake 后端](#2-工厂模式动态加载-mooncake-后端)
3. [Mooncake Python Binding 详解](#3-mooncake-python-binding-详解)
4. [零拷贝内存机制](#4-零拷贝内存机制)
5. [存储后端初始化全流程](#5-存储后端初始化全流程)
6. [KV Cache 数据流](#6-kv-cache-数据流)
7. [RPC 通信机制](#7-rpc-通信机制)
8. [关键代码走读](#8-关键代码走读)

---

## 1. 架构总览

### 1.1 完整架构图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Python Application Layer                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                         SGLang Server                                   │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐  │   │
│  │  │ HTTP Server  │  │  Scheduler   │  │   Model Runner (GPU)         │  │   │
│  │  │   :30001     │  │              │  │                              │  │   │
│  │  └──────┬───────┘  └──────┬───────┘  │  ┌────────────────────────┐  │  │   │
│  │         │                 │          │  │ L1: Device KV Cache    │  │  │   │
│  │         │                 │          │  │   (GPU Memory)         │  │  │   │
│  │         │                 │          │  └───────────┬────────────┘  │  │   │
│  │         │                 │          └──────────────┼───────────────┘  │   │
│  │         │                 │                         │                    │   │
│  │         │                 │          ┌──────────────▼───────────────┐   │   │
│  │         │                 │          │ HiRadixCache                 │   │   │
│  │         │                 │          │ - prefix matching            │   │   │
│  │         │                 │          │ - cache eviction             │   │   │
│  │         │                 │          └───────────┬──────────────────┘   │   │
│  │         │                 │                      │                      │   │
│  │         │                 └──────────────────────┘                      │   │
│  │         │                                                               │   │
│  │         └─────────────────────────────────────────────────────────────┘   │
│  │                                                                           │   │
│  │  ┌──────────────────────────────────────────────────────────────────┐    │   │
│  │  │                    Host Memory (CPU RAM)                         │    │   │
│  │  │  ┌──────────────────────────────────────────────────────────┐   │    │   │
│  │  │  │ L2: Host KV Cache                                        │   │    │   │
│  │  │  │  - MHATokenToKVPoolHost / MLATokenToKVPoolHost           │   │    │   │
│  │  │  │  - Raw buffer registered with Mooncake                   │   │    │   │
│  │  │  └──────────────────────────────────────────────────────────┘   │    │   │
│  │  └──────────────────────────────────────────────────────────────────┘    │   │
│  │                                                                           │   │
│  │  ┌──────────────────────────────────────────────────────────────────┐    │   │
│  │  │              MooncakeStore (Python Wrapper)                      │    │   │
│  │  │  ┌─────────────────────────────────────────────────────────┐    │    │   │
│  │  │  │ from mooncake.store import MooncakeDistributedStore    │    │    │   │
│  │  │  │                                                         │    │    │   │
│  │  │  │ - batch_get_into()  <- zero-copy read                   │    │    │   │
│  │  │  │ - batch_put_from()  <- zero-copy write                  │    │    │   │
│  │  │  │ - register_buffer() <- memory registration              │    │    │   │
│  │  │  └─────────────────────────────────────────────────────────┘    │    │   │
│  │  └──────────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                             │
│                                    │ Python C Extension (pybind11)              │
│                                    ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                     mooncake.store (store.so)                           │   │
│  │              C++ Python Binding (mooncake-integration/)                 │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │   │
│  │  │ class MooncakeStorePyWrapper                                    │   │   │
│  │  │                                                                 │   │   │
│  │  │   batch_get_into(keys, buffer_ptrs, sizes)                      │   │   │
│  │  │   -> store_->batch_get_into()  [GIL Released]                   │   │   │
│  │  │                                                                 │   │   │
│  │  │   batch_put_from(keys, buffer_ptrs, sizes)                      │   │   │
│  │  │   -> store_->batch_put_from()  [GIL Released]                   │   │   │
│  │  └─────────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                             │
│                                    │ C++ Library Linking                          │
│                                    ▼                                             │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ gRPC / TCP / RDMA
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Mooncake Service Layer                              │
│  ┌──────────────────────────────┐  ┌─────────────────────────────────────────┐  │
│  │   Mooncake Master Service    │  │      Mooncake Store (C++ Core)          │  │
│  │       (mooncake_master)      │  │                                         │  │
│  │                              │  │  ┌───────────────────────────────────┐  │  │
│  │  - gRPC server (:55051)      │  │  │ class RealClient : public PyClient│  │  │
│  │  - Metadata HTTP (:18888)    │  │  │                                   │  │  │
│  │  - Global segment management │  │  │  setup() -> init transfer engine  │  │  │
│  │  - Replica management        │  │  │  put() -> write to global segment │  │  │
│  │                              │  │  │  get() -> read from replica       │  │  │
│  └──────────────┬───────────────┘  │  └───────────────────────────────────┘  │  │
│                 │                  └─────────────────────────────────────────┘  │
│                 │                                   │                           │
│                 │ gRPC                              │ RDMA/TCP                  │
│                 ▼                                   ▼                           │
│  ┌──────────────────────────────┐  ┌─────────────────────────────────────────┐  │
│  │   Global Segment (Shared     │  │      Other Nodes / Replicas             │  │
│  │   Memory / Storage)          │  │                                         │  │
│  │                              │  │  - Peer-to-peer transfer                │  │
│  │  - Object storage            │  │  - RDMA zero-copy                       │  │
│  │  - Replica placement         │  │  - TCP fallback                         │  │
│  │  - Load balancing            │  │                                         │  │
│  └──────────────────────────────┘  └─────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 工厂模式：动态加载 Mooncake 后端

### 2.1 问题背景

**为什么需要工厂模式？**

SGLang 支持多种 L3 存储后端：`file`、`mooncake`、`nixl`、`hf3fs`、`aibrix`、`eic`。如果硬编码每种后端的导入逻辑，代码会变得臃肿且难以扩展。

### 2.2 工厂模式的实现

```python
# sglang/srt/mem_cache/storage/backend_factory.py

class StorageBackendFactory:
    """Factory for creating storage backend instances with support for dynamic loading."""

    _registry: Dict[str, Dict[str, Any]] = {}

    @classmethod
    def register_backend(cls, name: str, module_path: str, class_name: str) -> None:
        """注册存储后端（延迟加载）
        
        关键：这里只记录模块路径和类名，不立即导入！
        """
        if name in cls._registry:
            logger.warning(f"Backend '{name}' is already registered, overwriting")

        def loader() -> type[HiCacheStorage]:
            """延迟加载函数 - 只在真正需要时才导入"""
            return cls._load_backend_class(module_path, class_name, name)

        cls._registry[name] = {
            "loader": loader,           # 延迟加载器
            "module_path": module_path, # "sglang.srt.mem_cache.storage.mooncake_store.mooncake_store"
            "class_name": class_name,   # "MooncakeStore"
        }

    @classmethod
    def _load_backend_class(cls, module_path: str, class_name: str, backend_name: str):
        """从模块路径加载后端类"""
        module = importlib.import_module(module_path)
        backend_class = getattr(module, class_name)
        
        # 关键验证：确保继承自 HiCacheStorage
        if not issubclass(backend_class, HiCacheStorage):
            raise TypeError(f"Backend class {class_name} must inherit from HiCacheStorage")
        
        return backend_class
```

### 2.3 为什么 `--hicache-storage-backend mooncake` 可以工作？

```python
# 1. 启动时的注册（模块加载时自动执行）
StorageBackendFactory.register_backend(
    "mooncake",                                        # 后端名称
    "sglang.srt.mem_cache.storage.mooncake_store.mooncake_store",  # 模块路径
    "MooncakeStore",                                   # 类名
)

# 2. SGLang 启动参数解析
# sglang/srt/server_args.py
parser.add_argument(
    "--hicache-storage-backend",
    type=str,
    choices=["file", "mooncake", "hf3fs", "nixl", "aibrix", "dynamic", "eic"],
    default="file",
)

# 3. 创建后端的时刻
# sglang/srt/mem_cache/memory_pool_host.py
class HostKVCache:
    def __init__(..., allocator_type: str = "default"):
        # allocator_type 就是 "mooncake"
        self.allocator = get_allocator_from_storage(allocator_type)
        
        # 最终调用工厂创建存储后端
        self.storage = StorageBackendFactory.create_backend(
            backend_name="mooncake",  # 从命令行传递
            storage_config=storage_config,
            mem_pool_host=self,
        )

# 4. 工厂创建实例
class StorageBackendFactory:
    @classmethod
    def create_backend(cls, backend_name: str, ...):
        # 查找已注册的后端
        if backend_name in cls._registry:
            registry_entry = cls._registry[backend_name]
            
            # 关键：延迟加载！此时才执行 import
            backend_class = registry_entry["loader"]()
            
            # 创建实例
            return cls._create_builtin_backend(
                backend_name, backend_class, storage_config, mem_pool_host
            )
```

### 2.4 延迟加载的优势

```python
# 如果没有延迟加载，启动时会导入所有后端：
import sglang.srt.mem_cache.hicache_storage      # file
import sglang.srt.mem_cache.storage.mooncake_store.mooncake_store  # mooncake
import sglang.srt.mem_cache.storage.nixl.hicache_nixl              # nixl
# ... 其他后端

# 延迟加载只在需要时导入：
if backend_name == "mooncake":
    import sglang.srt.mem_cache.storage.mooncake_store.mooncake_store
```

---

## 3. Mooncake Python Binding 详解

### 3.1 构建过程

```cmake
# mooncake-integration/CMakeLists.txt

# 使用 pybind11 创建 Python 扩展模块
pybind11_add_module(store ${SOURCES}
    store/store_py.cpp    # 主要的 Python binding 代码
)

# 链接 Mooncake Store 库
target_link_libraries(store PUBLIC
    transfer_engine
    mooncake_store
    cachelib_memory_allocator
)

# 安装到 Python 包目录
install(TARGETS store DESTINATION ${PYTHON_SYS_PATH}/mooncake)
```

### 3.2 store_py.cpp 核心结构

```cpp
// mooncake-integration/store/store_py.cpp

#include <pybind11/gil.h>
#include <pybind11/stl.h>

// 关键类：Python 包装器
class MooncakeStorePyWrapper {
public:
    std::shared_ptr<PyClient> store_{nullptr};
    
    // ========== 零拷贝核心接口 ==========
    
    // 从预分配缓冲区批量读取（零拷贝）
    std::vector<int64_t> batch_get_into(
        const std::vector<std::string> &keys,
        const std::vector<uintptr_t> &buffer_ptrs,  // PyTorch tensor data_ptr
        const std::vector<size_t> &sizes
    ) {
        std::vector<void *> buffers;
        for (uintptr_t ptr : buffer_ptrs) {
            buffers.push_back(reinterpret_cast<void *>(ptr));
        }
        
        // 关键：释放 GIL，允许 Python 其他线程执行
        py::gil_scoped_release release_gil;
        
        // 调用 C++ 核心，直接写入用户提供的内存
        return store_->batch_get_into(keys, buffers, sizes);
    }
    
    // 批量写入（零拷贝）
    std::vector<int> batch_put_from(
        const std::vector<std::string> &keys,
        const std::vector<uintptr_t> &buffer_ptrs,
        const std::vector<size_t> &sizes
    ) {
        std::vector<void *> buffers;
        for (uintptr_t ptr : buffer_ptrs) {
            buffers.push_back(reinterpret_cast<void *>(ptr));
        }
        
        py::gil_scoped_release release_gil;
        return store_->batch_put_from(keys, buffers, sizes, ReplicateConfig{});
    }
};

// ========== Python 模块定义 ==========
PYBIND11_MODULE(store, m) {
    // 暴露 MooncakeDistributedStore 类给 Python
    py::class_<MooncakeStorePyWrapper>(m, "MooncakeDistributedStore")
        .def(py::init<>())
        .def("setup", &MooncakeStorePyWrapper::setup)
        .def("batch_get_into", &MooncakeStorePyWrapper::batch_get_into)
        .def("batch_put_from", &MooncakeStorePyWrapper::batch_put_from)
        .def("register_buffer", &MooncakeStorePyWrapper::register_buffer)
        .def("batch_is_exist", &MooncakeStorePyWrapper::batch_is_exist);
}
```

### 3.3 GIL 管理的重要性

```cpp
// 为什么需要 py::gil_scoped_release?

// 错误的做法：持有 GIL 进行 I/O
pybind11::list batch_get_slow(...) {
    // GIL 被持有，其他 Python 线程无法执行
    auto result = store_->batch_get_into(...);  // 网络 I/O 阻塞！
    return result;
}

// 正确的做法：释放 GIL
pybind11::list batch_get_fast(...) {
    {
        py::gil_scoped_release release;  // 释放 GIL
        auto result = store_->batch_get_into(...);  // 网络 I/O
    }  // 重新获取 GIL
    return result;
}
```

---

## 4. 零拷贝内存机制

### 4.1 内存注册流程

```python
# sglang/srt/mem_cache/storage/mooncake_store/mooncake_store.py

class MooncakeStore(HiCacheStorage):
    def register_mem_pool_host(self, mem_pool_host: HostKVCache):
        """注册 Host KV Cache 缓冲区到 Mooncake
        
        这是实现零拷贝的关键步骤！
        """
        buffer = mem_pool_host.kv_buffer
        buffer_ptr = buffer.data_ptr()      # PyTorch Tensor 内存地址
        buffer_size = buffer.numel() * buffer.element_size()
        
        # 将缓冲区注册到 Mooncake
        ret_code = self.store.register_buffer(buffer_ptr, buffer_size)
```

```cpp
// mooncake-integration/store/store_py.cpp

int register_buffer(uintptr_t buffer_addr, size_t capacity) {
    char *buffer = reinterpret_cast<char *>(buffer_addr);
    
    // 注册到 Transfer Engine
    // 这会：
    // 1. 锁定内存页 (mlock)
    // 2. 注册到 RDMA 设备 (ibv_reg_mr)
    // 3. 建立虚拟地址到物理地址的映射
    return engine_->registerLocalMemory(buffer, capacity, kWildcardLocation);
}
```

### 4.2 零拷贝数据传输

```
传统方式（有拷贝）：
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Mooncake     │ ──> │ 临时缓冲区    │ ──> │ PyTorch      │
│ Storage      │     │ (memcpy)     │     │ Tensor       │
└──────────────┘     └──────────────┘     └──────────────┘
    磁盘/RDMA            CPU 内存            GPU 内存

零拷贝方式：
┌──────────────┐                         ┌──────────────┐
│ Mooncake     │ ──────────────────────> │ PyTorch      │
│ Storage      │    RDMA READ 直接写入    │ Tensor       │
└──────────────┘    用户指定的内存地址     └──────────────┘
    磁盘/RDMA            已注册内存          GPU 内存
```

### 4.3 内存布局和对齐

```python
# sglang/srt/mem_cache/memory_pool_host.py

class MHATokenToKVPoolHost:
    def init_kv_buffer(self):
        """初始化 KV Buffer，不同布局影响存储效率"""
        
        if self.layout == "page_first":
            # 布局: (2, size, layer_num, head_num, head_dim)
            # 2 = K and V
            dims = (2, self.size, self.layer_num, self.head_num, self.head_dim)
        elif self.layout == "layer_first":
            # 布局: (2, layer_num, size, head_num, head_dim)
            dims = (2, self.layer_num, self.size, self.head_num, self.head_dim)
        elif self.layout == "page_first_direct":
            # 直接映射到页面
            dims = (2, self.page_num, self.layer_num, self.page_size, 
                    self.head_num, self.head_dim)
        
        # 分配内存
        self.kv_buffer = torch.empty(dims, dtype=self.dtype, device="cpu", 
                                     pin_memory=self.pin_memory)
```

---

## 5. 存储后端初始化全流程

### 5.1 调用链

```
1. SGLang 启动
   └─> python -m sglang.launch_server --hicache-storage-backend mooncake

2. 参数解析
   └─> server_args.py: ServerArgs.hicache_storage_backend = "mooncake"

3. ModelRunner 初始化
   └─> model_runner.py: 创建 HiRadixCache

4. HiRadixCache 初始化
   └─> hiradix_cache.py: 创建 MHATokenToKVPoolHost

5. Host KV Cache 初始化
   └─> memory_pool_host.py: 
       - 分配 Host 内存
       - 调用 get_allocator_from_storage("mooncake")

6. 存储后端创建
   └─> backend_factory.py:
       - StorageBackendFactory.create_backend("mooncake", ...)
       - 延迟加载 mooncake_store 模块
       - 创建 MooncakeStore 实例

7. MooncakeStore 初始化
   └─> mooncake_store.py:
       - 从环境变量读取配置
       - 调用 MooncakeDistributedStore.setup()

8. C++ 层初始化
   └─> store_py.cpp:
       - 创建 TransferEngine
       - 建立与 Master 的连接
       - 注册内存缓冲区
```

### 5.2 配置加载的优先级

```python
# mooncake_store.py

class MooncakeStore:
    def __init__(self, storage_config, mem_pool):
        # 配置优先级（从高到低）：
        # 1. extra_config（代码传入）
        # 2. 配置文件（SGLANG_HICACHE_MOONCAKE_CONFIG_PATH）
        # 3. 环境变量（MOONCAKE_MASTER, MOONCAKE_PROTOCOL 等）
        
        if extra_config is not None:
            self.config = MooncakeStoreConfig.load_from_extra_config(extra_config)
        elif envs.SGLANG_HICACHE_MOONCAKE_CONFIG_PATH.is_set():
            self.config = MooncakeStoreConfig.from_file()
        else:
            self.config = MooncakeStoreConfig.load_from_env()
```

---

## 6. KV Cache 数据流

### 6.1 写入流程（Write-Through）

```python
# 场景：新的请求计算出了新的 KV，需要缓存

class HiRadixCache:
    def insert(self, key, value):
        """插入新的 KV Cache"""
        
        # 1. 写入 L1 (GPU)
        device_indices = self.device_pool.alloc(value.size)
        self.device_pool.copy_from(value, device_indices)
        
        # 2. 写入 L2 (Host) - Write-Through
        host_indices = self.host_pool.alloc(value.size)
        self.host_pool.copy_from_device(device_indices, host_indices)
        
        # 3. 写入 L3 (Mooncake) - 异步后台写入
        if self.enable_storage:
            self.cache_controller.backup_to_storage(
                keys=[key],
                host_indices=host_indices
            )
```

```python
# mooncake_store.py

class MooncakeStore:
    def batch_set_v1(self, keys, host_indices, extra_info):
        """将 Host 中的 KV 写入 Mooncake"""
        
        # 1. 准备 key 列表
        # key 格式: "{prefix_hash}_{token_pos}_{tp_rank}_{pp_rank}_k"
        key_strs, buffer_ptrs, buffer_sizes = self._batch_preprocess(keys, host_indices)
        
        # 2. 检查哪些 key 已经存在（避免重复写入）
        exist_result = self._batch_exist(key_strs)
        
        # 3. 过滤已存在的 key
        set_keys = []
        set_buffer_ptrs = []
        for i, key in enumerate(key_strs):
            if exist_result[i] != 1:  # 不存在
                set_keys.append(key)
                set_buffer_ptrs.append(buffer_ptrs[i])
        
        # 4. 批量写入 Mooncake
        if set_keys:
            # 调用 C++ 层的零拷贝写入
            put_results = self._put_batch_zero_copy_impl(
                set_keys, set_buffer_ptrs, set_buffer_sizes
            )
```

### 6.2 读取流程（Cache Miss）

```python
# 场景：请求需要之前缓存的 KV，但 L1/L2 未命中

class HiRadixCache:
    def match_prefix(self, rid, input_ids):
        """前缀匹配，查找缓存的 KV"""
        
        # 1. 在 L1 (GPU) 中查找
        gpu_match = self._match_in_device_cache(rid, input_ids)
        if gpu_match.hit:
            return gpu_match
        
        # 2. 在 L2 (Host) 中查找
        host_match = self._match_in_host_cache(rid, input_ids)
        if host_match.hit:
            # 从 Host 拷贝到 GPU
            self._load_from_host_to_device(host_match)
            return host_match
        
        # 3. 在 L3 (Mooncake) 中查找
        storage_match = self._match_in_storage(rid, input_ids)
        if storage_match.hit:
            # 3a. 从 Mooncake 加载到 Host
            self._load_from_storage_to_host(storage_match)
            # 3b. 从 Host 拷贝到 GPU
            self._load_from_host_to_device(storage_match)
            return storage_match
        
        return None
```

```python
# mooncake_store.py

class MooncakeStore:
    def batch_get_v1(self, keys, host_indices, extra_info):
        """从 Mooncake 读取 KV 到 Host Memory"""
        
        # 1. 准备 key 列表和缓冲区信息
        key_strs, buffer_ptrs, buffer_sizes = self._batch_preprocess(keys, host_indices)
        
        # 2. 调用 C++ 层的零拷贝读取
        # 数据将直接写入 buffer_ptrs 指向的 Host Memory
        get_results = self._get_batch_zero_copy_impl(
            key_strs, buffer_ptrs, buffer_sizes
        )
        
        # 3. 处理结果
        # get_results[i] > 0: 成功读取的字节数
        # get_results[i] < 0: 错误码
        return self._batch_postprocess(get_results, is_set_operate=False)
```

---

## 7. RPC 通信机制

### 7.1 Master Service API

```protobuf
// Mooncake Master 提供的 gRPC 服务

service MasterService {
    // Segment 管理
    rpc MountSegment(MountSegmentRequest) returns (MountSegmentResponse);
    rpc UnmountSegment(UnmountSegmentRequest) returns (UnmountSegmentResponse);
    
    // 对象存储
    rpc Put(PutRequest) returns (PutResponse);
    rpc Get(GetRequest) returns (GetResponse);
    rpc Delete(DeleteRequest) returns (DeleteResponse);
    
    // 元数据查询
    rpc GetReplicaDesc(GetReplicaDescRequest) returns (GetReplicaDescResponse);
    rpc GetAllSegments(Empty) returns (GetAllSegmentsResponse);
}
```

### 7.2 元数据服务 HTTP API

```python
# Mooncake Master 启动时启用 HTTP Metadata Server
# mooncake_master --enable_http_metadata_server --http_metadata_server_port 18888

# 提供的 HTTP 端点：
GET /metadata              # 获取集群元数据
GET /get_all_segments      # 获取所有 segment 信息
GET /health                # 健康检查
```

### 7.3 客户端-服务端交互流程

```
1. 初始化连接
   ┌──────────┐                    ┌──────────────┐
   │ SGLang   │ ── setup() ──────> │   Master     │
   │ Client   │                    │   :55051     │
   └──────────┘                    └──────────────┘
        │                                 │
        │ 2. 返回集群拓扑                  │
        │<────────────────────────────────┘
        │
        ▼
   3. 打开 Segment
   ┌──────────┐                    ┌──────────────┐
   │ SGLang   │ ── openSegment() ─>│  Transfer    │
   │ Client   │                    │  Engine      │
   └──────────┘                    └──────────────┘

4. 数据传输（RDMA/TCP）
   ┌──────────┐                    ┌──────────────┐
   │ SGLang   │ ── RDMA READ ────> │  Global      │
   │ Client   │    / TCP           │  Segment     │
   └──────────┘                    └──────────────┘
```

---

## 8. 关键代码走读

### 8.1 plan-1.sh 中的关键环境变量

```bash
# 这些环境变量如何被 MooncakeStore 读取？

export MOONCAKE_MASTER="127.0.0.1:55051"
export MOONCAKE_PROTOCOL="tcp"
export MOONCAKE_TE_META_DATA_SERVER="http://127.0.0.1:18888/metadata"
export MOONCAKE_GLOBAL_SEGMENT_SIZE="4294967296"  # 4GB

# Python 代码中的读取点：
# sglang/srt/environ.py
class EnvConfig:
    MOONCAKE_MASTER = EnvStr(None)               # 读取 MOONCAKE_MASTER
    MOONCAKE_PROTOCOL = EnvStr("tcp")            # 读取 MOONCAKE_PROTOCOL
    MOONCAKE_TE_META_DATA_SERVER = EnvStr("P2PHANDSHAKE")
    MOONCAKE_GLOBAL_SEGMENT_SIZE = EnvStr("4gb")

# mooncake_store.py
config = MooncakeStoreConfig(
    master_server_address=envs.MOONCAKE_MASTER.get(),
    protocol=envs.MOONCAKE_PROTOCOL.get(),
    metadata_server=envs.MOONCAKE_TE_META_DATA_SERVER.get(),
    global_segment_size=parse_size(envs.MOONCAKE_GLOBAL_SEGMENT_SIZE.get()),
)
```

### 8.2 Mooncake Master 启动流程

```bash
# plan-1.sh 启动命令
mooncake_master \
    --rpc_port "55051" \
    --http_metadata_server_port "18888" \
    --enable_http_metadata_server \
    --metrics_port 9003

# 对应 C++ 代码中的参数解析
# mooncake-store/src/master/main.cpp

int main(int argc, char** argv) {
    gflags::ParseCommandLineFlags(&argc, &argv, true);
    
    // 1. 创建 Master Service
    MasterServiceImpl service;
    
    // 2. 启动 gRPC 服务器
    grpc::ServerBuilder builder;
    builder.AddListeningPort("0.0.0.0:55051", grpc::InsecureServerCredentials());
    builder.RegisterService(&service);
    std::unique_ptr<grpc::Server> server(builder.BuildAndStart());
    
    // 3. 启动 HTTP Metadata Server（如果启用）
    if (FLAGS_enable_http_metadata_server) {
        HttpMetadataServer http_server(FLAGS_http_metadata_server_port);
        http_server.Start();
    }
    
    server->Wait();
}
```

### 8.3 完整的数据拷贝链

```python
# 场景：从 Mooncake 加载 KV 到 GPU

# 1. Python 层发起请求
# sglang/srt/mem_cache/hiradix_cache.py
keys = ["kv_hash_1", "kv_hash_2", ...]
host_indices = torch.tensor([10, 11, 12, ...])  # Host buffer 中的位置

# 2. 调用 MooncakeStore
# sglang/srt/mem_cache/storage/mooncake_store/mooncake_store.py
results = self.storage.batch_get_v1(keys, host_indices, extra_info)

# 3. 准备缓冲区指针
key_strs = ["kv_hash_1_0_0_k", "kv_hash_1_0_0_v", ...]
ptr_list = []
for idx in host_indices:
    # 获取每个页面在 Host Buffer 中的地址
    ptr = self.mem_pool_host.get_page_buffer_ptr(idx)
    ptr_list.append(ptr)

# 4. 调用 C++ 层（零拷贝）
# store_py.cpp
self.store.batch_get_into(keys, ptr_list, sizes)
    -> py::gil_scoped_release release  # 释放 GIL
    -> store_->batch_get_into(keys, buffers, sizes)

# 5. C++ 核心层
# mooncake-store/src/client/real_client.cpp
int64_t RealClient::get_into(const string& key, void* buffer, size_t size) {
    // 1. 查询 Master 获取副本位置
    ReplicaDesc replica = master_client_->GetReplicaDesc(key);
    
    // 2. 使用 Transfer Engine 读取数据
    // 如果是 RDMA：直接 RDMA READ 到 buffer
    // 如果是 TCP：通过 socket 接收写入 buffer
    transfer_engine_->submitTransfer(batch_id, {
        .opcode = READ,
        .source = buffer,           // 目标地址（用户提供的 buffer）
        .target_offset = replica.addr,  // 源地址（远程 segment）
        .length = size,
    });
}

# 6. 回到 Python，数据已在 Host Buffer 中
# 现在需要拷贝到 GPU
# sglang/srt/mem_cache/memory_pool_host.py
def load_to_device_per_layer(self, device_pool, host_indices, device_indices, layer_id):
    # 使用 CUDA kernel 或 memcpy 拷贝到 GPU
    transfer_kv_per_layer(
        self.k_buffer[layer_id],    # Host K
        self.v_buffer[layer_id],    # Host V
        device_pool.k_buffer[layer_id],
        device_pool.v_buffer[layer_id],
        host_indices,
        device_indices,
    )
```

---

## 总结：为什么 SGLang + Mooncake 可以工作？

1. **插件化架构**：SGLang 使用工厂模式动态加载存储后端，Mooncake 作为其中一个插件注册到系统中

2. **统一接口**：MooncakeStore 继承 HiCacheStorage，实现了标准的三层缓存接口（batch_get/batch_set/batch_exists）

3. **零拷贝传输**：通过内存注册机制，Mooncake 可以直接读写 SGLang 的 Host KV Cache 缓冲区，避免数据拷贝

4. **GIL 管理**：Python Binding 在网络 I/O 时释放 GIL，允许 SGLang 的其他线程继续处理请求

5. **分层缓存策略**：L1 (GPU) → L2 (Host) → L3 (Mooncake) 的层级结构，配合 Write-Through 和 Prefetch 机制，最大化缓存命中率

6. **分布式存储**：Mooncake Master 管理全局 Segment，支持多副本和负载均衡，实现跨节点的 KV Cache 共享
