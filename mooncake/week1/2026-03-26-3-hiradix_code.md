# HiRadixCache 代码解析：HiRadixTree 的 KV Cache 数据组织实现

本文档详细分析 `hiradix_cache.py` 代码如何实现文中描述的 HiRadixTree 设计思想。

---

## 一、核心设计思想回顾

文中描述的 HiRadixTree 设计要点：

1. **基于 RadixAttention 的 RadixTree**：每个节点对应一段连续 Token 的 KV Cache，路径代表请求前缀，多请求共享前缀复用节点
2. **多级存储位置记录**：节点记录 KV Cache 可能存放于
   - 本地 GPU 内存 (L1)
   - CPU 内存 (L2)
   - L3 存储后端
   - 多个层级同时存在
3. **本地存储精确元数据**：维护具体存储地址
4. **L3 存储实时查询**：不持续同步 L3 元数据，访问时实时查询存储后端

---

## 二、继承关系与基础结构

### 2.1 类继承关系

```python
# hiradix_cache.py:51
class HiRadixCache(RadixCache):
    """
    HiRadixCache 继承自 RadixCache，扩展了多级缓存管理能力
    """
```

```python
# radix_cache.py:97-158
class TreeNode:
    counter = 0

    def __init__(self, id: Optional[int] = None, priority: int = 0):
        self.children = defaultdict(TreeNode)  # 子节点映射
        self.parent: TreeNode = None           # 父节点引用
        self.key: RadixKey = None              # 节点对应的 Token 序列
        
        # ===== L1: GPU 内存层 =====
        self.value: Optional[torch.Tensor] = None  # GPU 内存索引
        
        self.lock_ref = 0                      # 引用计数（防止驱逐）
        self.last_access_time = time.monotonic()
        self.creation_time = time.monotonic()
        self.hit_count = 0                     # 命中次数统计
        
        # ===== L2: CPU 内存层 =====
        self.host_ref_counter = 0              # 主机内存引用计数
        self.host_value: Optional[torch.Tensor] = None  # CPU 内存索引
        
        # ===== L3: 外部存储层 =====
        self.hash_value: Optional[List[str]] = None  # L3 存储对象的哈希值列表
        
        self.priority = priority               # 驱逐优先级
        self.id = TreeNode.counter if id is None else id
        TreeNode.counter += 1
```

### 2.2 存储位置状态属性

```python
# radix_cache.py:124-130
@property
def evicted(self):
    """判断节点是否已从 GPU 内存驱逐（L1 不存在）"""
    return self.value is None

@property
def backuped(self):
    """判断节点是否有 CPU 内存备份（L2 存在）"""
    return self.host_value is not None
```

**状态组合表：**

| `evicted` | `backuped` | 含义 | 数据位置 |
|-----------|-----------|------|---------|
| False | False | 仅在 GPU | L1 |
| False | True | GPU + CPU | L1 + L2 |
| True | True | 仅在 CPU | L2 |
| True | False | 已完全驱逐 | L3（可能）|

---

## 三、节点存储位置的维护

### 3.1 L1 (GPU) 到 L2 (CPU) 的备份

```python
# hiradix_cache.py:615-636
def write_backup(self, node: TreeNode, write_back=False):
    """
    将节点的 KV Cache 从 GPU (L1) 备份到 CPU (L2)
    
    参数:
        node: 需要备份的 TreeNode
        write_back: 是否为写回策略模式
    
    返回:
        备份的 token 数量
    """
    # 1. 通过 cache_controller 分配主机内存并执行传输
    host_indices = self.cache_controller.write(
        device_indices=node.value,  # GPU 内存索引
        node_id=node.id,
    )
    
    # 2. 如果分配失败，先执行驱逐再重试
    if host_indices is None:
        self.evict_host(len(node.value))
        host_indices = self.cache_controller.write(...)
    
    if host_indices is not None:
        # 3. 记录 CPU 内存索引到 node.host_value
        node.host_value = host_indices
        self.ongoing_write_through[node.id] = node
        
        if not write_back:
            # 4. 增加引用计数，防止在备份过程中被驱逐
            self.inc_lock_ref(node)
    
    return len(host_indices)
```

### 3.2 L2 (CPU) 到 L3 (外部存储) 的备份

```python
# hiradix_cache.py:638-649
def write_backup_storage(self, node: TreeNode):
    """
    将节点的 KV Cache 从 CPU (L2) 备份到 L3 存储后端
    
    注意：此方法不保存 L3 的具体位置元数据，只提交异步备份任务
    """
    # 1. 可选：获取前缀哈希（用于 L3 存储的层级组织）
    prefix_keys = (
        node.get_prefix_hash_values(node.parent)
        if self.hicache_storage_pass_prefix_keys
        else None
    )
    
    # 2. 提交异步备份操作到 storage backend
    operation_id = self.cache_controller.write_storage(
        node.host_value,    # CPU 内存索引
        node.key,           # Token 序列
        node.hash_value,    # 哈希值列表
        prefix_keys
    )
    
    # 3. 记录正在进行的备份操作
    self.ongoing_backup[operation_id] = node
    
    # 4. 保护主机内存，防止备份过程中被驱逐
    node.protect_host()
```

### 3.3 节点分裂时的多层数据分割

```python
# hiradix_cache.py:1269-1294
def _split_node(self, key: RadixKey, child: TreeNode, split_len: int):
    """
    当节点部分匹配时，分裂节点并正确分割各层数据
    
    例如：原节点 key=[1,2,3,4]，新请求匹配 [1,2]，则分裂为：
    - 新节点: key=[1,2]，value=原 value[:2]
    - 原子节点: key=[3,4]，value=原 value[2:]
    """
    new_node = TreeNode(priority=child.priority)
    new_node.children = {self.get_child_key_fn(key[split_len:]): child}
    new_node.parent = child.parent
    new_node.lock_ref = child.lock_ref
    new_node.key = child.key[:split_len]
    new_node.hit_count = child.hit_count

    # ===== 分割 L1 (GPU) 数据 =====
    if child.evicted:
        new_node.value = None
    else:
        new_node.value = child.value[:split_len].clone()
        child.value = child.value[split_len:].clone()
    
    # ===== 分割 L2 (CPU) 数据 =====
    if child.backuped:
        new_node.host_value = child.host_value[:split_len].clone()
        child.host_value = child.host_value[split_len:].clone()

    # ===== 分割 L3 哈希值 =====
    new_node.hash_value, child.hash_value = split_node_hash_value(
        child.hash_value, split_len, self.page_size
    )
    
    child.parent = new_node
    child.key = child.key[split_len:]
    new_node.parent.children[self.get_child_key_fn(key)] = new_node
    return new_node
```

---

## 四、L3 存储的实时查询机制

### 4.1 预取触发条件检查

```python
# hiradix_cache.py:1163-1201
def prefetch_from_storage(
    self,
    req_id: str,
    last_host_node: TreeNode,
    new_input_tokens: List[int],
    last_hash: Optional[str] = None,
    prefix_keys: Optional[List[str]] = None,
):
    """
    从 L3 存储预取 KV Cache 到 CPU 内存
    
    特点：
    1. 不预先保存 L3 元数据
    2. 实时查询存储后端确认数据是否存在
    3. 仅在需要时才发起预取
    """
    # 1. 按页大小对齐预取长度
    prefetch_length = len(new_input_tokens) - (
        len(new_input_tokens) % self.page_size
    )
    new_input_tokens = new_input_tokens[:prefetch_length]
    
    # 2. 检查预取条件
    if (
        not self.enable_storage           # L3 未启用
        or prefetch_length < self.prefetch_threshold  # 小于阈值
        or self.cache_controller.prefetch_rate_limited()  # 速率限制
    ):
        return

    # 3. 保护节点，防止预取过程中被驱逐
    last_host_node.protect_host()
    
    # 4. 分配主机内存
    host_indices = self.cache_controller.mem_pool_host.alloc(prefetch_length)
    if host_indices is None:
        self.evict_host(prefetch_length)
        host_indices = self.cache_controller.mem_pool_host.alloc(prefetch_length)
    
    if host_indices is None:
        last_host_node.release_host()
        return  # 内存不足，放弃预取
    
    # 5. 提交预取操作（实时查询 L3）
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

### 4.2 缓存匹配与 L3 状态检测

```python
# hiradix_cache.py:1126-1161
def match_prefix(self, params: MatchPrefixParams):
    """
    匹配请求前缀，返回各层缓存命中情况
    
    返回 MatchResult 包含：
    - device_indices: GPU (L1) 命中的索引
    - last_device_node: 最后一个 L1 命中的节点
    - last_host_node: 最后一个 L2 命中的节点（含已驱逐但有备份的）
    - host_hit_length: 在 L2 中命中的 token 数量
    """
    key = params.key
    value, last_node = self._match_prefix_helper(self.root_node, key)
    
    if value:
        value = torch.cat(value)
    else:
        value = torch.empty((0,), dtype=torch.int64, device=self.device)

    # ===== 统计 L2 (CPU) 命中情况 =====
    host_hit_length = 0
    last_host_node = last_node
    
    # 从最后一个匹配节点向上追溯，统计已驱逐但有备份的节点
    while last_node.evicted:
        host_hit_length += len(last_node.host_value)
        last_node = last_node.parent
    
    # 找到最后一个有备份的节点
    while not last_host_node.backuped:
        last_host_node = last_host_node.parent

    return MatchResult(
        device_indices=value,       # L1 命中
        last_device_node=last_node, # L1 最后节点
        last_host_node=last_host_node,  # L2 最后节点
        host_hit_length=host_hit_length,  # L2 命中长度
    )
```

### 4.3 L3 实时查询实现

```python
# cache_controller.py:878-908
def _storage_hit_query(self, operation) -> tuple[list[str], int]:
    """
    实时查询 L3 存储，确认哪些 token 存在
    
    特点：
    1. 实时计算 token 的哈希值
    2. 调用 storage_backend.batch_exists 实时查询
    3. 不依赖预先保存的 L3 元数据
    
    返回:
        hash_value: 命中的哈希值列表
        storage_hit_count: 命中的 token 数量
    """
    last_hash = operation.last_hash
    tokens_to_fetch = operation.token_ids
    prefix_keys = operation.prefix_keys.copy() if operation.prefix_keys else None

    storage_query_count = 0
    hash_value = []

    # 批量查询，每批处理 page_size * storage_batch_size 个 token
    for start in range(
        0, len(tokens_to_fetch), self.page_size * self.storage_batch_size
    ):
        end = min(
            start + self.page_size * self.storage_batch_size, len(tokens_to_fetch)
        )
        batch_tokens = tokens_to_fetch[start:end]
        batch_hashes = []
        
        # 为每个 page 计算哈希值
        for i in range(0, len(batch_tokens), self.page_size):
            last_hash = self.get_hash_str(
                batch_tokens[i : i + self.page_size], last_hash
            )
            batch_hashes.append(last_hash)
        
        # ===== 实时查询 L3 存储 =====
        extra_info = HiCacheStorageExtraInfo(prefix_keys=prefix_keys)
        hit_page_num = self.storage_backend.batch_exists(batch_hashes, extra_info)
        
        hash_value.extend(batch_hashes[:hit_page_num])
        storage_query_count += hit_page_num * self.page_size
        
        # 如果部分未命中，停止查询（前缀匹配特性）
        if hit_page_num < len(batch_hashes):
            break
        
        if prefix_keys and len(prefix_keys) > 0:
            prefix_keys += batch_hashes

    return hash_value, storage_query_count
```

---

## 五、多级存储的插入流程

### 5.1 插入时计算 L3 哈希值

```python
# hiradix_cache.py:1359-1375
def insert(self, params: InsertParams) -> InsertResult:
    """
    插入新的 KV Cache 到 RadixTree
    
    特点：
    1. 如果存储启用，计算节点的哈希值用于后续 L3 备份
    2. 根据 write_policy 决定是否立即备份到 L2
    """
    # ... 前缀匹配和节点创建 ...
    
    if len(key):
        new_node = TreeNode(priority=priority)
        new_node.parent = node
        new_node.key = key
        new_node.value = value.clone()  # L1 存储
        node.children[child_key] = new_node
        self.evictable_size_ += len(value)
        
        # ===== 计算 L3 哈希值 =====
        if self.enable_storage:
            new_node.hash_value = compute_node_hash_values(
                new_node, self.page_size
            )
        
        # 根据策略决定是否立即备份到 L2
        if self.cache_controller.write_policy != "write_back":
            self._inc_hit_count(new_node, chunked)
    
    return InsertResult(prefix_len=total_prefix_length)
```

### 5.2 哈希值计算

```python
# radix_cache.py (compute_node_hash_values)
def compute_node_hash_values(node: TreeNode, page_size: int) -> List[str]:
    """
    计算节点每个 page 的哈希值，用于 L3 存储标识
    
    从根节点到当前节点的路径哈希链
    """
    hash_values = []
    last_hash = None
    
    # 获取完整的 token 序列
    tokens = []
    current = node
    while current.parent is not None:
        tokens = list(current.key) + tokens
        current = current.parent
    
    # 按 page_size 分段计算哈希
    for i in range(0, len(tokens), page_size):
        page_tokens = tokens[i:i + page_size]
        last_hash = get_hash_str(page_tokens, last_hash)
        hash_values.append(last_hash)
    
    return hash_values
```

---

## 六、数据加载回 GPU 的流程

### 6.1 从 L2 加载回 L1

```python
# hiradix_cache.py:872-928
def load_back(self, node: TreeNode, mem_quota: Optional[int] = None):
    """
    将数据从 CPU (L2) 加载回 GPU (L1)
    
    用于：节点被驱逐（evicted=True）但仍有备份（backuped=True）时
    """
    start_time = time.perf_counter()
    last_hit_node = node
    nodes_to_load = []
    
    # 收集所有需要加载的节点（从当前节点向上追溯）
    while node.evicted:
        assert node.backuped, "No backup available on evicted nodes"
        nodes_to_load.insert(0, node)
        node = node.parent
    else:
        ancester_node = node  # 第一个有 GPU 数据的祖先节点

    # 保护祖先节点，防止加载过程中被驱逐
    delta = self.inc_lock_ref(ancester_node)

    # 合并所有需要加载的 host_indices
    host_indices = torch.cat([n.host_value for n in nodes_to_load])
    
    # 检查大小限制
    if len(host_indices) < self.load_back_threshold:
        self.dec_lock_ref(ancester_node)
        return None

    # 分配 GPU 内存并发起加载
    device_indices = self.cache_controller.load(
        host_indices=host_indices, node_id=last_hit_node.id
    )
    
    if device_indices is None:
        # GPU 内存不足，先驱逐
        self.evict(EvictParams(num_tokens=len(host_indices)))
        device_indices = self.cache_controller.load(...)
    
    self.dec_lock_ref(ancester_node)
    if device_indices is None:
        return None  # 加载失败

    # 更新各节点的 L1 索引
    self.ongoing_load_back[last_hit_node.id] = last_hit_node
    offset = 0
    for node in nodes_to_load:
        node.value = device_indices[offset : offset + len(node.host_value)]
        offset += len(node.host_value)
    
    self.evictable_size_ += len(device_indices)
    self.inc_lock_ref(last_hit_node)
    
    return device_indices
```

---

## 七、驱逐策略与状态管理

### 7.1 三级驱逐流程

```python
# hiradix_cache.py:774-818
def evict(self, params: EvictParams) -> EvictResult:
    """
    三级驱逐策略：
    1. 优先驱逐无备份的 GPU 数据
    2. write_back 模式下：先备份到 CPU，再驱逐 GPU
    3. 已备份的节点：直接释放 GPU 引用
    """
    eviction_heap = [...]
    
    while num_evicted < num_tokens and len(eviction_heap):
        _priority, x = heapq.heap
        pop(eviction_heap)

        if x.lock_ref > 0:
            continue  # 被锁定，不能驱逐

        if not x.backuped:
            if self.cache_controller.write_policy == "write_back":
                # 先备份到 CPU，再驱逐 GPU
                num_evicted += self.write_backup(x, write_back=True)
                write_back_nodes.append(x)
            else:
                # 直接驱逐（删除节点）
                num_evicted += self._evict_regular(x)
        else:
            # 已有备份，直接释放 GPU 数据
            num_evicted += self._evict_backuped(x)

    # write_back 模式下确认备份完成
    if self.cache_controller.write_policy == "write_back":
        self.writing_check(write_back=True)
        for node in write_back_nodes:
            self._evict_backuped(node)  # 现在可以安全驱逐 GPU
```

### 7.2 二级驱逐（CPU 内存）

```python
# hiradix_cache.py:839-870
def evict_host(self, num_tokens: int):
    """
    从 CPU (L2) 驱逐到 L3 存储或完全删除
    
    注意：只有在节点已从 GPU 驱逐后，才会考虑 CPU 驱逐
    """
    leaves = list(self.evictable_host_leaves)  # 可驱逐的叶节点集合
    eviction_heap = [...]

    while num_evicted < num_tokens and len(eviction_heap):
        _priority, x = heapq.heappop(eviction_heap)
        
        # 只驱逐已完全从 GPU 驱逐的节点
        if not x.evicted:
            continue

        # 检查是否被存储操作保护
        if x.host_ref_counter > 0:
            continue

        # 执行驱逐
        num_evicted += self.cache_controller.evict_host(x.host_value)
        
        # 从树中移除节点（如果是叶节点）
        key = self.get_child_key_fn(x.key)
        x.parent.children.pop(key, None)
```

---

## 八、关键状态追踪机制

### 8.1 进行中的操作追踪

```python
# hiradix_cache.py:147-156
def __init__(self, ...):
    # ...
    
    # 记录正在写入 CPU 的节点（L1 -> L2）
    self.ongoing_write_through = {}
    
    # 记录正在加载回 GPU 的节点（L2 -> L1）
    self.ongoing_load_back = {}
    
    # 记录正在从 L3 预取的请求
    self.ongoing_prefetch = {}
    
    # 记录正在备份到 L3 的操作
    self.ongoing_backup = {}
    
    # 追踪从 L3 加载的 token 数量（用于统计）
    self.prefetch_loaded_tokens_by_reqid: dict[str, int] = {}
```

### 8.2 节点保护机制

```python
# radix_cache.py:132-141
def protect_host(self):
    """
    保护节点主机内存，防止在存储操作（预取/备份）过程中被驱逐
    
    通过增加 host_ref_counter 实现
    """
    self.host_ref_counter += 1

def release_host(self):
    """释放主机内存保护"""
    if self.host_ref_counter > 0:
        self.host_ref_counter -= 1
    else:
        raise RuntimeError("Host reference counter is already zero.")
```

---

## 九、总结：代码如何体现设计思想

| 设计要点 | 代码实现 | 关键数据结构/方法 |
|---------|---------|-----------------|
| **RadixTree 基础结构** | 继承 `RadixCache`，使用 `TreeNode` | `TreeNode.children`, `TreeNode.parent`, `TreeNode.key` |
| **L1 GPU 存储** | `node.value` | `torch.Tensor` 类型的 GPU 内存索引 |
| **L2 CPU 存储** | `node.host_value` | `torch.Tensor` 类型的 CPU 内存索引 |
| **L3 外部存储** | `node.hash_value` | `List[str]` 类型的哈希值列表 |
| **存储位置判断** | `evicted`, `backuped` 属性 | `@property` 装饰器简化状态判断 |
| **L3 实时查询** | `prefetch_from_storage` | 调用 `storage_backend.batch_exists` |
| **精确本地元数据** | `host_value`, `value` | 具体的内存索引 Tensor |
| **不持续同步 L3** | 仅在需要时查询 | `_storage_hit_query` 实时计算哈希并查询 |
| **节点分裂** | `_split_node` | 同时分割 `value`, `host_value`, `hash_value` |

**设计亮点：**

1. **轻量级 L3 元数据**：只保存哈希值列表，不保存具体位置，大幅降低内存开销
2. **延迟加载**：L3 数据仅在需要时才查询和加载，避免无效的元数据同步
3. **状态分离**：通过 `evicted` 和 `backuped` 两个布尔属性，清晰表示 4 种存储状态
4. **引用计数保护**：`host_ref_counter` 和 `lock_ref` 保护数据在传输过程中不被破坏
