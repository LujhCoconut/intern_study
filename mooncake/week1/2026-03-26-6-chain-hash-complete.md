# 链式哈希设计详解 - 从原理到机器执行全流程

> 日期: 2026-03-26  
> 主题：SHA256 链式哈希的原理、目的与实际执行流程

---

## 一句话回答

**SHA256 的单向性 ≠ 不能作为输入再哈希**

就像你把照片放进碎纸机（不可逆），但你可以把碎纸片和其他东西一起再放进碎纸机。

---

## 关键区分

| 概念 | 含义 | 链式哈希的用法 |
|------|------|---------------|
| **单向性** | 无法从哈希值反推出原始内容 | ✅ 我们不需要反推 |
| **确定性** | 相同输入 → 相同输出 | ✅ 用来保证唯一性 |
| **雪崩效应** | 输入微小变化 → 输出完全不同 | ✅ 用来区分不同前缀 |

---

## 链式哈希的真正目的

### 目的 1：建立前缀依赖关系

```
场景：存储以下三个序列
A: "Hello"
B: "Hello world"
C: "Hello world!"

普通哈希（无链式）：
  hash("Hello") = X1
  hash("Hello world") = Y1  ← 和 X1 没关系
  hash("Hello world!") = Z1  ← 和 Y1 没关系

问题：无法从哈希值判断谁是谁的前缀！

链式哈希：
  hash("Hello") = A1
  hash(A1 + " world") = B2   ← 包含了 A1 的信息
  hash(B2 + "!") = C3        ← 包含了 B2（间接包含 A1）的信息

结果：
  - C3 依赖于 B2
  - B2 依赖于 A1
  - 形成链式结构
```

### 目的 2：防止哈希冲突 + 保证前缀唯一性

```
假设没有链式：
  hash("Hello") = A1
  hash("world") = B2

如果有另一个序列 "Hello world"：
  hash("Hello world") = ?
  
  它可能恰好等于 hash("foo bar") = C3
  这就产生了冲突！

链式哈希的优势：
  Page 1 "Hello":     hash("Hello") = A1
  Page 2 " world":    hash(A1 + " world") = B2
  Page 3 "!":         hash(B2 + "!") = C3

  现在 "Hello world!" 的唯一标识是 [A1, B2, C3]
  不可能和其他序列冲突，因为：
  - 要想得到 B2，必须先有 A1
  - 要想得到 C3，必须先有 B2
```

### 目的 3：支持前缀匹配

```
在 Mooncake Store 中查询时：

请求："Hello world! How are you?"

Step 1: 计算 "Hello" 的哈希 = A1
        查询 Mooncake: "A1 存在吗？" → 存在！

Step 2: 用 A1 + " world" 计算哈希 = B2
        查询 Mooncake: "B2 存在吗？" → 存在！

Step 3: 用 B2 + "!" 计算哈希 = C3
        查询 Mooncake: "C3 存在吗？" → 存在！

结果：缓存命中 3 个 page！

如果没有链式哈希：
  - 只能分别查询 "Hello"、"Hello world"、"Hello world!"
  - 无法建立它们之间的关系
  - 也无法增量计算哈希
```

---

## 无情机器视角：我怎么处理请求

> 我是 HiCache，我收到一个请求。这是我的执行日志...

### 场景设定

**我的记忆（L3 Mooncake 里存了什么）：**
```
之前有人请求过："Hello world!"
我在 Mooncake 里存了：
- Key: "a1b2c3d4..." → Page 1: "Hello world! Ho"
- Key: "e5f6g7h8..." → Page 2: "w are you?xxx"
```

**新来的请求：**
```
"Hello world! How are you?"
```

---

### 第一步：我收到请求，分 Page

```
[执行日志]
收到 token_ids: [15496, 11, 616, 329, 13058, 4823, ...]
文本: "Hello world! How are you?"

分 Page (page_size=16):
  Page 1: [15496, 11, 616, 329, 13058, ...] = "Hello world! Ho"
  Page 2: [329, 13058, 4823, ...] = "w are you?xxxxx"
  Page 3: [4823, ...] = "you?"

共 3 个 pages
```

---

### 第二步：我计算链式哈希（关键！）

```
[执行日志 - 哈希计算]

--- 计算 Page 1 哈希 ---
输入: tokens [15496, 11, 616, ...] (16个)
操作: SHA256(tokens)
结果: "a1b2c3d4e5f6..."
记为: hash[0] = "a1b2c3d4..."

--- 计算 Page 2 哈希（链式！）---
输入: 
  - 前一页哈希: "a1b2c3d4..."
  - 当前 tokens: [329, 13058, ...] (16个)
操作: SHA256("a1b2c3d4..." + [329, 13058, ...])
结果: "e5f6g7h8i9j0..."
记为: hash[1] = "e5f6g7h8..."

--- 计算 Page 3 哈希（继续链式！）---
输入:
  - 前一页哈希: "e5f6g7h8..."
  - 当前 tokens: [4823, ...] (16个)
操作: SHA256("e5f6g7h8..." + [4823, ...])
结果: "k1l2m3n4o5p6..."
记为: hash[2] = "k1l2m3n4..."

最终得到 keys: ["a1b2c3d4...", "e5f6g7h8...", "k1l2m3n4..."]
```

**注意重点：**
- hash[1] 包含了 hash[0] 的信息
- hash[2] 包含了 hash[1]（间接包含 hash[0]）的信息
- 如果 Page 1 不同，Page 2 的哈希肯定不同！

---

### 第三步：我查 Mooncake（分 Page 查询！）

```
[执行日志 - 查询 Mooncake]

我发送查询:
  "查 key: a1b2c3d4... 存在吗？"
  "查 key: e5f6g7h8... 存在吗？"
  "查 key: k1l2m3n4... 存在吗？"

Mooncake 返回:
  key a1b2c3d4...: ✅ 存在！
  key e5f6g7h8...: ✅ 存在！
  key k1l2m3n4...: ❌ 不存在

结果分析:
  Page 1 ("Hello world! Ho"): 命中！
  Page 2 ("w are you?xxx"): 命中！
  Page 3 ("you?"): 未命中
```

---

### 第四步：我只 Prefetch 命中的 Pages

```
[执行日志 - Prefetch]

决策: 
  "Page 1 和 Page 2 在 Mooncake 里有，我要取出来！"
  "Page 3 没有，只能重新计算"

执行 prefetch:
  batch_get_v1(
    keys=["a1b2c3d4...", "e5f6g7h8..."],
    host_indices=[1000..1031]  # 我的 L2 内存位置
  )

耗时: ~1ms (RDMA 传输)

现在我的 L2 内存:
  [1000..1015]: Page 1 数据
  [1016..1031]: Page 2 数据
```

---

## 关键问题：为什么不用普通哈希？

```
假设我用普通哈希（不用链式）：

Page 1 哈希: SHA256("Hello world! Ho") = "a1b2c3d4..."
Page 2 哈希: SHA256("w are you?xxx") = "xyz789..."  ← 和 Page 1 没关系！
Page 3 哈希: SHA256("you?") = "abc123..."          ← 和前面都没关系！

问题来了：

场景 A:
用户请求: "Hello world! How are you?"
我生成 keys: ["a1b2c3d4...", "xyz789...", "abc123..."]
查 Mooncake: 都命中！ prefectch 3 pages

场景 B:
用户请求: "world! How are you?" (没有 "Hello")
我生成 keys:
  Page 1: SHA256("world! Ho") = "ddd111..."  ← 不同了！
  Page 2: SHA256("w are you") = "xyz789..."  ← 巧合地和 A 的 Page 2 相同！

这时候:
- 我的 Page 2 key 和 A 的 Page 2 key 相同
- 但内容是不同的！（A 的是 "w are you?xxx"，B 的是 "w are you"）
- 发生哈希冲突！我会读到错误的数据！
```

---

## 链式哈希的真正威力：逐页验证

```
回到链式哈希的场景：

场景 B（没有 "Hello"）:
用户请求: "world! How are you?"

我计算:
  Page 1: SHA256("world! Ho") = "ddd111..."
  Page 2: SHA256("ddd111..." + "w are you") = "fff222..."
  
注意！
- B 的 Page 2 哈希是 "fff222..."
- A 的 Page 2 哈希是 "e5f6g7h8..."
- 它们一定不同！因为输入包含了不同的前一页哈希！

所以我不会错误地读到 A 的数据！
```

---

## 另一个关键场景：前缀匹配

```
场景：
用户 A 之前请求过："Hello world! How are you?"
我在 Mooncake 存了：
  - "a1b2c3d4..." → Page 1
  - "e5f6g7h8..." → Page 2  
  - "k1l2m3n4..." → Page 3

用户 B 现在请求："Hello world! How are you today?"

我计算 B 的哈希：
  Page 1: SHA256("Hello world! Ho") = "a1b2c3d4..."  ← 和 A 相同！
  Page 2: SHA256("a1b2c3d4..." + "w are you") = "e5f6g7h8..."  ← 和 A 相同！
  Page 3: SHA256("e5f6g7h8..." + "?xxxx") = "k1l2m3n4..."  ← 和 A 相同！
  Page 4: SHA256("k1l2m3n4..." + " today") = "p9q8r7s6..."  ← 新的！

查 Mooncake：
  Page 1: ✅ 存在（A 存过）
  Page 2: ✅ 存在（A 存过）
  Page 3: ✅ 存在（A 存过）
  Page 4: ❌ 不存在

结果：
- B 重用 A 的前 3 个 pages！
- 只需要计算第 4 个 page！

这就是"前缀之间的依赖关系"的作用：
- B 的 Page 3 依赖于 B 的 Page 2
- B 的 Page 2 依赖于 B 的 Page 1
- 因为 B 的 Page 1 和 A 的 Page 1 相同
- 所以 B 的 Page 2、3 的 key 也和 A 的相同！
- 天然支持前缀共享！
```

---

## 如果不用链式哈希会怎样？

```
普通哈希场景：

A 存了 "Hello world! How are you?"
  key1 = SHA256("Hello world! Ho")
  key2 = SHA256("w are you?xxx")
  key3 = SHA256("xxxxx")

B 请求 "Hello world! How are you today?"
  我怎么知道 B 的前 3 个 pages 和 A 的一样？
  
  我需要：
  1. 把 B 的 tokens 和 A 的 tokens 逐页对比
  2. Page 1: B 的 "Hello world! Ho" vs A 的 "Hello world! Ho" → 相同！
  3. Page 2: B 的 "w are you" vs A 的 "w are you?xxx" → 等等，可能不一样长！
  4. ...
  
  这太复杂了！我需要存原始 tokens 才能对比！

链式哈希场景：
  B 计算 hash[0] = SHA256("Hello world! Ho") → 和 A 的 hash[0] 相同！
  B 直接查 Mooncake: "a1b2c3d4... 存在吗？" → 存在！
  
  不需要存原始 tokens，只需要比较 64 字符的哈希！
```

---

## 实际代码中的计算

```python
# sglang/srt/mem_cache/hicache_storage.py
import hashlib

def get_hash_str(token_ids: List[int], prior_hash: str = None) -> str:
    hasher = hashlib.sha256()
    
    # 如果有前一个哈希，先放入
    if prior_hash:
        hasher.update(bytes.fromhex(prior_hash))  # 关键！包含前一页哈希
    
    # 放入当前 page 的 tokens
    for t in token_ids:
        hasher.update(t.to_bytes(4, byteorder="little"))
    
    return hasher.hexdigest()

# 使用示例
page1_tokens = [15496]  # "Hello"
hash1 = get_hash_str(page1_tokens)  
# hash1 = "a1b2c3..."

page2_tokens = [11, 616]  # ", how"
hash2 = get_hash_str(page2_tokens, prior_hash=hash1)  
# hash2 = hash(hash1 + page2_tokens) = "d4e5f6..."

page3_tokens = [329, 13058]  # " are you"
hash3 = get_hash_str(page3_tokens, prior_hash=hash2)
# hash3 = hash(hash2 + page3_tokens) = "g7h8i9..."
```

---

## 常见误解澄清

### 误解 1："既然单向，怎么还能用哈希值作为输入？"

```
错误理解：
  "SHA256 是单向的，意味着我不能再用它的输出"

正确理解：
  SHA256 的单向性是指：
    hash → 原始数据  ❌ 不可能
  
  但完全可以：
    hash → 作为新的输入再 hash  ✅ 完全可行
  
  就像：
    你把文件碎成纸屑（不可逆）
    但你可以把纸屑和新的文件一起再碎一次 ✅
```

### 误解 2："这样是不是能反推了？"

```
问题：既然 hash2 = hash(hash1 + data2)，能不能从 hash2 推出 hash1？

答案：不能！

  hash2 = SHA256(hash1 + data2)
  
  要知道 hash1，你需要：
  1. 破解 SHA256（计算上不可能）
  2. 或者知道 data2 并且能分离出来（不可能，因为是混合的）

链式哈希的安全性依然保持。
```

### 误解 3："为什么要这么复杂？"

```
简单方案：直接用 "Hello world" 作为 key

问题：
  1. Key 太长（可能有 thousands of tokens）
  2. 查询时需要精确匹配整个序列
  3. 无法做前缀匹配

链式哈希方案：
  1. 每个 page 固定长度（如 16 tokens）
  2. 用 64 字符的哈希值作为 key
  3. 支持增量查询（先查 page1，再查 page2...）
```

---

## 类比理解

### 类比 1：俄罗斯套娃

```
普通哈希：
  小套娃 A
  中套娃 B  
  大套娃 C
  它们是独立的，看不出关系

链式哈希：
  小套娃 A
  中套娃 B（里面装着小套娃 A）
  大套娃 C（里面装着中套娃 B，B 里装着 A）
  
  想打开 C，必须先打开 B
  想打开 B，必须先打开 A
```

### 类比 2：区块链（简化版）

```
区块链的每个区块包含前一个区块的哈希：

Block 1: hash(数据1) = H1
Block 2: hash(H1 + 数据2) = H2
Block 3: hash(H2 + 数据3) = H3

特性：
- 改了 Block 1 的数据 → H1 变 → H2 变 → H3 变
- 整个链都受影响，很容易检测篡改

HiCache 的链式哈希类似：
- 改了前面的 token → 后面所有哈希都变
- 保证前缀的完整性
```

### 类比 3：接力赛

```
普通哈希：三个独立的跑步者
  跑者 A 跑 100米，记录成绩
  跑者 B 跑 100米，记录成绩  
  跑者 C 跑 100米，记录成绩
  无法知道他们是不是同一队的

链式哈希：接力赛
  跑者 A 跑完，把接力棒给 B
  跑者 B 拿着 A 的棒继续跑，再把棒给 C
  跑者 C 拿着 B 的棒（间接拿着 A 的棒）
  
  拿到 C 的棒 = 知道 A、B、C 是连续的一队
```

---

## 总结：我是怎么利用链式哈希的

```
作为机器，我的执行流程是：

1. 收到请求 → 分 pages
2. 对每个 page，用链式方式计算哈希
   - Page 1: 直接 hash(tokens)
   - Page 2: hash(Page1_hash + tokens)
   - Page 3: hash(Page2_hash + tokens)
   - ...

3. 得到一串 keys: [hash0, hash1, hash2, ...]

4. 逐个查 Mooncake:
   - hash0 存在？→ Prefetch Page 1
   - hash1 存在？→ Prefetch Page 2
   - hash2 存在？→ Prefetch Page 3
   - ...直到某个不存在为止

5. 不存在的 pages，重新计算 KV Cache

这就是"分 page 查询"！

链式哈希保证：
- 如果 Page N 的 key 匹配，Page N-1 一定也匹配（依赖关系）
- 我可以安全地重用缓存的 pages，不用担心数据错乱
- 前缀相同的请求自动共享缓存
```

---

## 核心要点回顾

| 问题 | 答案 |
|------|------|
| SHA256 单向，还能链式吗？ | ✅ 可以，我们只是把哈希值当作普通数据再哈希 |
| 链式的目的是什么？ | 建立前缀依赖、防止冲突、支持增量匹配 |
| 这样安全吗？ | ✅ 安全，无法从后面的哈希反推前面的 |
| 为什么要这样设计？ | 支持前缀缓存、分 page 存储、快速查询 |

**核心思想**：
- SHA256 的单向性：防止从哈希反推内容
- 链式哈希：建立内容之间的依赖关系
- 两者不矛盾，是互补的设计！
