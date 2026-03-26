# SGLang + Mooncake 完整流程详解：从 L3 存储读取和预取

本文详细描述当用户发起请求且数据全部不在 L1 (GPU)、L2 (CPU) 命中时，SGLang + Mooncake 如何从 L3 存储读取和预取的完整流程。

---

## 目录

1. [整体流程概览](#整体流程概览)
2. [第 1 阶段：前缀匹配与未命中检测](#第-1-阶段前缀匹配与未命中检测)
3. [第 2 阶段：触发预取](#第-2-阶段触发预取)
4. [第 3 阶段：后台预取线程](#第-3-阶段后台预取线程)
5. [第 4 阶段：等待预取完成](#第-4-阶段等待预取完成)
6. [第 5 阶段：L2 → L1 加载](#第-5-阶段l2--l1-加载)
7. [第 6 阶段：执行推理](#第-6-阶段执行推理)
8. [第 7 阶段：存储新生成的 KV](#第-7-阶段存储新生成的-kv)
9. [时序图](#时序图)
10. [关键设计点总结](#关键设计点总结)

---

## 整体流程概览

```
用户请求："北京有哪些景点？"
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  第 1 阶段：前缀匹配（HiRadixCache.match_prefix）                            │
│  └── L1 未命中 → L2 未命中 → 需要从 L3 加载                                   │
└─────────────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  第 2 阶段：触发预取（prefetch_from_storage）                                │
│  └── 检查阈值 → 分配 L2 内存 → 提交异步预取请求                                │
└─────────────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  第 3 阶段：后台预取线程（prefetch_thread_func）                             │
│  └── 计算哈希 → 查询 L3 存在性 → RDMA 零拷贝传输到 L2                          │
└─────────────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  第 4 阶段：等待预取完成（check_prefetch_progress）                          │
│  └── 轮询进度 → TP 同步 → 插入 RadixTree → 释放未使用内存                       │
└─────────────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  第 5 阶段：L2 → L1 加载（load_back）                                        │
│  └── 追溯祖先节点 → 合并索引 → 分配 GPU 内存 → 逐层异步传输                      │
└─────────────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  第 6 阶段：执行推理（GPU 计算）                                              │
│  └── 读取历史 KV Cache → 计算新 token → 生成回复                               │
└─────────────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  第 7 阶段：存储新生成的 KV（insert + backup）                               │
│  └── 创建新节点 → 存 L1 → 可能触发 L2/L3 备份                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 第 1 阶段：前缀匹配与未命中检测

### 代码位置
`hiradix_cache.py:1126-1161`

### 流程详解

```python
def match_prefix(self, params: MatchPrefixParams):
    """
    匹配请求前缀，检查 L1/L2/L3 各层缓存命中情况
    
    返回 MatchResult：
    - device_indices: GPU (L1) 命中的索引
    - last_device_node: 最后一个 L1 命中的节点
    - last_host_node: 最后一个 L2 命中的节点
    - host_hit_length: 在 L2 中命中的 token 数量
    """
    # 在 RadixTree 中匹配前缀
    value, last_node = self._match_prefix_helper(self.root_node, key)
    
    if value:
        value = torch.cat(value)
    else:
        value = torch.empty((0,), dtype=torch.int64, device=self.device)

    # ===== 检查 L1 (GPU) 命中情况 =====
    # 从最后一个匹配节点向上追溯，统计已驱逐但有备份的节点
    host_hit_length = 0
    last_host_node = last_node
    
    # 如果 node.evicted = True，表示 L1 未命中
    while last_node.evicted:
        host_hit_length += len(last_node.host_value)  # 累加 L2 中的数据
        last_node = last_node.parent  # 向上找有 GPU 数据的祖先
    
    # ===== 检查 L2 (CPU) 命中情况 =====
    # 如果 node.backuped = False，表示 L2 也未命中
    while not last_host_node.backuped:
        last_host_node = last_host_node.parent

    return MatchResult(
        device_indices=value,           # L1 命中部分（可能为空）
        last_device_node=last_node,     # L1 最后节点
        last_host_node=last_host_node,  # L2 最后节点
        host_hit_length=host_hit_length,# L2 命中长度
    )
```

### 未命中判断逻辑

| 条件 | 含义 | 数据位置 |
|------|------|---------|
| `node.evicted = True` | L1 (GPU) 未命中 | 不在 GPU |
| `node.backuped = False` | L2 (CPU) 未命中 | 不在 CPU |
| 需要查询 L3 | 只能从远程存储获取 | Mooncake Store |

---

## 第 2 阶段：触发预取

### 代码位置
`hiradix_cache.py:1163-1201`

### 流程详解

```python
def prefetch_from_storage(
    self,
    req_id: str,
    last_host_node: TreeNode,
    new_input_tokens: List[int],
    last_hash: Optional[str] = None,
    prefix_keys: Optional[List[str]] = None,
):
    """
    从 L3 存储预取 KV Cache 到 CPU (L2) 内存
    """
    # 1. 按页大小对齐预取长度（通常 page_size = 64）
    prefetch_length = len(new_input_tokens) - (
        len(new_input_tokens) % self.page_size
    )
    new_input_tokens = new_input_tokens[:prefetch_length]
    
    # 2. 检查预取条件
    if (
        not self.enable_storage           # L3 未启用
        or prefetch_length < self.prefetch_threshold  # 小于阈值（默认 256 tokens）
        or self.cache_controller.prefetch_rate_limited()  # 速率限制
    ):
        return  # 放弃预取

    # 3. 保护节点，防止预取过程中被驱逐
    last_host_node.protect_host()  # host_ref_counter += 1
    
    # 4. 分配 L2 (CPU) 内存（预分配）
    host_indices = self.cache_controller.mem_pool_host.alloc(prefetch_length)
    if host_indices is None:
        # 内存不足，先驱逐一些
        self.evict_host(prefetch_length)
        host_indices = self.cache_controller.mem_pool_host.alloc(prefetch_length)
    
    if host_indices is None:
        last_host_node.release_host()
        return  # 内存不足，放弃预取
    
    # 5. 提交异步预取操作
    operation = self.cache_controller.prefetch(
        req_id, host_indices, new_input_tokens, last_hash, prefix_keys
    )
    
    # 6. 记录正在进行的预取
    self.ongoing_prefetch[req_id] = (
        last_host_node,
        new_input_tokens,
        host_indices,
        operation,
    )
    self.cache_controller.prefetch_tokens_occupied += len(new_input_tokens)
```

### 关键参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `page_size` | 页大小，预取的基本单位 | 64 tokens |
| `prefetch_threshold` | 触发预取的最小长度 | 256 tokens |
| `host_indices` | L2 内存池预分配的索引 | torch.Tensor |

---

## 第 3 阶段：后台预取线程

### 代码位置
`cache_controller.py:910-959`

### 流程详解

```python
def prefetch_thread_func(self):
    """
    后台线程：从 L3 存储预取数据到 L2 主机内存
    """
    while not self.storage_stop_event.is_set():
        # 1. 从队列获取预取任务
        operation = self.prefetch_queue.get(block=True, timeout=1)
        if operation is None:
            continue
        
        # 2. 查询 L3 存储命中率（关键设计：实时查询！）
        hash_value, storage_hit_count = self._storage_hit_query(operation)
        
        # 3. TP 同步：确保所有 rank 看到相同的命中情况
        if self.tp_world_size > 1:
            storage_hit_count_tensor = torch.tensor(storage_hit_count, dtype=torch.int)
            torch.distributed.all_reduce(
                storage_hit_count_tensor,
                op=torch.distributed.ReduceOp.MIN,
                group=self.prefetch_tp_group,
            )
            storage_hit_count = storage_hit_count_tensor.item()
        
        # 4. 判断是否值得预取
        if storage_hit_count < self.prefetch_threshold:
            # 命中率太低，撤销预取
            self.prefetch_revoke_queue.put(operation.request_id)
            self.append_host_mem_release(operation.host_indices)
            continue
        
        # 5. 更新实际要预取的哈希值列表
        operation.hash_value = hash_value[:storage_hit_count // self.page_size]
        
        # 6. 执行实际数据传输（零拷贝）
        self.page_get_func(operation, batch_hashes, batch_host_indices)
        #    └── MooncakeStore.batch_get_v1()
        #        └── store.batch_get_into()  [RDMA 零拷贝]
```

### L3 实时查询（核心设计）

```python
def _storage_hit_query(self, operation) -> tuple[list[str], int]:
    """
    实时查询 L3 存储，确认哪些 token 存在
    
    特点：不预先保存 L3 位置，实时计算哈希并查询
    """
    last_hash = operation.last_hash
    tokens_to_fetch = operation.token_ids
    
    storage_query_count = 0
    hash_value = []

    # 批量处理（每批 page_size * storage_batch_size 个 token）
    for start in range(0, len(tokens_to_fetch), self.page_size * self.storage_batch_size):
        end = min(start + self.page_size * self.storage_batch_size, len(tokens_to_fetch))
        batch_tokens = tokens_to_fetch[start:end]
        batch_hashes = []
        
        # 为每个 page 计算哈希值（实时计算！）
        for i in range(0, len(batch_tokens), self.page_size):
            last_hash = self.get_hash_str(
                batch_tokens[i : i + self.page_size], last_hash
            )
            batch_hashes.append(last_hash)
        
        # 实时查询 Mooncake 哪些 hash 存在
        extra_info = HiCacheStorageExtraInfo(prefix_keys=prefix_keys)
        hit_page_num = self.storage_backend.batch_exists(batch_hashes, extra_info)
        #    └── Mooncake Master 返回存在性
        
        hash_value.extend(batch_hashes[:hit_page_num])
        storage_query_count += hit_page_num * self.page_size
        
        # 前缀匹配特性：部分未命中就停止
        if hit_page_num < len(batch_hashes):
            break
    
    return hash_value, storage_query_count
```

### RDMA 零拷贝传输

```python
def _page_get_zero_copy(self, operation, hash_values, host_indices, extra_info):
    """
    从 L3 存储直接读取到 L2 内存，零拷贝传输
    """
    results = self.storage_backend.batch_get_v1(
        hash_values, host_indices, extra_info
    )
    #    └── MooncakeStore.batch_get_v1()
    #        └── self.store.batch_get_into(key_strs, buffer_ptrs, buffer_sizes)
    #            └── RDMA 直接写入 host_indices 指向的内存
```

---

## 第 4 阶段：等待预取完成

### 代码位置
`hiradix_cache.py:1046-1107`

### 流程详解

```python
def check_prefetch_progress(self, req_id: str) -> bool:
    """
    检查预取进度，完成后将数据插入 RadixTree
    """
    if req_id not in self.ongoing_prefetch:
        return True  # 已完成或已撤销
    
    last_host_node, token_ids, host_indices, operation = self.ongoing_prefetch[req_id]
    
    # 1. 检查是否可以终止（根据策略）
    if not self.can_terminate_prefetch(operation):
        return False  # 还没完成，继续等待
    
    # 2. 终止预取操作，获取实际完成的 token 数
    completed_tokens, hash_value = self.cache_controller.terminate_prefetch(operation)
    
    # 3. TP 同步（确保所有 rank 完成相同数量）
    if self.tp_world_size > 1:
        completed_tokens_tensor = torch.tensor(completed_tokens, dtype=torch.int)
        torch.distributed.all_reduce(
            completed_tokens_tensor,
            op=torch.distributed.ReduceOp.MIN,
            group=self.tp_group,
        )
        min_completed_tokens = completed_tokens_tensor.item()
    
    # 4. 将成功预取的数据插入 RadixTree（L2）
    fetched_token_ids = token_ids[:min_completed_tokens]
    written_indices = host_indices[:min_completed_tokens]
    
    matched_length = self._insert_helper_host(
        last_host_node,
        RadixKey(token_ids=fetched_token_ids, extra_key=last_host_node.key.extra_key),
        written_indices,
        hash_value[:min_completed_tokens // self.page_size],
    )
    
    # 5. 释放未使用的内存
    self.cache_controller.mem_pool_host.free(host_indices[:matched_length])
    self.cache_controller.append_host_mem_release(
        host_indices[min_completed_tokens:completed_tokens]
    )
    
    # 6. 解除节点保护
    last_host_node.release_host()
    del self.ongoing_prefetch[req_id]
    
    # 7. 统计从 L3 加载的 token 数
    loaded_from_storage = min_completed_tokens - matched_length
    self.prefetch_loaded_tokens_by_reqid[req_id] = loaded_from_storage
    
    return True
```

### 预取终止策略

```python
def can_terminate_prefetch(self, operation: PrefetchOperation):
    """
    根据策略决定是否可以终止预取
    """
    if self.prefetch_stop_policy == "best_effort":
        return True  # 随时可以终止
    
    if self.prefetch_stop_policy == "wait_complete":
        return operation.completed_tokens == len(operation.hash_value) * self.page_size
    
    if self.prefetch_stop_policy == "timeout":
        return completed or self.is_prefetch_timeout(operation)
        # 超时计算：base_time + num_pages * time_per_page
```

---

## 第 5 阶段：L2 → L1 加载

### 代码位置
`hiradix_cache.py:872-928`

### 流程详解

```python
def load_back(self, node: TreeNode, mem_quota: Optional[int] = None):
    """
    将数据从 CPU (L2) 加载回 GPU (L1)
    """
    start_time = time.perf_counter()
    last_hit_node = node
    nodes_to_load = []
    
    # 1. 收集所有需要加载的节点（从当前节点向上追溯）
    while node.evicted:
        assert node.backuped, "No backup available on evicted nodes"
        nodes_to_load.insert(0, node)  # 保持顺序
        node = node.parent
    else:
        ancester_node = node  # 第一个有 GPU 数据的祖先节点

    # 2. 保护祖先节点，防止加载过程中被驱逐
    delta = self.inc_lock_ref(ancester_node)

    # 3. 合并所有需要加载的 host_indices
    host_indices = torch.cat([n.host_value for n in nodes_to_load])
    
    # 4. 检查大小阈值（避免加载太少不划算）
    if len(host_indices) < self.load_back_threshold:  # 默认 10
        self.dec_lock_ref(ancester_node)
        return None

    # 5. 分配 L1 (GPU) 内存
    device_indices = self.cache_controller.load(
        host_indices=host_indices, node_id=last_hit_node.id
    )
    
    if device_indices is None:
        # GPU 内存不足，先执行驱逐
        self.evict(EvictParams(num_tokens=len(host_indices)))
        device_indices = self.cache_controller.load(host_indices=host_indices, node_id=last_hit_node.id)
    
    self.dec_lock_ref(ancester_node)
    if device_indices is None:
        return None  # 加载失败

    # 6. 更新各节点的 L1 索引
    self.ongoing_load_back[last_hit_node.id] = last_hit_node
    offset = 0
    for node in nodes_to_load:
        node.value = device_indices[offset : offset + len(node.host_value)]
        offset += len(node.host_value)
    
    self.evictable_size_ += len(device_indices)
    self.inc_lock_ref(last_hit_node)
    
    return device_indices
```

### 异步逐层传输

```python
# cache_controller.py:711-748
def start_loading(self) -> int:
    """
    启动异步加载（逐层传输）
    """
    producer_id = self.layer_done_counter.update_producer()
    op = CacheOperation.merge_ops(self.load_queue)
    host_indices, device_indices = self.move_indices(op)
    
    producer_event = self.layer_done_counter.events[producer_id]
    producer_event.start_event.record()
    
    with device_module.stream(self.load_stream):
        producer_event.start_event.wait(self.load_stream)
        
        # 逐层异步传输
        for i in range(self.layer_num):
            self.mem_pool_host.load_to_device_per_layer(
                self.mem_pool_device,
                host_indices,
                device_indices,
                i,  # layer_id
                self.io_backend,
            )
            producer_event.complete(i)  # 标记该层完成
    
    self.ack_load_queue.append(HiCacheAck(...))
    return producer_id
```

---

## 第 6 阶段：执行推理

### 流程

```python
# 现在 L1 (GPU) 有了完整的历史 KV Cache
# node.value = torch.tensor([GPU内存索引...])

# 执行模型 forward
with torch.cuda.stream(compute_stream):
    # 1. 从 KV Cache 读取历史信息
    past_kvs = gather_kv_cache(node.value)  # 从 L1 读取
    
    # 2. 计算新的 token
    logits = model.forward(input_ids, past_key_values=past_kvs)
    
    # 3. 采样生成新 token
    new_token = sample(logits)
    
    # 4. 返回结果
    return "故宫、长城、天坛..."
```

---

## 第 7 阶段：存储新生成的 KV

### 流程

```python
# hiradix_cache.py:1296-1375
def insert(self, params: InsertParams) -> InsertResult:
    """
    插入新生成的 KV Cache 到 RadixTree
    """
    # 1. 创建新节点
    new_node = TreeNode(priority=priority)
    new_node.parent = node
    new_node.key = key  # 新 token
    new_node.value = value.clone()  # 存到 L1 (GPU)
    
    # 2. 计算 L3 哈希值（为将来备份准备）
    if self.enable_storage:
        new_node.hash_value = compute_node_hash_values(new_node, self.page_size)
    
    # 3. 根据策略决定是否立即备份到 L2
    if self.cache_controller.write_policy != "write_back":
        self._inc_hit_count(new_node, chunked)
        # 达到一定命中次数后触发 write_backup()
    
    # 4. 如果已备份到 L2，可能触发 L3 备份
    if node.backuped and self.enable_storage:
        self.write_backup_storage(node)
```

### 异步备份到 L3

```python
def write_backup_storage(self, node: TreeNode):
    """
    将节点的 KV Cache 从 L2 备份到 L3 存储
    """
    prefix_keys = node.get_prefix_hash_values(node.parent) if self.hicache_storage_pass_prefix_keys else None
    
    # 提交异步备份任务
    operation_id = self.cache_controller.write_storage(
        node.host_value,    # CPU 内存索引
        node.key,           # Token 序列
        node.hash_value,    # 哈希值列表
        prefix_keys
    )
    
    self.ongoing_backup[operation_id] = node
    node.protect_host()  # 防止备份过程中被驱逐
```

---

## 时序图

```
时间 ─────────────────────────────────────────────────────────────────▶

用户        SGLang       HiRadixCache    HiCacheController    MooncakeStore    Mooncake
 │            │                │                   │                  │           │
 │──请求────▶│                │                   │                  │           │
 │            │─match_prefix──▶│                   │                  │           │
 │            │                │                   │                  │           │
 │            │◀─L1/L2未命中──│                   │                  │           │
 │            │                │                   │                  │           │
 │            │                │─prefetch_from_───▶│                  │           │
 │            │                │  storage          │                  │           │
 │            │                │                   │                  │           │
 │            │                │◀─提交异步任务────│                  │           │
 │            │                │                   │                  │           │
 │            │◀─返回（继续   │                   │                  │           │
 │            │   处理其他）   │                   │                  │           │
 │            │                │                   │                  │           │
 │            │                │                   │─prefetch_thread──▶│          │
 │            │                │                   │  （后台）        │          │
 │            │                │                   │                  │          │
 │            │                │                   │─_storage_hit─────▶│          │
 │            │                │                   │  _query          │          │
 │            │                │                   │                  │          │
 │            │                │                   │◀─返回存在性─────│          │
 │            │                │                   │                  │          │
 │            │                │                   │─batch_get_v1─────▶│          │
 │            │                │                   │                  │          │
 │            │                │                   │                  │─RDMA读────▶│
 │            │                │                   │                  │          │
 │            │                │                   │                  │◀──数据────│
 │            │                │                   │                  │          │
 │            │                │                   │◀─零拷贝到L2─────│          │
 │            │                │                   │                  │          │
 │            │◀─check_prefetch─│                  │                  │          │
 │            │  _progress      │                  │                  │          │
 │            │                │                  │                  │          │
 │            │─确认完成───────▶│                  │                  │          │
 │            │                │                  │                  │          │
 │            │                │─load_back────────▶│                  │          │
 │            │                │  （L2→L1）       │                  │          │
 │            │                │                  │                  │          │
 │            │                │◀─完成───────────│                  │          │
 │            │                │   node.value=GPU索引                │          │
 │            │                │                  │                  │          │
 │            │◀─现在可以推理了─│                  │                  │          │
 │            │                │                  │                  │          │
 │◀─返回结果──│                │                  │                  │          │
 │            │                │                  │                  │          │
```

---

## 关键设计点总结

| 设计点 | 说明 | 优势 |
|--------|------|------|
| **实时查询 L3** | 不预先保存 L3 位置，用时计算 hash 并查询 | 降低元数据开销，灵活适应存储变化 |
| **零拷贝传输** | RDMA 直接将数据写入 L2 内存，不经过 CPU 拷贝 | 高性能，低延迟 |
| **异步预取** | 后台线程处理 L3 读取，不阻塞主推理流程 | 隐藏 I/O 延迟，提高吞吐量 |
| **分层加载** | L3→L2→L1 逐级晋升，按需加载 | 节省 GPU 显存，只加载需要的数据 |
| **页对齐** | 按页（通常 64 tokens）批量处理 | 减少传输次数，提高 I/O 效率 |
| **前缀匹配** | 基于 RadixTree 的共享前缀检测 | 最大化缓存命中率，减少重复存储 |
| **引用计数保护** | `protect_host()` / `release_host()` | 防止数据传输过程中被驱逐 |
| **TP 同步** | 多卡场景下 all_reduce 确保一致性 | 保证分布式环境下数据一致性 |

---

## 流程总结图

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户请求                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 1: 前缀匹配                                              │
│  ├── L1 (GPU) 检查: value is None? → 未命中 ✗                   │
│  ├── L2 (CPU) 检查: backuped? → 未命中 ✗                        │
│  └── 结论: 需要从 L3 加载                                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 2: 触发预取                                              │
│  ├── 检查: 预取长度 > 阈值? → 是                                 │
│  ├── 分配: L2 内存 (host_indices)                               │
│  └── 提交: 异步预取任务 → prefetch_queue                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 3: 后台预取 (prefetch_thread)                            │
│  ├── 计算: token 的 SHA256 hash 列表                            │
│  ├── 查询: storage_backend.batch_exists() → 哪些存在             │
│  ├── 传输: RDMA zero-copy → 直接到 L2 内存                      │
│  └── 完成: 标记 completed_tokens                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 4: 等待完成 (check_prefetch_progress)                    │
│  ├── 轮询: can_terminate_prefetch()?                            │
│  ├── 插入: _insert_helper_host() → RadixTree 更新               │
│  ├── 释放: 未使用内存 → host_mem_release_queue                  │
│  └── 解除: node.release_host()                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 5: L2 → L1 加载 (load_back)                              │
│  ├── 追溯: 收集所有 evicted 但 backuped 的节点                   │
│  ├── 合并: host_indices = cat([n.host_value])                   │
│  ├── 分配: GPU 内存 (device_indices)                            │
│  ├── 传输: 逐层异步拷贝 (H2D)                                   │
│  └── 更新: node.value = device_indices (L1 有数据了)             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 6: 执行推理                                              │
│  └── GPU 计算，生成回复                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 7: 存储新 KV                                             │
│  ├── L1: 新节点 node.value = GPU 索引                           │
│  ├── L2: 可能触发 write_backup()                                │
│  └── L3: 可能触发 write_backup_storage() (异步)                 │
└─────────────────────────────────────────────────────────────────┘
```
