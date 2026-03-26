# HiCache 存储写入流程代码详解

> 从 `write_storage()` 到 Mooncake Store 的完整调用链深度解析

## 1. 核心参数辨析

### 1.1 hash_value vs prefix_keys vs last_hash

```python
class StorageOperation:
    def __init__(
        self,
        host_indices: torch.Tensor,
        token_ids: List[int],
        last_hash: Optional[str] = None,      # 单个字符串：node的最后一个page hash
        hash_value: Optional[List[str]] = None,  # 列表：node的所有pages hash
        prefix_keys: Optional[List[str]] = None, # 列表：祖先node的hash链
    ):
```

| 参数 | 类型 | 含义 | 用途 |
|------|------|------|------|
| `last_hash` | `str` | 当前node的**最后一个page**的hash | 快速身份标识，O(1)检查是否已存储 |
| `hash_value` | `List[str]` | 当前node的**所有pages**的hash列表 | 实际要存储的内容 |
| `prefix_keys` | `List[str]` | 从root到parent的所有hash | 建立依赖链，前缀恢复时使用 |

**示例**：
```
Radix Tree: root -> "Hello"(A) -> " world!"(B) -> " How"(C)

Node A ("Hello"):
  - last_hash = "a1b2..." (唯一的page)
  - hash_value = ["a1b2..."]
  - prefix_keys = []

Node B (" world!"):
  - last_hash = "g7h8..." (最后一个page)
  - hash_value = ["e5f6...", "g7h8..."]  (2个pages)
  - prefix_keys = ["a1b2..."]  (父节点A的hash)

Node C (" How"):
  - last_hash = "i9j0..."
  - hash_value = ["i9j0..."]
  - prefix_keys = ["a1b2...", "e5f6...", "g7h8..."]  (祖先A和B)
```

---

## 2. 异步任务调度机制

### 2.1 StorageOperation.counter 的作用

```python
class StorageOperation:
    counter = 0  # 类变量，全局共享
    
    def __init__(self, ...):
        self.id = StorageOperation.counter  # 分配唯一ID
        StorageOperation.counter += 1       # 原子递增
```

**用途**：
- 每个异步操作有全局唯一标识
- 便于追踪、调试和回调匹配
- 支持同步等待：`wait_for_operation(op_id)`

**线程安全性**：
> 虽然GIL保证了解释器级别的互斥，但这里线程安全主要靠"StorageOperation只在单一线程（主调度线程）创建"。若多线程同时创建，仍需加锁。

### 2.2 多线程模型

```python
# cache_controller.py 中启动后台线程
def start(self):
    # 生产者：主线程调用 write_storage()
    # 消费者：后台线程执行 _page_backup()
    self.backup_thread = threading.Thread(
        target=self.backup_thread_func,  # ← 后台线程入口
        daemon=True
    )
    self.backup_thread.start()
```

**线程检查方法**：
```python
import threading
print(f"活动线程数: {threading.active_count()}")
for t in threading.enumerate():
    print(f"  - {t.name}")
```

### 2.3 backup_thread_func 阻塞机制

```python
def backup_thread_func(self):
    while not self.storage_stop_event.is_set():
        try:
            # block=True: 队列为空时阻塞等待（不是轮询！）
            # timeout=1: 最多等1秒，超时抛Empty检查stop信号
            operation = self.backup_queue.get(block=True, timeout=1)
            if not self.backup_skip:
                self._page_backup(operation)
            self.ack_backup_queue.put(operation)  # 通知完成
        except Empty:
            continue  # 检查stop_event，继续等待
```

| 模式 | CPU占用 | 响应延迟 | 实现 |
|------|---------|----------|------|
| 忙轮询 | 100% | 极低 | `while True: if not q.empty(): ...` |
| **阻塞等待+超时** | **~0%** | **极低（被唤醒）** | `q.get(block=True, timeout=1)` |

> 超时仅为了定期检查退出信号，让线程能优雅终止。

---

## 3. 分批写入与链式哈希

### 3.1 为什么 hash_value 可以遍历？

```python
class TreeNode:
    def __init__(self):
        self.hash_value: List[str] = []  # ← 列表！不是单个字符串
        # 例如：["e5f6...", "g7h8...", "i9j0..."]
```

一个node可能包含多个pages，每个page有一个SHA256 hash：

```python
# 分批遍历
for i in range(0, len(operation.hash_value), self.storage_batch_size):
    batch_hashes = operation.hash_value[i : i + storage_batch_size]
    # i=0: ["hash_a", "hash_b", "hash_c", "hash_d"]
    # i=4: ["hash_e", "hash_f"]
```

### 3.2 链式哈希的动态维护

```python
def _page_backup(self, operation):
    prefix_keys = operation.prefix_keys  # 初始：祖先hash
    
    for batch in operation.hash_value:
        extra_info = HiCacheStorageExtraInfo(prefix_keys=prefix_keys)
        success = self.page_set_func(batch_hashes, batch_host_indices, extra_info)
        
        # 关键：把当前批次追加到依赖链
        if prefix_keys and len(prefix_keys) > 0:
            prefix_keys += batch_hashes  # ← 下一批的前缀包含当前批
```

**动态变化过程**：
```
初始: prefix_keys = ["A", "B"]

第1批写入 ["C1", "C2"]:
  传入 prefix: ["A", "B"]
  成功后: prefix_keys = ["A", "B", "C1", "C2"]

第2批写入 ["C3", "C4"]:
  传入 prefix: ["A", "B", "C1", "C2"]  # 包含了第1批
  成功后: prefix_keys = ["A", "B", "C1", "C2", "C3", "C4"]
  
第3批写入 ["C5"]:
  传入 prefix: ["A", "B", "C1", "C2", "C3", "C4"]
  ...
```

**目的**：告诉存储后端依赖关系，确保前缀恢复时数据不孤立。

---

## 4. 地址翻译与零拷贝

### 4.1 batch_host_indices 是什么？

```python
# batch_host_indices ≠ page 数据
# batch_host_indices = page 的索引坐标

batch_host_indices = operation.host_indices[
    i * self.page_size : (i + len(batch_hashes)) * self.page_size
]
# 例如: tensor([0, 1, 2, ..., 255]) 表示第 0~255 个token位置
```

**长度关系验证**：
```python
assert len(keys) == len(host_indices) // self.page_size
# 1个key → 1个page → page_size个indices
```

### 4.2 get_page_buffer_meta：索引 → 指针

```python
def get_page_buffer_meta(self, indices):
    """
    把抽象的索引坐标翻译成实际的内存指针 (uintptr_t)
    实现零拷贝传输的关键
    """
    kv_buffer_data_ptr = self.kv_buffer.data_ptr()  # 基地址
    ptr_list = []
    
    for index in range(0, len(indices), self.page_size):
        # 计算K cache地址
        k_ptr = kv_buffer_data_ptr + indices[index] * stride
        # 计算V cache地址（MHA时）
        v_ptr = k_ptr + v_offset
        ptr_list.extend([k_ptr, v_ptr])
    
    return ptr_list, element_size_list
```

**内存布局示意**（page_first）：
```
Host KV Buffer:
┌──────────┬──────────┬──────────┬──────────┐
│ page_0   │ page_1   │ page_2   │ ...      │
│ [K][V]   │ [K][V]   │ [K][V]   │          │
│  ↑  ↑       ↑  ↑       ↑  ↑               │
│ ptr0 ptr1  ptr2 ptr3  ptr4 ptr5          │
└──────────┴──────────┴──────────┴──────────┘
    ▲
    └─ indices[0]=0  → 计算得 ptr0, ptr1
    └─ indices[64]=64 → 计算得 ptr2, ptr3
```

---

## 5. MHA vs MLA 存储格式

### 5.1 _get_mha_buffer_meta（传统注意力）

```python
def _get_mha_buffer_meta(self, keys, indices):
    ptr_list, element_size_list = self.mem_pool_host.get_page_buffer_meta(indices)
    key_list = []
    for key_ in keys:
        key_list.append(f"{key_}_{self.mha_suffix}_k")  # K单独存
        key_list.append(f"{key_}_{self.mha_suffix}_v")  # V单独存
    # len(key_list) = 2 * len(keys)
    return key_list, ptr_list, element_size_list
```

### 5.2 _get_mla_buffer_meta（DeepSeek优化）

```python
def _get_mla_buffer_meta(self, keys, indices):
    ptr_list, element_size_list = self.mem_pool_host.get_page_buffer_meta(indices)
    key_list = []
    for key_ in keys:
        key_list.append(f"{key_}_{self.mla_suffix}_k")  # 只有融合的KV
    # len(key_list) = len(keys)
    return key_list, ptr_list, element_size_list
```

| 机制 | 存储内容 | 每page的keys | 代表模型 |
|------|----------|--------------|----------|
| MHA | K和V分开 | 2个 (`xxx_k`, `xxx_v`) | Llama, Qwen |
| MLA | K/V融合 | 1个 (`xxx_k`) | DeepSeek-V2/V3 |

---

## 6. 额外配置项

### 6.1 extra_backend_tag（命名空间前缀）

```python
if self.extra_backend_tag is not None:
    prefix = self.extra_backend_tag
    keys = [f"{prefix}_{key}" for key in keys]
    # 如: "sglang_v1_a1b2c3_0_0_k"
```

**用途**：多租户/多版本隔离
- 不同SGLang实例共享同一个Mooncake Store
- 通过前缀区分： `sglang_v1_` vs `sglang_v2_` vs `experiment_ljh_`

### 6.2 backup_skip（调试开关）

```python
if not self.backup_skip:
    self._page_backup(operation)  # 真正写入
self.ack_backup_queue.put(operation)  # 无论是否跳过，都通知完成
```

- `backup_skip=True`：测试模式，模拟存储不实际写入
- `backup_skip=False`：生产模式，正常写入

---

## 7. 完整调用流程图

```
【主线程】                                    【后台线程 backup_thread】
     │                                               │
     ▼                                               ▼
write_storage(host_indices, token_ids,          backup_thread_func()
              hash_value, prefix_keys)                │
     │                                                ├──► backup_queue.get(timeout=1)
     ├──► StorageOperation(...) ─────────────────────►│
     │                                                ├──► _page_backup(operation)
     ├──► backup_queue.put(op)                             │
     │                                                     ├──► _batch_preprocess()
     └──► return operation.id                              │         ├──► get_page_buffer_meta()
                                                           │         └──► 索引→指针转换
                                                           ├──► page_set_func()
                                                           │         └──► Mooncake batch_set()
                                                           └──► ack_queue.put(op)
```

---

## 8. 关键设计总结

1. **异步解耦**：前台快速入队，后台慢速IO，不阻塞推理
2. **链式哈希**：`prefix_keys`动态扩展，建立page间依赖
3. **零拷贝**：`get_page_buffer_meta`翻译地址，避免内存复制
4. **分批写入**：大node拆batch，降低单次传输压力
5. **格式兼容**：MHA/MLA通过不同函数适配不同存储布局
