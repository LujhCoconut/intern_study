# SGLang HiCache 核心模块深度解析

**日期**: 2026-03-25  
**文档版本**: v1  
**分析对象**: 
- `hiradix_cache.py` - 三层 Radix Cache 实现
- `hicache_storage.py` - 存储后端抽象层

---

## 目录

1. [架构定位](#1-架构定位)
2. [hicache_storage.py 详解](#2-hicache_storagepy-详解)
3. [hiradix_cache.py 详解](#3-hiradix_cachepy-详解)
4. [核心操作流程](#4-核心操作流程)
5. [数据流完整示例](#5-数据流完整示例)
6. [关键设计决策](#6-关键设计决策)

---

## 1. 架构定位

### 1.1 在 SGLang 中的位置

```
sglang/srt/mem_cache/
├── hiradix_cache.py      # ← 三层缓存管理（L1/L2/L3）
├── hicache_storage.py    # ← L3 存储后端抽象
├── radix_cache.py        # 基础 Radix Tree（L1 单层）
├── memory_pool_host.py   # L2 Host 内存池
├── memory_pool.py        # L1 GPU 内存池
└── storage/              # L3 具体后端实现
    ├── mooncake_store/   # Mooncake 后端
    ├── nixl/            # NIXL 后端
    └── ...
```

### 1.2 三层缓存架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         HiRadixCache                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  L1: GPU Device Cache (显存)                                    │
│  ├── 存储: MHATokenToKVPool (GPU 内存)                         │
│  ├── 索引: node.value (torch.Tensor - GPU indices)             │
│  └── 速度: ~100-500 GB/s (HBM 带宽)                            │
│                                                                 │
│  L2: Host Memory Cache (CPU 内存)                               │
│  ├── 存储: MHATokenToKVPoolHost (Pinned CPU 内存)              │
│  ├── 索引: node.host_value (torch.Tensor - Host indices)       │
│  └── 速度: ~10-50 GB/s (DDR 带宽)                              │
│                                                                 │
│  L3: External Storage (分布式存储)                              │
│  ├── 存储: MooncakeStore / File / NIXL                         │
│  ├── 索引: node.hash_value (List[str] - 每 page 的 SHA256)    │
│  └── 速度: ~1-10 GB/s (网络带宽)                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. hicache_storage.py 详解

### 2.1 文件职责

这是 **L3 存储后端的抽象层**，定义了所有存储后端必须实现的接口。采用**策略模式 (Strategy Pattern)**，允许 SGLang 在运行时切换不同的存储实现。

### 2.2 核心数据结构

#### 2.2.1 HiCacheStorageConfig - 后端配置

```python
@dataclass
class HiCacheStorageConfig:
    tp_rank: int              # Tensor Parallel rank (0, 1, 2, 3...)
    tp_size: int              # TP 总大小 (如 4)
    pp_rank: int              # Pipeline Parallel rank
    pp_size: int              # PP 总大小
    is_mla_model: bool        # 是否 MLA 模型 (DeepSeek-V3 等)
    is_page_first_layout: bool  # 内存布局: page_first vs layer_first
    model_name: Optional[str]   # 模型名称 (用于生成 key)
    extra_config: Optional[dict] = None  # 后端特定配置
```

**为什么需要这些配置？**

- `tp_rank/tp_size`: 每个 TP rank 存储不同的 KV shard，需要区分 key
- `pp_rank/pp_size`: PP 情况下，不同 stage 存储不同层的 KV
- `is_mla_model`: MLA 和 MHA 的 KV 格式不同，影响存储 key 生成
- `extra_config`: Mooncake 需要 master 地址、协议等配置

#### 2.2.2 HiCacheStorageExtraInfo - 操作额外信息

```python
@dataclass
class HiCacheStorageExtraInfo:
    prefix_keys: Optional[List[str]] = None  # 分层哈希前缀
    extra_info: Optional[dict] = None
```

**分层哈希的作用**: 通过 prefix keys 可以实现更高效的存储查找，例如按模型层或序列前缀组织存储。

### 2.3 抽象基类 HiCacheStorage

```python
class HiCacheStorage(ABC):
    """
    L3 存储后端的抽象接口。
    
    设计原则:
    1. 零拷贝: target_location/target_sizes 允许直接操作内存
    2. 批量操作: batch_* 方法支持批量处理，摊销开销
    3. 异步友好: 方法不阻塞，返回状态可查询
    """
```

#### 2.3.1 核心抽象方法

```python
@abstractmethod
def get(self, key: str, target_location, target_sizes) -> torch.Tensor | None:
    """
    从存储读取单个 key 的数据。
    
    Args:
        key: 存储键（通常是 SHA256 hash）
        target_location: 目标内存地址（通常是 Host KV Cache 的指针）
        target_sizes: 数据大小
    
    Returns:
        成功: 返回 target_location（零拷贝写入）
        失败: 返回 None
    
    零拷贝实现:
        - 传统方式: 存储 → 临时 buffer → memcpy → target_location
        - 零拷贝:   存储 → RDMA/TCP 直接写入 target_location
    """

@abstractmethod
def set(self, key: str, value, target_location, target_sizes) -> bool:
    """
    写入单个 key 到存储。
    
    Args:
        target_location: 源内存地址（Host KV Cache 中的数据）
    
    零拷贝实现:
        - 直接读取 target_location 指向的内存，无需拷贝
    """

@abstractmethod
def exists(self, key: str) -> bool:
    """检查 key 是否存在于存储中（用于缓存命中判断）"""
```

#### 2.3.2 批量操作接口

```python
def batch_exists(self, keys: List[str], extra_info) -> int:
    """
    批量检查存在性。
    
    关键设计: 返回从起始位置**连续**存在的 key 数量。
    
    为什么这样设计？
    - KV Cache 是连续的 token 序列
    - 如果 key[0] 不存在，key[1] 即使存在也无法使用（前缀不匹配）
    - 如果 key[0..5] 存在但 key[6] 不存在，只能使用 key[0..5]
    
    示例:
        keys = ["hash_a", "hash_b", "hash_c", "hash_d"]
        exists = [True, True, True, False]
        返回: 3 (前 3 个连续存在)
    """
    for i in range(len(keys)):
        if not self.exists(keys[i]):
            return i
    return len(keys)
```

### 2.4 HiCacheFile - 文件后端实现

这是最简单的存储后端，直接读写本地文件系统，用于**测试和本地开发**。

#### 2.4.1 初始化

```python
class HiCacheFile(HiCacheStorage):
    def __init__(self, storage_config, file_path="/tmp/hicache"):
        # 为不同 TP rank 创建独立子目录
        # 避免 rank 0 和 rank 1 写入同一个文件
        self.config_suffix = f"_{model_name}_{tp_rank}_{tp_size}"
        
        # 路径示例: /tmp/hicache/Qwen2.5-0.5B_0_4/
```

#### 2.4.2 零拷贝读取实现

```python
def get(self, key, target_location, target_sizes):
    """
    从文件读取 KV Cache 到 target_location（Host Memory）
    """
    key = self._get_suffixed_key(key)  # 添加 "_0_4" 后缀
    tensor_path = os.path.join(self.file_path, f"{key}.bin")
    
    with open(tensor_path, "rb", buffering=0) as f:
        # 核心技术: memoryview + numpy buffer protocol
        # 
        # target_location: PyTorch Tensor (Host pinned memory)
        # .numpy(): 获取 numpy array view（共享内存）
        # memoryview(): 创建可写的 buffer view
        # f.readinto(): 直接读取文件内容到 buffer，零拷贝
        
        buf = memoryview(
            target_location.view(torch.uint8).contiguous().numpy()
        )
        
        bytes_read = f.readinto(buf)
        if bytes_read != expected_size:
            raise IOError(f"Short read for {key}")
    
    return target_location
```

**零拷贝的关键点**:
- `target_location.view(torch.uint8)`: 将任意 dtype tensor 视为字节流
- `.contiguous().numpy()`: 获取连续的 numpy array（与 tensor 共享内存）
- `memoryview()`: 创建 Python buffer protocol 对象，支持 `readinto`
- `f.readinto(buf)`: 系统调用，直接写入用户缓冲区，无内核-用户空间拷贝

#### 2.4.3 零拷贝写入实现

```python
def set(self, key, value, target_location, target_sizes):
    """
    将 target_location 的数据写入文件
    """
    key = self._get_suffixed_key(key)
    tensor_path = os.path.join(self.file_path, f"{key}.bin")
    
    # numpy().tofile() 也是零拷贝
    # 直接将 tensor 的内存内容写入文件，无中间 buffer
    target_location.contiguous().view(dtype=torch.uint8).numpy().tofile(
        tensor_path
    )
```

---

## 3. hiradix_cache.py 详解

### 3.1 文件职责

这是 **HiCache 的核心控制器**，管理 L1/L2/L3 三层缓存的协调工作：
- 扩展基础 RadixCache，添加 L2/L3 支持
- 管理 Host Memory Pool (L2)
- 协调 Storage Backend (L3)
- 处理异步操作（write-through、prefetch、load-back）

### 3.2 核心组件

#### 3.2.1 三层缓存结构

```python
class HiRadixCache(RadixCache):
    def __init__(self, params, server_args):
        # L1: GPU KV Cache（继承自父类）
        self.kv_cache = params.token_to_kv_pool_allocator.get_kvcache()
        
        # L2: Host KV Cache
        self.token_to_kv_pool_host = MHATokenToKVPoolHost(
            self.kv_cache,                    # L1 引用（用于计算大小）
            server_args.hicache_ratio,        # Host/GPU 大小比（默认 2.0）
            server_args.hicache_size,         # 手动指定大小（0=自动）
            self.page_size,
            server_args.hicache_mem_layout,   # "page_first" 或 "layer_first"
            allocator_type=server_args.hicache_storage_backend,
        )
        
        # L3: Storage Backend
        self.enable_storage = server_args.hicache_storage_backend is not None
        
        # 控制器：协调三层操作
        self.cache_controller = HiCacheController(
            mem_pool_device=params.token_to_kv_pool_allocator,
            mem_pool_host=self.token_to_kv_pool_host,
            storage_backend=server_args.hicache_storage_backend,
            write_policy=server_args.hicache_write_policy,  # write_through/write_back
            prefetch_threshold=...,      # 触发预取的最小 token 数
            prefetch_timeout_base=...,   # 预取超时基础值
        )
```

#### 3.2.2 TreeNode 的三层状态扩展

基础 RadixCache 的 TreeNode 只有 `value`（GPU indices）。HiRadixCache 扩展了 L2/L3 状态：

```python
class TreeNode:
    # === L1: GPU Device ===
    value: Optional[torch.Tensor] = None  # GPU memory indices
    evicted: bool = False                 # 是否已从 GPU 驱逐
    
    # === L2: Host Memory ===
    host_value: Optional[torch.Tensor] = None  # Host memory indices
    backuped: bool = False                     # 是否有 Host 备份
    host_ref_counter: int = 0                  # Host 引用计数
    
    # === L3: External Storage ===
    hash_value: List[str] = []  # 每个 page 的 SHA256（作为 storage key）
    
    # === 控制字段 ===
    lock_ref: int = 0           # 锁定计数（防止驱逐）
    hit_count: int = 0          # 访问计数（触发 write-through）
    last_access_time: float     # 最后访问时间（LRU 策略）
```

**状态转换图**:

```
新节点插入
    │
    ▼
┌─────────────┐    write_backup()     ┌─────────────┐
│  L1 Only    │ ────────────────────> │  L1 + L2    │
│ (value)     │                       │ (backuped)  │
└─────────────┘                       └──────┬──────┘
    │                                        │
    │ evict()                                │ write_backup_storage()
    ▼                                        ▼
┌─────────────┐                       ┌─────────────┐
│  L2 Only    │ <──────────────────── │ L1 + L2 + L3│
│ (evicted)   │   load_back()         │ (storage)   │
└─────────────┘                       └─────────────┘
    │
    │ evict_host()
    ▼
┌─────────────┐
│   L3 Only   │  (从 Host 驱逐，但仍在 Storage)
│             │  可通过 prefetch_from_storage() 恢复
└─────────────┘
```

### 3.3 核心操作流程

#### 3.3.1 match_prefix() - 三层缓存查找

```python
def match_prefix(self, params: MatchPrefixParams) -> MatchResult:
    """
    查找 token 序列在三层缓存中的命中情况。
    
    这是推理开始时的第一步：检查哪些 KV 可以复用。
    """
    key = params.key  # token_ids
    
    # Step 1: 基础 Radix Tree 匹配（L1）
    # 沿着 Radix Tree 匹配 token 序列
    value, last_node = self._match_prefix_helper(self.root_node, key)
    # value: L1 中命中的 GPU indices 列表
    # last_node: L1 最后命中的节点
    
    # Step 2: 计算 L2 命中长度
    host_hit_length = 0
    last_host_node = last_node
    
    # 从 L1 未命中位置开始，向上追溯计算 L2 命中
    while last_node.evicted:
        # 节点被驱逐但可能有 Host 备份
        if last_node.backuped:
            host_hit_length += len(last_node.host_value)
        last_node = last_node.parent
    
    # 找到 L2 最后命中的位置（最近的 backuped 祖先）
    while not last_host_node.backuped:
        last_host_node = last_host_node.parent
    
    return MatchResult(
        device_indices=torch.cat(value) if value else empty,  # L1 命中
        last_device_node=last_node,       # L1 最后位置
        last_host_node=last_host_node,    # L2 最后位置
        host_hit_length=host_hit_length,  # L2 可用长度
    )
```

**示例**: 请求 token 序列 `[1, 2, 3, 4, 5, 6, 7]`

```
Radix Tree 状态:
- Node A: key=[1,2], value=GPU[10,11] (在 L1)
- Node B: key=[3,4], value=GPU[12,13], evicted=True, host_value=Host[20,21] (在 L2)
- Node C: key=[5,6,7], evicted=True, backuped=False (不在 L2，可能在 L3)

match_prefix 结果:
- device_indices = GPU[10, 11, 12, 13]  (L1 命中 [1,2,3,4])
- last_device_node = Node A (L1 最后完整节点)
- last_host_node = Node B (L2 最后命中)
- host_hit_length = 2 (L2 中 [3,4] 可用，需要从 Host 加载)

后续操作:
1. 复用 L1: GPU[10,11] 直接用于 attention
2. load_back(): 将 Host[20,21] → GPU[14,15]
3. 计算 [5,6,7] 的新 KV
4. 触发 prefetch_from_storage() 尝试从 L3 获取 [5,6,7]
```

#### 3.3.2 write_backup() - L1 → L2 备份

```python
def write_backup(self, node: TreeNode, write_back=False):
    """
    将节点的 GPU KV Cache 备份到 Host Memory。
    
    触发时机:
    - write_through 策略: hit_count >= threshold 时自动触发
    - write_back 策略: evict() 时触发
    """
    # 1. 在 Host Memory Pool 中分配空间
    host_indices = self.cache_controller.write(
        device_indices=node.value,  # 源: GPU
        node_id=node.id,
    )
    
    if host_indices is None:
        # Host 内存不足，先驱逐其他节点
        self.evict_host(len(node.value))
        host_indices = self.cache_controller.write(...)
    
    # 2. 记录备份信息
    node.host_value = host_indices  # L2 索引
    node.backuped = True            # 标记为已备份
    
    # 3. 跟踪操作（用于异步确认）
    self.ongoing_write_through[node.id] = node
    
    if not write_back:
        self.inc_lock_ref(node)  # 锁定防止被驱逐
    
    # 4. 如果启用 L3，继续异步备份到 Storage
    if self.enable_storage:
        self.write_backup_storage(node)
    
    return len(host_indices)
```

**异步确认机制**:

```python
def writing_check(self):
    """
    定期检查 write-through 操作是否完成。
    在每次 forward 后被调用。
    """
    for _, finish_event, ack_list in self.cache_controller.ack_write_queue:
        if not finish_event.query():
            break  # 按顺序检查，遇到未完成就停止
        
        # 确认完成
        finish_event.synchronize()
        for ack_id in ack_list:
            backuped_node = self.ongoing_write_through.pop(ack_id)
            self.dec_lock_ref(backuped_node)  # 解除锁定
            
            # 继续备份到 L3
            if self.enable_storage:
                self.write_backup_storage(backuped_node)
```

#### 3.3.3 write_backup_storage() - L2 → L3 备份

```python
def write_backup_storage(self, node: TreeNode):
    """
    将 Host 中的 KV 异步备份到 L3 Storage。
    """
    # 1. 生成分层哈希（用于高效查找）
    # prefix_keys 允许按前缀组织存储
    prefix_keys = (
        node.get_prefix_hash_values(node.parent)
        if self.hicache_storage_pass_prefix_keys
        else None
    )
    
    # 2. 提交异步备份操作
    operation_id = self.cache_controller.write_storage(
        host_indices=node.host_value,
        key=node.key,
        hash_value=node.hash_value,  # 每 page 的 SHA256
        prefix_keys=prefix_keys,
    )
    
    # 3. 跟踪操作状态
    self.ongoing_backup[operation_id] = node
    
    # 4. 保护 Host 内存（防止在备份完成前被驱逐）
    node.protect_host()  # host_ref_counter += 1
```

#### 3.3.4 evict() - GPU 内存驱逐

```python
def evict(self, params: EvictParams) -> EvictResult:
    """
    当 GPU 内存不足时，驱逐缓存到 Host。
    
    策略:
    1. 优先驱逐未 backuped 的节点（如果 write_policy=write_back，先备份）
    2. 已 backuped 的节点直接释放 GPU 内存
    """
    # 使用堆按优先级排序（LRU + 自定义策略）
    eviction_heap = [(priority, node) for node in self.evictable_leaves]
    
    while num_evicted < num_tokens:
        _priority, node = heapq.heappop(eviction_heap)
        
        if node.lock_ref > 0:
            continue  # 被锁定，跳过
        
        if not node.backuped:
            # 尚未备份到 Host
            if self.cache_controller.write_policy == "write_back":
                # write_back: 先备份，再驱逐
                num_evicted += self.write_backup(node, write_back=True)
            else:
                # write_through: 应该已经备份过了，直接丢弃
                num_evicted += self._evict_regular(node)
        else:
            # 已备份，直接释放 GPU 内存
            num_evicted += self._evict_backuped(node)

def _evict_backuped(self, node: TreeNode):
    """驱逐已备份节点：释放 GPU，保留 Host"""
    # 释放 GPU indices
    self.cache_controller.evict_device(node.value)
    node.value = None
    node.evicted = True  # 标记为已驱逐
    return len(node.host_value)
```

#### 3.3.5 load_back() - L2 → L1 加载

```python
def load_back(self, node: TreeNode, mem_quota=None):
    """
    当需要被驱逐的 KV 时，从 Host 加载回 GPU。
    
    触发时机: match_prefix 发现部分 token 在 L2 但不在 L1
    """
    # 1. 收集需要加载的节点链
    nodes_to_load = []
    while node.evicted:
        assert node.backuped, "被驱逐节点必须有 Host 备份"
        nodes_to_load.insert(0, node)  # 从祖先到后代
        node = node.parent
    
    # 2. 分配 GPU 内存
    host_indices = torch.cat([n.host_value for n in nodes_to_load])
    device_indices = self.cache_controller.load(
        host_indices=host_indices,
        node_id=target_node.id,
    )
    
    if device_indices is None:
        # GPU 内存不足，先驱逐
        self.evict(EvictParams(num_tokens=len(host_indices)))
        device_indices = self.cache_controller.load(...)
    
    # 3. 更新节点状态
    for node in nodes_to_load:
        node.value = device_indices[offset : offset + len(node.host_value)]
        node.evicted = False
    
    return device_indices
```

#### 3.3.6 prefetch_from_storage() - L3 → L2 预取

```python
def prefetch_from_storage(self, req_id, last_host_node, new_input_tokens, ...):
    """
    当 L1 和 L2 都未命中时，从 L3 Storage 异步预取。
    
    触发时机: match_prefix 后发现 Host 命中长度不足
    """
    # 1. 检查预取条件
    if len(new_input_tokens) < self.prefetch_threshold:
        return  # Token 数太少，不值得预取
    
    if self.cache_controller.prefetch_rate_limited():
        return  # 预取速率受限（避免 I/O 风暴）
    
    # 2. 分配 Host 内存
    last_host_node.protect_host()  # 防止被驱逐
    host_indices = self.cache_controller.mem_pool_host.alloc(prefetch_length)
    
    if host_indices is None:
        # Host 内存不足，先驱逐
        self.evict_host(prefetch_length)
        host_indices = ...
    
    # 3. 提交异步预取操作
    operation = self.cache_controller.prefetch(
        req_id=req_id,
        host_indices=host_indices,
        token_ids=new_input_tokens,
        last_hash=last_hash,
        prefix_keys=prefix_keys,
    )
    
    # 4. 跟踪操作状态
    self.ongoing_prefetch[req_id] = (
        last_host_node,
        new_input_tokens,
        host_indices,
        operation,
    )

def check_prefetch_progress(self, req_id: str) -> bool:
    """
    检查预取进度，完成时将数据插入 Radix Tree。
    
    返回: True 表示可以终止预取，False 表示需要继续等待
    """
    if req_id not in self.ongoing_prefetch:
        return True  # 没有进行中的预取
    
    last_host_node, token_ids, host_indices, operation = \
        self.ongoing_prefetch[req_id]
    
    # 检查是否可以终止（根据 prefetch_stop_policy）
    if not self.can_terminate_prefetch(operation):
        return False
    
    # 终止预取，获取结果
    completed_tokens, hash_value = self.cache_controller.terminate_prefetch(operation)
    
    # 将预取的数据插入 Radix Tree（L2）
    matched_length = self._insert_helper_host(
        last_host_node,
        RadixKey(token_ids=token_ids[:completed_tokens]),
        host_indices[:completed_tokens],
        hash_value,
    )
    
    # 释放未匹配部分的内存
    self.cache_controller.mem_pool_host.free(host_indices[matched_length:completed_tokens])
    
    # 更新统计
    loaded_from_storage = completed_tokens - matched_length
    self.prefetch_loaded_tokens_by_reqid[req_id] = loaded_from_storage
    
    del self.ongoing_prefetch[req_id]
    return True
```

---

## 4. 核心操作流程

### 4.1 推理请求处理流程

```
1. 请求到达
   └── token_ids = [1, 2, 3, 4, 5, 6, 7]

2. match_prefix() - 三层查找
   ├── L1 命中: [1, 2] → GPU indices [10, 11]
   ├── L2 命中: [3, 4] → Host indices [20, 21]
   └── L3 未命中: [5, 6, 7]

3. 加载 L2 → L1
   └── load_back(): Host[20, 21] → GPU[12, 13]

4. 触发 L3 预取
   └── prefetch_from_storage(): 异步获取 [5, 6, 7]

5. 执行 Attention
   ├── 复用 GPU[10, 11, 12, 13]（已有 KV）
   └── 计算 [5, 6, 7] 的新 KV → GPU[14, 15, 16]

6. insert() 插入新节点
   └── Node: key=[5,6,7], value=GPU[14,15,16]

7. 异步后台操作
   ├── write_backup(): GPU[14,15,16] → Host[22,23,24]
   └── write_backup_storage(): Host[22,23,24] → L3 Storage

8. 定期检查（check_hicache_events）
   ├── writing_check(): 确认 write-through 完成
   ├── loading_check(): 确认 load-back 完成
   └── check_prefetch_progress(): 确认预取完成，插入 Tree
```

---

## 5. 数据流完整示例

### 场景：长对话的 KV Cache 复用

```
对话历史（已存储在 L3）:
"你好，请介绍一下机器学习的基本概念..."

新请求:
"你好，请介绍一下机器学习的基本概念，特别是深度学习部分"

处理流程:

1. Tokenize
   └── tokens = [101, 872, 1962, 6413, 681, 9245, ...]

2. match_prefix() 查找
   ├── Radix Tree 匹配前缀 "你好，请介绍一下机器学习的"
   ├── L1 (GPU): 命中 "你好，请介绍" (最近使用)
   ├── L2 (Host): 命中 "一下机器学习的" (已驱逐但备份)
   └── L3 (Storage): 存在 "基本概念" (需要从 Storage 加载)

3. 三层加载
   ├── L1 复用: 直接读取 GPU HBM
   ├── L2 加载: Host → GPU (DDR → HBM，~10GB/s)
   └── L3 预取: Storage → Host (网络，~1GB/s，异步)

4. 计算新部分
   └── "特别是深度学习部分" → 生成新 KV

5. 存储优化
   ├── 新 KV 立即写入 L1 (GPU)
   ├── 访问次数达标后写入 L2 (Host，Write-Through)
   └── 后台异步写入 L3 (Storage)

性能收益:
- 前缀复用率: 80% (仅需计算 20% 新 token)
- 显存节省: 80% (大部分 KV 在 Host/Storage)
- 延迟: 从 100ms → 30ms (3x 加速)
```

---

## 6. 关键设计决策

### 6.1 为什么使用 Radix Tree？

| 特性 | 优势 |
|------|------|
| 前缀匹配 | 自然地支持共享前缀的 KV Cache 复用 |
| 压缩存储 | 共同前缀只存储一次，节省内存 |
| 分层驱逐 | 树结构支持细粒度的缓存管理 |
| 快速查找 | O(m) 复杂度，m 为 token 数，与总缓存大小无关 |

### 6.2 Write-Through vs Write-Back

```
Write-Through (默认):
├── 优点: 数据一致性高，故障恢复快
├── 缺点: 写入延迟高（需等 Host 确认）
└── 适用: 对可靠性要求高的生产环境

Write-Back:
├── 优点: 写入延迟低（仅写 GPU）
├── 缺点: 故障可能丢数据
└── 适用: 对延迟敏感、可容忍重建的场景
```

### 6.3 为什么需要 host_ref_counter？

```python
# 场景：Node 正在被 backup 到 L3，同时被选中驱逐

if node.host_ref_counter > 0:
    # 有正在进行的操作依赖此 Host 内存
    continue  # 跳过驱逐

# 避免数据丢失：
# - 正在 backup 到 L3 时，不能释放 Host 内存
# - 正在 prefetch 到 Host 时，不能覆盖
```

### 6.4 Prefetch 的三种终止策略

```python
self.prefetch_stop_policy = "best_effort" | "wait_complete" | "timeout"

best_effort:
└── 立即终止，返回已获取的数据
    适用: 对延迟敏感，部分数据即可

wait_complete:
└── 等待所有请求的 token 都获取完成
    适用: 对完整性要求高

timeout:
└── 完成或超时后终止
    适用: 平衡延迟和完整性（默认）
```

---

## 总结

| 模块 | 职责 | 关键技术 |
|------|------|----------|
| `hicache_storage.py` | L3 存储抽象 | 零拷贝、批量操作、策略模式 |
| `hiradix_cache.py` | 三层缓存协调 | Radix Tree、引用计数、异步 I/O |

这两个模块共同实现了 **高效、可靠、可扩展** 的分层 KV Cache 系统：
- **高效**: 零拷贝 + 异步 I/O 最小化性能开销
- **可靠**: 引用计数 + 状态机确保数据一致性
- **可扩展**: 插件化后端支持多种存储实现
