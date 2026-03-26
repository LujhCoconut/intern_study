# Mooncake 零拷贝（Zero-Copy）机制深度解析

**日期**: 2026-03-25  
**主题**: Mooncake 高性能数据传输的核心机制

---

## 目录

1. [什么是零拷贝](#1-什么是零拷贝)
2. [Mooncake 三层零拷贝架构](#2-mooncake-三层零拷贝架构)
3. [代码实现详解](#3-代码实现详解)
4. [完整数据流示例](#4-完整数据流示例)
5. [性能收益分析](#5-性能收益分析)
6. [关键总结](#6-关键总结)

---

## 1. 什么是零拷贝

### 1.1 传统方式的问题

```
场景：从存储读取 1GB KV Cache 到 Host Memory

传统方式（4次数据拷贝）：
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Storage │───→│  Kernel  │───→│  User    │───→│  Python  │───→│  PyTorch │
│  (Disk/  │    │  Buffer  │    │  Buffer  │    │  Object  │    │  Tensor  │
│   RDMA)  │    │  (内核)   │    │  (C++)   │    │  (转换)  │    │  (目标)  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │               │               │
   DMA拷贝       内核→用户      跨语言拷贝       Python对象      创建Tensor
   (硬件完成)    空间拷贝        (memcpy)        内存分配        视图
     
时间:  ~5ms        ~20ms          ~20ms           ~30ms          ~10ms
CPU:   0%          100%           100%            100%           50%
总计:  ~85ms，CPU 占用 350%
```

**问题**:
- 数据在内存中多次复制
- CPU 参与每一次拷贝，消耗计算资源
- 浪费内存带宽（4倍数据量通过内存总线）
- 延迟高，吞吐量低

### 1.2 零拷贝的核心思想

```
传统方式: 数据必须"流经"CPU，CPU 把数据从 A 复制到 B
零拷贝:   让硬件（RDMA 网卡/磁盘控制器）直接把数据写入目标内存

零拷贝方式（1次数据拷贝）：
┌──────────┐                    ┌──────────────────────────────────────┐
│  Storage │─── RDMA READ ─────→│  Host KV Cache (PyTorch Tensor)      │
│  (Remote)│    直接写入目标     │  - 已注册内存 (Registered Memory)    │
│          │    绕过CPU拷贝     │  - Pinned Memory (页面锁定)          │
└──────────┘                    └──────────────────────────────────────┘

时间:  ~20ms（仅一次 DMA 传输）
CPU:   <5%（仅提交请求和等待完成）
```

---

## 2. Mooncake 三层零拷贝架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Mooncake 零拷贝三层架构                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Level 1: 存储 → Host Memory (L2)                                           │
│  ┌─────────────────────────────────┐    ┌─────────────────────────────────┐ │
│  │  Storage (Remote/Distributed)   │    │  Host KV Cache                  │ │
│  │  - Segment on Node C            │───→│  - PyTorch Tensor               │ │
│  │  - Stored on SSD/Memory         │    │  - Pinned CPU Memory            │ │
│  └─────────────────────────────────┘    └─────────────────────────────────┘ │
│            RDMA READ 直接写入目标内存                                       │
│            绕过 CPU，零拷贝                                                 │
│                                                                             │
│  Level 2: Host Memory (L2) → GPU Memory (L1)                               │
│  ┌─────────────────────────────────┐    ┌─────────────────────────────────┐ │
│  │  Host KV Cache                  │    │  GPU KV Cache                   │ │
│  │  (Pinned Memory)                │───→│  (Device Memory)                │ │
│  │  - Registered with CUDA         │    │  - HBM on GPU                   │ │
│  └─────────────────────────────────┘    └─────────────────────────────────┘ │
│            GPU Direct RDMA (GDR) 或 cudaMemcpyAsync                         │
│            GPU 直接访问 Host Memory，CPU 不参与                             │
│                                                                             │
│  Level 3: Python ↔ C++ (跨语言零拷贝)                                       │
│  ┌─────────────────────────────────┐    ┌─────────────────────────────────┐ │
│  │  Python Layer                   │    │  C++ Layer (Mooncake Store)     │ │
│  │  - torch.Tensor                 │←──→│  - PyBind11 Wrapper             │ │
│  │  - data_ptr() 获取地址          │    │  - 直接读写内存                 │ │
│  │  - memoryview 共享内存          │    │  - 无序列化开销                 │ │
│  └─────────────────────────────────┘    └─────────────────────────────────┘ │
│            共享内存地址，无数据转换                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 代码实现详解

### 3.1 关键：内存注册（Memory Registration）

```cpp
// mooncake-integration/store/store_py.cpp
class MooncakeStorePyWrapper {
public:
    // 注册 PyTorch Tensor 的内存到 Mooncake
    int register_buffer(uintptr_t buffer_addr, size_t capacity) {
        char* buffer = reinterpret_cast<char*>(buffer_addr);
        
        // 关键操作 1: 锁定内存页（防止被 swap 到磁盘）
        // mlock(buffer, capacity);
        
        // 关键操作 2: 向 Transfer Engine 注册
        // 这会把内存注册到 RDMA 设备（ibv_reg_mr）
        return engine_->registerLocalMemory(buffer, capacity, kWildcardLocation);
    }
};
```

**为什么要注册？**

| 原因 | 说明 |
|------|------|
| **页面锁定** | `mlock()` 防止 OS 将内存 swap 到磁盘，确保物理地址稳定 |
| **DMA 映射** | RDMA 网卡需要知道物理地址才能直接读写 |
| **安全隔离** | 只有注册的内存才能被远程 RDMA 访问（防止非法访问） |
| **注册一次** | 只需注册一次，后续可重复使用 |

### 3.2 Python 层：注册整个 L2 缓冲区

```python
# sglang/srt/mem_cache/storage/mooncake_store/mooncake_store.py
class MooncakeStore(HiCacheStorage):
    def register_mem_pool_host(self, mem_pool_host: HostKVCache):
        """注册整个 Host KV Cache 缓冲区到 Mooncake"""
        buffer = mem_pool_host.kv_buffer  # PyTorch Tensor
        
        # 获取 Tensor 的内存地址
        buffer_ptr = buffer.data_ptr()      # 如: 0x7f8a3c200000
        buffer_size = buffer.numel() * buffer.element_size()
        
        # 注册到 Mooncake（一次注册，多次使用）
        ret_code = self.store.register_buffer(buffer_ptr, buffer_size)
        
        if ret_code:
            raise RuntimeError(f"Failed to register buffer: {ret_code}")
```

### 3.3 C++ 层：读取时的零拷贝实现

```cpp
// store_py.cpp - batch_get_into
pybind11::list batch_get_into(
    const std::vector<std::string>& keys,
    const std::vector<uintptr_t>& buffer_ptrs,  // 目标内存地址（已注册）
    const std::vector<size_t>& sizes
) {
    // 转换地址格式
    std::vector<void*> buffers;
    for (uintptr_t ptr : buffer_ptrs) {
        buffers.push_back(reinterpret_cast<void*>(ptr));
    }
    
    // 关键：释放 Python GIL，允许其他 Python 线程执行
    py::gil_scoped_release release_gil;
    
    // C++ 层：直接写入用户提供的内存地址
    // RDMA 网卡直接把数据写入这些地址，不经过 CPU 拷贝
    std::vector<int64_t> total_lengths = store_->batch_get_into(
        keys, buffers, sizes
    );
    
    // 重新获取 GIL，返回结果
    py::gil_scoped_acquire acquire_gil;
    
    // 返回读取的字节数（不是数据本身，数据已经在 buffer 里了）
    py::list results_list;
    for (auto length : total_lengths) {
        results_list.append(length);
    }
    return results_list;
}
```

**关键点**：
- `buffer_ptrs` 是 PyTorch Tensor 的内存地址
- 数据直接写入这些地址，**没有临时缓冲区**
- `gil_scoped_release`：网络 I/O 时不阻塞其他 Python 线程

### 3.4 C++ 核心层：RDMA 零拷贝传输

```cpp
// real_client.cpp
std::vector<int64_t> RealClient::batch_get_into(
    const std::vector<std::string>& keys,
    const std::vector<void*>& buffers,
    const std::vector<size_t>& sizes
) {
    std::vector<int64_t> results;
    
    for (size_t i = 0; i < keys.size(); i++) {
        // 1. 查询 Master：这个 key 的数据在哪个 Segment
        ReplicaDesc replica = master_client_->GetReplicaDesc(keys[i]);
        
        // 2. 获取 RDMA 连接（到目标节点）
        Transport::SegmentHandle handle = engine_->openSegment(replica.segment_name);
        
        // 3. 创建 RDMA 读请求
        TransferRequest entry{
            .opcode = TransferRequest::READ,     // 读操作
            .length = sizes[i],                   // 读取长度
            .source = buffers[i],                 // 目标地址（本地已注册内存）
            .target_id = handle,                  // 远程 Segment
            .target_offset = replica.offset       // 远程偏移量
        };
        
        // 4. 提交 RDMA 请求（硬件执行，CPU 不参与拷贝）
        batch_id_t batch_id = engine_->allocateBatchID(1);
        engine_->submitTransfer(batch_id, {entry});
        
        // 5. 等待完成（异步或同步）
        TransferStatus status;
        while (true) {
            engine_->getTransferStatus(batch_id, 0, status);
            if (status.s == TransferStatusEnum::COMPLETED) {
                results.push_back(sizes[i]);
                break;
            }
        }
        
        engine_->freeBatchID(batch_id);
    }
    
    return results;
}
```

### 3.5 RDMA 零拷贝 vs 传统 TCP

```
传统 TCP 读取（CPU 参与）：
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ Remote   │────→│ NIC      │────→│ Kernel   │────→│ User     │
│ Memory   │     │ (网卡)   │     │ Buffer   │     │ Buffer   │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                        │                │                │
                        └────────────────┴────────────────┘
                              CPU 拷贝数据（消耗 CPU 周期）

RDMA 零拷贝读取（硬件完成）：
┌──────────┐                                          ┌──────────┐
│ Remote   │──────────────────────────────────────────→│ Local    │
│ Memory   │         RDMA READ (硬件执行)               │ Memory   │
│          │                                          │ (已注册) │
└──────────┘                                          └──────────┘
                              │
                              └→ 数据直接通过 PCIe 和网卡传输
                                 CPU 只收到"传输完成"中断
```

### 3.6 Python ↔ C++ 跨语言零拷贝

```cpp
// 从 PyTorch Tensor 提取信息（不拷贝数据）
struct PyTensorInfo {
    uintptr_t data_ptr;         // Tensor 内存地址
    size_t tensor_size;         // Tensor 大小
    TensorMetadata metadata;    // 元数据（shape, dtype）
};

PyTensorInfo extract_tensor_info(const py::object& tensor) {
    PyTensorInfo info;
    
    // 直接获取内存地址
    info.data_ptr = tensor.attr("data_ptr")().cast<uintptr_t>();
    
    // 获取大小
    size_t numel = tensor.attr("numel")().cast<size_t>();
    size_t element_size = tensor.attr("element_size")().cast<size_t>();
    info.tensor_size = numel * element_size;
    
    // 获取形状和类型（用于序列化元数据）
    py::tuple shape = tensor.attr("shape").cast<py::tuple>();
    info.metadata.ndim = shape.size();
    for (int i = 0; i < shape.size(); i++) {
        info.metadata.shape[i] = shape[i].cast<int64_t>();
    }
    
    return info;  // 只返回指针和元数据，不拷贝数据！
}

// 使用 memoryview 实现零拷贝转换
pybind11::object buffer_to_tensor(BufferHandle* buffer_handle) {
    size_t total_length = buffer_handle->size();
    
    // 创建 memoryview（共享内存，不拷贝）
    py::memoryview mv(py::buffer_info(
        buffer_handle->ptr(),           // 内存地址
        total_length,                   // 大小
        false                           // 只读
    ));
    
    // 从 memoryview 创建 NumPy array（零拷贝）
    py::object np_array = py::module_::import("numpy").attr("frombuffer")(
        mv, py::dtype("uint8")
    );
    
    // 从 NumPy 创建 PyTorch Tensor（零拷贝，共享内存）
    py::object tensor = py::module_::import("torch").attr("from_numpy")(np_array);
    
    return tensor;
}
```

---

## 4. 完整数据流示例

### 场景：从 Mooncake L3 读取 KV Cache 到 SGLang L2

```python
# Step 1: HiRadixCache 发现需要从 L3 加载
keys = ["hash_token_1", "hash_token_2", "hash_token_3"]
host_indices = torch.tensor([100, 101, 102])  # L2 中的目标位置

# Step 2: 调用 MooncakeStore（Python 层）
results = self.storage.batch_get_v1(keys, host_indices)

# Step 3: MooncakeStore 准备 buffer 指针
def batch_get_v1(self, keys, host_indices):
    # 3.1 准备 key 列表
    key_strs = []
    for key in keys:
        key_strs.append(f"{key}_{self.mha_suffix}_k")  # K cache
        key_strs.append(f"{key}_{self.mha_suffix}_v")  # V cache
    
    # 3.2 获取 buffer 指针（零拷贝关键！）
    ptr_list = []
    for idx in host_indices:
        # 获取每个 page 在 Host Buffer 中的地址
        ptr = self.mem_pool_host.get_page_buffer_ptr(idx)
        ptr_list.append(ptr)
    
    # 3.3 调用 C++ 接口（传递指针，不是数据！）
    return self.store.batch_get_into(key_strs, ptr_list, sizes)

# Step 4: C++ 层处理
def batch_get_into(self, keys, buffer_ptrs, sizes):
    for i, key in enumerate(keys):
        # 4.1 查询 Master 获取 replica 位置
        replica = master_client_->GetReplicaDesc(key)
        # 返回: Segment "192.168.1.10:12345", offset 4096
        
        # 4.2 RDMA 直接读取
        entry = TransferRequest{
            opcode: READ,
            source: buffer_ptrs[i],           # 目标：SGLang 的 Host Buffer
            target_id: handle,                 # 源：Mooncake Segment
            target_offset: replica.offset,     # 源偏移
            length: sizes[i]
        }
        engine_->submitTransfer(batch_id, [entry])

# Step 5: RDMA 硬件执行（零拷贝发生在这里）
#
# 5.1 RDMA 网卡通过 PCIe 读取远程 Segment 的内存
# 5.2 通过 DMA 写入本地已注册的 Host Buffer
# 5.3 CPU 完全不参与数据拷贝！
#
# 数据流：
# Remote Node Memory ──RDMA──→ Local Host Buffer
#       (Segment)              (PyTorch Tensor)
#            ↓                      ↓
#      ┌─────────┐            ┌─────────┐
#      │ 数据在  │  ────→     │ 数据在  │
#      │ Node C  │   直接     │ Node A  │
#      │ SSD/HBM │   写入     │ DRAM    │
#      └─────────┘            └─────────┘

# Step 6: 数据现在已经在 SGLang 的 L2 Host Buffer 中
# 没有经历过任何中间缓冲区！
```

### 内存地址传递流程

```
Python 层:                   C++ 层:                     硬件层:
┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
│ torch.Tensor     │        │ uintptr_t        │        │ Physical Address │
│ (逻辑地址)        │───────→│ (整数地址)       │───────→│ (DMA 使用)       │
│                  │        │                  │        │                  │
│ data_ptr()       │        │ reinterpret_cast │        │ ibv_reg_mr       │
│ = 0x7f8a3c200000 │        │ = 0x7f8a3c200000 │        │ = 映射到物理地址  │
└──────────────────┘        └──────────────────┘        └──────────────────┘
       │                           │                           │
       │                           │                           │
       └───────────────────────────┴───────────────────────────┘
                           同一内存地址
                           共享同一块内存
                           无数据拷贝
```

---

## 5. 性能收益分析

### 5.1 定量对比

| 操作 | 数据量 | 传统方式 | 零拷贝方式 | 加速比 | CPU 节省 |
|------|--------|----------|------------|--------|----------|
| L3 → L2 读取 | 1 GB | ~100 ms | ~20 ms | **5x** | **90%** |
| L2 → L3 写入 | 1 GB | ~100 ms | ~20 ms | **5x** | **90%** |
| L2 → L1 加载 | 1 GB | ~50 ms | ~25 ms | **2x** | **50%** |
| Python ↔ C++ | 1 GB | ~30 ms | ~0.1 ms | **300x** | **99%** |

### 5.2 定性收益

```
1. 延迟降低
   └── 1GB 数据传输: 100ms → 20ms（减少 80ms）
   └── 对于推理请求，这是关键的尾延迟优化

2. 吞吐量提升
   └── 传统: CPU 成为瓶颈，最大 5GB/s
   └── 零拷贝: 瓶颈在网卡，可达 50-100GB/s
   └── 提升 10-20x

3. CPU 利用率优化
   └── 传统: CPU 100% 忙于拷贝数据
   └── 零拷贝: CPU <10%，可用于执行推理计算
   └── 节省的 CPU 可处理更多并发请求

4. 内存带宽节省
   └── 传统: 数据经过 4 次内存拷贝，消耗 4x 带宽
   └── 零拷贝: 只经过 1 次 DMA，节省 75% 内存带宽
   └── 内存带宽可用于模型计算
```

### 5.3 实际场景收益

```
场景：70B 模型，batch_size=32，序列长度 4K

KV Cache 大小计算：
- 每层 KV: 2 (K+V) × 32 (head) × 128 (dim) × 4096 (seq) × 2 (fp16) = 128 MB
- 80 层: 128 MB × 80 = 10.24 GB

传统方式（无 L3 缓存）:
- 每个请求都重新计算全部 KV
- 计算时间: ~500ms

使用 Mooncake L3 + 零拷贝:
- 80% KV 命中 L3，只需加载
- 加载时间: 10GB × 80% × 20ms/GB = 160ms
- 计算时间: 10GB × 20% × 500ms = 100ms
- 总时间: ~260ms（比传统快 48%）
```

---

## 6. 关键总结

### 6.1 零拷贝的三层含义

| 层级 | 零拷贝实现 | 关键技术 | 代码位置 |
|------|-----------|----------|----------|
| **Storage → Host** | RDMA 直接写入已注册内存 | `registerLocalMemory`, `ibv_reg_mr` | `store_py.cpp` |
| **Host → GPU** | GPU Direct RDMA / Pinned Memory | `cudaHostRegister`, `cudaMemcpyAsync` | `memory_pool_host.py` |
| **Python ↔ C++** | memoryview + Buffer Protocol | `data_ptr()`, `memoryview`, `numpy` | `store_py.cpp` |

### 6.2 为什么能做到零拷贝？

```
1. 内存注册（Memory Registration）
   └── 提前告诉硬件内存的物理地址
   └── 锁定页面，防止 swap
   └── 建立 DMA 映射

2. RDMA 技术（Remote Direct Memory Access）
   └── 网卡直接读写内存
   └── 绕过 CPU 和操作系统内核
   └── 硬件完成数据传输

3. Python Buffer Protocol
   └── 共享内存地址
   └── 不序列化/反序列化数据
   └── memoryview 提供零拷贝视图
```

### 6.3 使用条件与限制

```python
# 1. 内存必须注册（一次性）
store.register_buffer(ptr, size)

# 2. 内存必须锁定（Pinned）
tensor = torch.empty(..., pin_memory=True)

# 3. 地址必须对齐
# RDMA 通常要求 4KB 或 64B 对齐
# 否则可能导致性能下降或失败

# 4. 内存必须持续有效
# 在 RDMA 传输完成前，内存不能被释放
# 否则会导致数据损坏或程序崩溃
```

### 6.4 核心设计哲学

```
传统系统: 数据移动靠 CPU 拷贝（灵活但慢）
Mooncake: 数据移动靠硬件 DMA（受限但快）

权衡：
├── 牺牲灵活性（内存必须预先注册，不能随意分配）
├── 换取性能（10-100x 吞吐量提升，延迟降低 80%）
└── 对于 LLM KV Cache 场景，这是值得的（大块数据，可预测访问）
```

零拷贝是 Mooncake 实现 **TB/s 级聚合带宽** 和 **微秒级延迟** 的核心技术，也是其区别于传统存储系统的关键优势！
