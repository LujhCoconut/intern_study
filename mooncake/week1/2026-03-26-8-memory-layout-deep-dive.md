# KV Cache 内存布局与寻址公式深度解析

> 从 PyTorch Tensor 底层到零拷贝传输的完整地址计算原理

## 目录
1. [基础概念：多维数组的线性存储](#1-基础概念多维数组的线性存储)
2. [PyTorch Tensor 的内存布局](#2-pytorch-tensor-的内存布局)
3. [KV Cache 的数据结构](#3-kv-cache-的数据结构)
4. [两种内存布局详解](#4-两种内存布局详解)
5. [寻址公式完整推导](#5-寻址公式完整推导)
6. [为什么需要不同的 Layout](#6-为什么需要不同的-layout)
7. [零拷贝传输的适配](#7-零拷贝传输的适配)

---

## 1. 基础概念：多维数组的线性存储

### 1.1 计算机内存是一维的

```
物理内存视图（一维线性地址空间）:
地址:    0x1000   0x1004   0x1008   0x100C   0x1010   ...
         ├────────┼────────┼────────┼────────┼────────┤
数据:    [  0   ] [  1   ] [  2   ] [  3   ] [  4   ]
         └────────┴────────┴────────┴────────┴────────┘
```

### 1.2 多维数组必须"展平"存储

```python
# 逻辑视图（2D矩阵）: 3行 × 4列
┌────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │  ← row 0
├────┼────┼────┼────┤
│ 4  │ 5  │ 6  │ 7  │  ← row 1
├────┼────┼────┼────┤
│ 8  │ 9  │ 10 │ 11 │  ← row 2
└────┴────┴────┴────┘

# 物理存储（行优先 Row-Major）:
地址偏移:  0     4     8     12    16    20    24    ...
          [0]   [1]   [2]   [3]   [4]   [5]   [6]   ...

# 访问 element[row][col] 的公式:
# address = base + (row * num_cols + col) * element_size
```

**关键洞察**：多维索引 → 线性地址 必须有一个**映射函数**。

---

## 2. PyTorch Tensor 的内存布局

### 2.1 Tensor 的元信息

```python
import torch

# 创建一个 tensor
x = torch.randn(3, 4, 5)  # shape = [3, 4, 5]

# 关键属性
print(x.shape)      # torch.Size([3, 4, 5]) - 逻辑维度
print(x.stride())   # (20, 5, 1)              - 步长
print(x.storage_offset())  # 0                 - 起始偏移
print(x.data_ptr()) # 0x7f8a1000              - 物理地址
```

### 2.2 Stride（步长）的概念

```
Tensor: shape = [D0, D1, D2], stride = [S0, S1, S2]

访问元素 x[i][j][k] 的地址:
address = base_ptr + i*S0 + j*S1 + k*S2

示例: shape=[3, 4, 5], stride=[20, 5, 1]
x[1][2][3] 的偏移 = 1*20 + 2*5 + 3*1 = 20 + 10 + 3 = 33
```

**Stride 的物理意义**：
- `S0 = 20`: 沿第0维移动1步，跳过20个元素（= 4×5）
- `S1 = 5`: 沿第1维移动1步，跳过5个元素（= 5）
- `S2 = 1`: 沿第2维移动1步，跳过1个元素

### 2.3  contiguous（连续）vs non-contiguous

```python
# 连续存储（Row-Major）
x = torch.arange(12).reshape(3, 4)
# [[ 0,  1,  2,  3],
#  [ 4,  5,  6,  7],
#  [ 8,  9, 10, 11]]
# stride = (4, 1) - 标准行优先

# 转置后变得不连续
y = x.t()
# [[ 0,  4,  8],
#  [ 1,  5,  9],
#  [ 2,  6, 10],
#  [ 3,  7, 11]]
# stride = (1, 4) - 列优先！物理上仍是行优先存储

print(y.is_contiguous())  # False
```

**重要**：零拷贝传输要求内存**连续**，否则需要额外的 `contiguous()` 复制。

---

## 3. KV Cache 的数据结构

### 3.1 Transformer 的 KV Cache 维度

```python
# 单次前向传播的 KV shape:
# K: [batch_size, num_heads, seq_len, head_dim]
# V: [batch_size, num_heads, seq_len, head_dim]

# 推理时逐步扩展，使用 PagedAttention 优化后:
# 物理存储: [num_pages, page_size, num_layers, 2(K/V), num_heads, head_dim]
# 但不同的框架选择不同的物理布局
```

### 3.2 SGLang 的 KV Cache 参数

```python
self.layer_num      # L - Transformer 层数（如 32, 64）
self.size           # S - 总 token 容量（如 32768）
self.page_size      # P - 每页 token 数（如 64）
self.head_num       # H - 注意力头数（如 8, 16）
self.head_dim       # D - 每个头的维度（如 128）
self.dtype.itemsize # E - 元素字节数（float32=4, float16=2）

总页数 = S // P
每页大小 = L * P * H * D * E 字节
```

### 3.3 需要存储什么

```
对于每个 token 位置，需要存储:
├── Layer 0
│   ├── K: [num_heads, head_dim]  = H×D 个元素
│   └── V: [num_heads, head_dim]  = H×D 个元素
├── Layer 1
│   ├── K: [num_heads, head_dim]
│   └── V: [num_heads, head_dim]
... 共 layer_num 层

每个 token 的数据量 = layer_num × 2 × H × D × E 字节
```

---

## 4. 两种内存布局详解

### 4.1 为什么需要不同的 Layout

不同操作有不同的**访问模式**：

| 操作 | 访问模式 | 最优 Layout |
|------|----------|-------------|
| 计算 Attention | 按 layer 顺序读 K,V | layer_first |
| 存储/传输 Page | 一次性读写整个 page | page_first |
| CPU ↔ GPU 拷贝 | 大块连续内存最快 | 视硬件而定 |

### 4.2 Layout 1: `layer_first`

```
物理存储顺序: [Layer][Page][Token][Head][Dim]

内存布局:
┌──────────────────────────────────────────────────────────────┐
│ Layer 0                                                      │
│ ├─ Page 0 ─┬─ Page 1 ─┬─ Page 2 ─┬─ ...                       │
│ │ [K][V]   │ [K][V]   │ [K][V]   │                            │
│ └──────────┴──────────┴──────────┘                            │
├──────────────────────────────────────────────────────────────┤
│ Layer 1                                                      │
│ ├─ Page 0 ─┬─ Page 1 ─┬─ ...                                  │
│ │ [K][V]   │ [K][V]   │                                       │
│ └──────────┴──────────┘                                       │
├──────────────────────────────────────────────────────────────┤
│ ... 其他 Layer                                               │
└──────────────────────────────────────────────────────────────┘

Stride 设计:
- 同 Layer 内相邻 Page: stride = page_size × H × D × E
- 不同 Layer 之间: stride = total_tokens × H × D × E
```

### 4.3 Layout 2: `page_first`

```
物理存储顺序: [Page][Layer][Token][Head][Dim]

内存布局:
┌──────────────────────────────────────────────────────────────┐
│ Page 0                                                       │
│ ├─ Layer 0 ─┬─ Layer 1 ─┬─ Layer 2 ─┬─ ...                   │
│ │  [K][V]   │  [K][V]   │  [K][V]   │                        │
│ └───────────┴───────────┴───────────┘                        │
├──────────────────────────────────────────────────────────────┤
│ Page 1                                                       │
│ ├─ Layer 0 ─┬─ Layer 1 ─┬─ ...                               │
│ │  [K][V]   │  [K][V]   │                                    │
│ └───────────┴───────────┘                                    │
├──────────────────────────────────────────────────────────────┤
│ ... 其他 Page                                                │
└──────────────────────────────────────────────────────────────┘

Stride 设计:
- 同 Page 内相邻 Layer: stride = page_size × H × D × E
- 不同 Page 之间: stride = layer_num × page_size × H × D × E
```

---

## 5. 寻址公式完整推导

### 5.1 基础参数定义

```python
# 常量定义（简化表示）
E = self.dtype.itemsize        # 元素大小（4或2字节）
H = self.head_num              # 头数
D = self.head_dim              # 头维度
P = self.page_size             # 每页token数
L = self.layer_num             # 层数
S = self.size                  # 总token容量

# 派生常量
HEAD_BYTES = H * D * E                    # 一个head的数据量
TOKEN_BYTES = L * 2 * H * D * E           # 一个token的K+V数据量
LAYER_BYTES = S * H * D * E               # 一层所有token的K数据量
PAGE_BYTES = L * P * H * D * E            # 一页的K数据量（V另算）
```

### 5.2 `layer_first` 寻址推导

**目标**：找到 `Page p, Layer l, Token t`（页内第t个token）的 K 和 V 地址。

**Step 1: Layer 的基地址偏移**
```
每个 Layer 包含所有 Pages 的所有 Tokens:
Layer 大小 = S tokens × H heads × D dims × E bytes = S × H × D × E

Layer l 的基地址 = base_ptr + l × (S × H × D × E)
                 = base_ptr + l × LAYER_BYTES
```

**Step 2: Page 在 Layer 内的偏移**
```
每个 Page 包含 P tokens:
Page 在 Layer 内的大小 = P × H × D × E

Page p 在 Layer l 内的偏移 = p × (P × H × D × E)
                           = p × PAGE_SIZE_BYTES
```

**Step 3: Token 在 Page 内的偏移**
```
Token t 在 Page 内的偏移 = t × H × D × E
                         = t × HEAD_BYTES
```

**Step 4: 综合 K 的地址公式**
```
k_ptr = base_ptr 
      + l × (S × H × D × E)      ← Layer 偏移
      + p × (P × H × D × E)      ← Page 偏移  
      + t × (H × D × E)          ← Token 偏移
```

**代码实现**（对应源码）：
```python
k_ptr = (
    kv_buffer_data_ptr                                    # base
    + indices[index] * self.head_num * self.head_dim * self.dtype.itemsize  # token偏移
    + layer_id * self.size * self.head_num * self.head_dim * self.dtype.itemsize  # layer偏移
)
# 注意：indices[index] 已经是全局token索引 = p × P + t
```

**Step 5: V 的地址**
```
V 紧跟在所有 Layer 的 K 之后:
v_offset = L × S × H × D × E = L × LAYER_BYTES

v_ptr = k_ptr + v_offset
```

### 5.3 `page_first` 寻址推导

**目标**：同样找到 `Page p, Layer l, Token t` 的地址。

**Step 1: Page 的基地址偏移**
```
每个 Page 包含所有 Layers:
Page 大小 = L layers × P tokens × H heads × D dims × E bytes

Page p 的基地址 = base_ptr + p × (L × P × H × D × E)
                = base_ptr + p × PAGE_TOTAL_BYTES
```

**Step 2: Layer 在 Page 内的偏移**
```
每个 Layer 在 Page 内占:
Layer 在 Page 内大小 = P × H × D × E

Layer l 在 Page p 内的偏移 = l × (P × H × D × E)
```

**Step 3: Token 偏移（同上）**
```
Token t 偏移 = t × H × D × E
```

**Step 4: 综合 K 的地址公式**
```
k_ptr = base_ptr
      + p × (L × P × H × D × E)   ← Page 偏移
      + l × (P × H × D × E)       ← Layer 偏移（在Page内）
      + t × (H × D × E)           ← Token 偏移
```

**提取公因式简化**：
```
k_ptr = base_ptr
      + indices[index] × (L × H × D × E)   # indices[index] = p × P + t
```

**代码实现**（对应源码）：
```python
k_ptr = (
    kv_buffer_data_ptr
    + indices[index]                              # token索引
    * self.layer_num                              # 乘上 layer_num
    * self.head_num * self.head_dim * self.dtype.itemsize  # 每层数据量
)
```

**为什么可以这么简化？**

```
indices[index] = p × P + t

indices[index] × L × H × D × E
= (p × P + t) × L × H × D × E
= p × P × L × H × D × E + t × L × H × D × E
= p × (L × P × H × D × E)        ← Page 偏移
+ t × L × H × D × E              ← Token 偏移，但需要调整...

等等，这和前面的推导不一致！token 的偏移应该是 t × H × D × E 才对。

问题出在哪里？

仔细看代码，在 page_first 中，layer 循环被去掉了！
意味着：k_ptr 指向的是 Page 中所有 Layers 的 K 的起始位置。
实际上 indices[index] 是 page 的起始位置，然后一次性取出所有 layer。
```

**重新理解 `page_first`**：

```python
# 在 page_first 中，指针粒度是整个 Page 的所有 Layers
# 即：一次 get/put 操作传输一个完整的 Page

for index in range(0, len(indices), self.page_size):
    # indices[index] 是 Page 的起始 token 索引
    # 计算该 Page 的基地址
    k_ptr = base + indices[index] * stride_per_token
    
# 其中 stride_per_token = layer_num * head_num * head_dim * itemsize
# 这是因为在 page_first 布局中，一个 token 的所有 layer 数据是连续的
```

### 5.4 两种布局的地址计算对比表

| 布局 | 存储顺序 | K 地址公式 | 适用场景 |
|------|----------|-----------|----------|
| `layer_first` | `[Layer][Page][Token]` | `base + l×S×H×D×E + idx×H×D×E` | 逐层计算 |
| `page_first` | `[Page][Layer][Token]` | `base + idx×L×H×D×E` | 整页传输 |

---

## 6. 为什么需要不同的 Layout

### 6.1 `layer_first` 的优势

```
计算 Attention 时的访问模式:
for layer in layers:           ← 外层循环 layer
    load K_cache[layer]         ← 连续读取该 layer 的所有 pages
    load V_cache[layer]
    compute attention
    
优势：同一 Layer 的数据在内存中连续，CPU/GPU缓存友好
```

### 6.2 `page_first` 的优势

```
存储/传输时的访问模式:
for page in pages:             ← 外层循环 page
    write page to disk/rdma     ← 一次性写一个完整的 page

优势：
1. 一次系统调用传输完整数据
2. RDMA 可以高效处理大块连续内存
3. 与存储的"页"概念对齐
```

### 6.3 SGLang 的选择

```python
# 根据 use case 选择
if layout == "layer_first":
    # 主要用于：GPU kernel 计算 attention
    # 因为 kernel 内层循环是 layer
    ...
elif layout in ["page_first", "page_first_direct"]:
    # 主要用于：与 Mooncake Store 等后端交互
    # 因为需要整页读写存储
    ...
```

---

## 7. 零拷贝传输的适配

### 7.1 为什么需要 `get_page_buffer_meta`

```
Mooncake Store 的接口:
put_batch(keys: List[str], 
          ptrs: List[uintptr_t],   ← 需要实际内存地址
          sizes: List[size_t])     ← 每个地址的数据大小

但 HiCache 管理的是抽象索引：[0, 1, 2, ..., 63] 表示第0-63个token

需要：索引 → 地址 的转换
```

### 7.2 转换过程

```python
# 输入: indices = [0, 1, 2, ..., 63, 64, ..., 127]  (2 pages, page_size=64)
# 输出: ptr_list = [ptr_page0, ptr_page1]

# Step 1: 找到每页的基地址
page0_base = base_ptr + indices[0] * stride   # indices[0] = 0
page1_base = base_ptr + indices[64] * stride  # indices[64] = 64

# Step 2: 计算每页的数据大小
page_size_bytes = layer_num * page_size * head_num * head_dim * itemsize

# Step 3: 返回给 Mooncake
ptr_list = [page0_base, page1_base]
size_list = [page_size_bytes, page_size_bytes]
```

### 7.3 MHA vs MLA 的差异

```
MHA (Multi-Head Attention):
├─ K cache: [layer_num, num_pages, page_size, num_heads, head_dim]
└─ V cache: [layer_num, num_pages, page_size, num_heads, head_dim]
            ↓ 分开存储
ptr_list = [page0_K_ptr, page0_V_ptr, page1_K_ptr, page1_V_ptr, ...]

MLA (Multi-head Latent Attention - DeepSeek):
├─ KV cache: [layer_num, num_pages, page_size, kv_cache_dim]  # 融合存储
            ↓ 只有一个
ptr_list = [page0_ptr, page1_ptr, ...]
```

---

## 8. 总结

### 8.1 核心公式速查

```python
# 通用常量
HEAD_STRIDE = head_num * head_dim * itemsize
TOKEN_STRIDE_MHA = layer_num * 2 * head_num * head_dim * itemsize  # K+V
TOKEN_STRIDE_MLA = layer_num * kv_cache_dim * itemsize              # 融合

# layer_first: 先定位 Layer，再定位 Token
k_ptr = base + layer_id * (size * HEAD_STRIDE) + token_idx * HEAD_STRIDE

# page_first: 直接定位 Page（包含所有 Layers）
page_ptr = base + page_start_idx * (layer_num * HEAD_STRIDE)
```

### 8.2 设计精髓

1. **抽象与实现的分离**：上层使用逻辑索引，底层翻译成物理地址
2. **Layout 的灵活性**：根据访问模式选择最优存储顺序
3. **零拷贝的核心**：一次性获取所有指针，避免传输时的内存复制
4. **硬件友好**：大块连续内存最适合 RDMA 和 DMA 传输

### 8.3 一句话

> **寻址公式是"逻辑索引空间"到"物理内存空间"的映射函数。不同的 Layout 对应不同的映射策略，选择哪种取决于主要的访问模式（计算优先 vs 传输优先）。**
