# SGLang HiCache 与 Mooncake 集成完全指南

> 本文档创建时间：2026-03-25  
> 适用版本：SGLang v0.4.x + Mooncake v0.3.x  
> 目标读者：使用 SGLang 进行大模型推理的开发者

---

## 目录

1. [概述](#一概述)
2. [快速开始](#二快速开始)
3. [架构原理](#三架构原理)
4. [配置详解](#四配置详解)
5. [使用方式](#五使用方式)
6. [PD 分离场景](#六pd-分离场景)
7. [性能优化](#七性能优化)
8. [故障排查](#八故障排查)

---

## 一、概述

### 1.1 什么是 HiCache？

**HiCache** 是 SGLang 的分层 KV Cache 管理系统，将存储分为三层：

| 层级 | 存储介质 | 特点 |
|------|----------|------|
| **L1** | GPU HBM | 速度最快，容量最小 |
| **L2** | CPU DRAM | 速度较快，容量中等 |
| **L3** | 分布式存储 | 速度较慢，容量最大 |

### 1.2 Mooncake 作为 L3 存储后端

**Mooncake** 是一个专为 LLM 推理设计的高性能分布式缓存系统，支持 RDMA、多网卡资源，可实现零拷贝、超高速数据传输。

**集成优势：**
- 🚀 **高性能**：利用 RDMA 实现微秒级延迟
- 📦 **大容量**：支持集群级 KV Cache 共享
- 🔄 **零拷贝**：避免数据在 CPU/GPU 之间反复拷贝
- 🌐 **跨节点**：支持 Prefill/Decode 分离部署

### 1.3 典型部署架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                      SGLang + Mooncake 部署架构                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────────────┐         ┌─────────────────────┐          │
│   │   Prefill Node      │         │   Decode Node       │          │
│   │   (计算 Prompt)      │         │   (生成 Token)      │          │
│   │                     │         │                     │          │
│   │  ┌───────────────┐  │         │  ┌───────────────┐  │          │
│   │  │ GPU (L1 Cache)│  │         │  │ GPU (L1 Cache)│  │          │
│   │  │ KV Cache      │  │         │  │ KV Cache      │  │          │
│   │  └───────┬───────┘  │         │  └───────┬───────┘  │          │
│   │          │          │         │          │          │          │
│   │  ┌───────▼───────┐  │         │  ┌───────▼───────┐  │          │
│   │  │ CPU (L2 Cache)│  │         │  │ CPU (L2 Cache)│  │          │
│   │  │ HiCache Pool  │  │         │  │ HiCache Pool  │  │          │
│   │  └───────┬───────┘  │         │  └───────┬───────┘  │          │
│   │          │          │         │          │          │          │
│   └──────────┼──────────┘         └──────────┼──────────┘          │
│              │                               │                     │
│              └───────────────┬───────────────┘                     │
│                              │                                      │
│                   ┌──────────▼──────────┐                          │
│                   │    Mooncake L3      │  ← 集群共享存储层         │
│                   │   (RDMA/TCP)        │                          │
│                   └──────────┬──────────┘                          │
│                              │                                      │
│         ┌────────────────────┼────────────────────┐                 │
│         ▼                    ▼                    ▼                 │
│    ┌─────────┐         ┌─────────┐         ┌─────────┐             │
│    │ Node 1  │         │ Node 2  │         │ Node N  │             │
│    │ Memory  │         │ Memory  │         │ Memory  │             │
│    └─────────┘         └─────────┘         └─────────┘             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、快速开始

### 2.1 环境准备

#### 安装 SGLang（带 HiCache 支持）

```bash
# 安装最新版 SGLang
pip install sglang[all]

# 验证安装
python -c "import sglang; print(sglang.__version__)"
```

#### 安装 Mooncake

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

#### 安装 Router（用于 PD 分离）

```bash
pip install sglang-router
```

### 2.2 启动 Mooncake 服务

```bash
# 启动 Master 服务（启用内置 HTTP Metadata Server）
mooncake_master --enable_http_metadata_server=true

# 或使用环境变量指定配置
export MOONCAKE_MASTER="127.0.0.1:50051"
export MOONCAKE_PROTOCOL="tcp"
export MOONCAKE_TE_META_DATA_SERVER="http://127.0.0.1:8080/metadata"
```

### 2.3 启动 SGLang with Mooncake

```bash
MOONCAKE_MASTER=127.0.0.1:50051 python -m sglang.launch_server \
    --model-path meta-llama/Llama-2-7b-hf \
    --page-size 64 \
    --enable-hierarchical-cache \
    --hicache-storage-prefetch-policy timeout \
    --hicache-storage-backend mooncake
```

### 2.4 测试请求

```bash
curl -X POST http://127.0.0.1:30000/generate \
    -H "Content-Type: application/json" \
    -d '{
        "text": "Tell me a story about",
        "sampling_params": {"temperature": 0.7}
    }'
```

---

## 三、架构原理

### 3.1 HiCache 三层架构详解

```
┌─────────────────────────────────────────────────────────────────────┐
│                         HiCache 架构原理                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                      HiRadixTree                               │ │
│  │                   (元数据索引层)                                │ │
│  │                                                                │ │
│  │   Root ──┬── "The" ──┬── "quick" ──┬── "brown"                │ │
│  │          │   (L1)    │    (L1)     │    (L2)                  │ │
│  │          │           │             │                          │ │
│  │          └── "A" ────┴── "story" ──┴── "about"               │ │
│  │                      (L3)                                       │ │
│  │                                                                │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│           ┌──────────────────┼──────────────────┐                   │
│           ▼                  ▼                  ▼                   │
│      ┌─────────┐       ┌─────────┐       ┌─────────┐               │
│      │  L1 GPU │       │  L2 CPU │       │ L3 Moon │               │
│      │  Cache  │       │  Cache  │       │  cake   │               │
│      │         │       │         │       │         │               │
│      │• 最快   │       │• 较快   │       │• 大容量 │               │
│      │• 热数据 │       │• 温数据 │       │• 冷数据 │               │
│      │• 计算中 │       │• 待使用 │       │• 可共享 │               │
│      └─────────┘       └─────────┘       └─────────┘               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 工作流程

#### 1. Local Match（本地匹配）

```python
# 伪代码示意
def local_match(request_tokens):
    """
    1. 从 HiRadixTree 根节点开始遍历
    2. 匹配请求的 token 前缀
    3. 返回 L1 和 L2 中的匹配结果
    """
    node = radix_tree.root
    matched_tokens = []
    
    for token in request_tokens:
        if token in node.children:
            node = node.children[token]
            matched_tokens.append(token)
        else:
            break
    
    # 返回 L1 和 L2 的匹配长度
    l1_match = min(len(matched_tokens), l1_capacity)
    l2_match = len(matched_tokens) - l1_match
    return l1_match, l2_match
```

#### 2. Prefetch（预取）

```python
def prefetch_from_l3(miss_tokens, policy="timeout"):
    """
    从 L3 (Mooncake) 预取缺失的 KV Cache
    
    policy 选项:
    - best_effort: 立即返回，不等待
    - wait_complete: 等待全部完成
    - timeout: 超时或完成时返回（推荐）
    """
    if len(miss_tokens) < prefetch_threshold:
        return []  # 少于阈值不预取
    
    # 异步预取
    prefetch_tasks = []
    for token in miss_tokens:
        task = async_fetch_from_mooncake(token)
        prefetch_tasks.append(task)
    
    # 根据策略等待
    if policy == "timeout":
        wait_with_timeout(prefetch_tasks, timeout=calculated_timeout)
    elif policy == "wait_complete":
        wait_all(prefetch_tasks)
    
    return completed_tasks
```

#### 3. Write-back（回写）

```python
def write_back_to_l3(l2_evicted_pages, policy="write_through"):
    """
    将 L2 驱逐的数据回写到 L3
    
    policy 选项:
    - write_through: 立即写回
    - write_through_selective: 仅热数据写回
    - write_back: 驱逐时才写回
    """
    for page in l2_evicted_pages:
        if policy == "write_through":
            mooncake_store.put(page.key, page.data)
        elif policy == "write_through_selective" and page.hit_count > threshold:
            mooncake_store.put(page.key, page.data)
```

### 3.3 MooncakeStore 类实现

```python
# python/sglang/srt/mem_cache/storage/mooncake_store/mooncake_store.py

class MooncakeStore(HiCacheStorage):
    """
    Mooncake L3 存储后端实现
    """
    
    def __init__(self, storage_config, mem_pool):
        # 1. 加载配置（环境变量 / 配置文件 / extra_config）
        self.config = self._load_config()
        
        # 2. 初始化 MooncakeDistributedStore
        self.store = MooncakeDistributedStore()
        self.store.setup(
            local_hostname=self.config.local_hostname,
            metadata_server=self.config.metadata_server,
            global_segment_size=self.config.global_segment_size,
            ...
        )
        
        # 3. 注册内存池（零拷贝关键）
        self.register_mem_pool_host(mem_pool)
    
    def register_mem_pool_host(self, mem_pool_host):
        """
        注册 HostKVCache 内存池，实现零拷贝传输
        """
        buffer = mem_pool_host.kv_buffer
        buffer_ptr = buffer.data_ptr()
        buffer_size = buffer.numel() * buffer.element_size()
        
        # 向 Mooncake 注册缓冲区
        self.store.register_buffer(buffer_ptr, buffer_size)
    
    def batch_get_v1(self, keys, host_indices):
        """
        批量从 Mooncake 获取 KV Cache（零拷贝）
        """
        # 1. 预处理：生成 key 和 buffer 指针
        key_strs, buffer_ptrs, buffer_sizes = self._batch_preprocess(keys, host_indices)
        
        # 2. 零拷贝获取
        get_results = self.store.batch_get_into(key_strs, buffer_ptrs, buffer_sizes)
        
        # 3. 后处理：转换结果
        return self._batch_postprocess(get_results)
    
    def batch_set_v1(self, keys, host_indices):
        """
        批量写入 Mooncake（零拷贝）
        """
        # 1. 检查 key 是否已存在（去重）
        exist_result = self._batch_exist(key_strs)
        
        # 2. 仅写入不存在的 key
        new_keys = [k for k, exist in zip(keys, exist_result) if not exist]
        
        # 3. 零拷贝写入
        put_results = self.store.batch_put_from(new_keys, buffer_ptrs, buffer_sizes)
        
        return put_results
```

### 3.4 数据布局

#### 内存布局类型

| 布局类型 | 说明 | 适用场景 |
|----------|------|----------|
| `layer_first` | 按层存储，与 GPU 计算一致 | GPU 计算优化 |
| `page_first` | 按页连续存储，I/O 友好 | 存储传输优化 |
| `page_first_direct` | 页内按层分组 | 平衡计算和 I/O |

```
┌─────────────────────────────────────────────────────────────────────┐
│                     不同内存布局对比                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  layer_first:  [Layer0][Layer1][Layer2]...[LayerN]                 │
│                适合 GPU 逐层计算                                     │
│                                                                     │
│  page_first:   [Page0_all_layers][Page1_all_layers]...             │
│                适合批量 I/O 传输                                     │
│                                                                     │
│  page_first_direct:                                                │
│                [Page0_L0][Page0_L1]...[Page1_L0][Page1_L1]...        │
│                平衡计算和 I/O                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 四、配置详解

### 4.1 环境变量配置

| 环境变量 | 说明 | 示例 |
|----------|------|------|
| `MOONCAKE_MASTER` | Master 服务地址 | `127.0.0.1:50051` |
| `MOONCAKE_PROTOCOL` | 传输协议 | `tcp` / `rdma` |
| `MOONCAKE_DEVICE` | RDMA 设备名 | `mlx5_0,mlx5_1` |
| `MOONCAKE_TE_META_DATA_SERVER` | Metadata 服务 URL | `http://127.0.0.1:8080/metadata` |
| `MOONCAKE_GLOBAL_SEGMENT_SIZE` | 全局内存段大小 | `4294967296` (4GB) |
| `MOONCAKE_LOCAL_HOSTNAME` | 本机主机名 | `192.168.1.100` |

### 4.2 SGLang 启动参数

#### 核心参数

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `--enable-hierarchical-cache` | 启用 HiCache | 必需 |
| `--hicache-ratio` | L2/L1 内存比例 | 2-4 |
| `--hicache-size` | L2 内存大小(GB) | 根据物理内存设置 |
| `--page-size` | 每页 token 数 | 64 |

#### 存储后端参数

| 参数 | 说明 | 选项 |
|------|------|------|
| `--hicache-storage-backend` | L3 存储后端 | `mooncake` |
| `--hicache-storage-prefetch-policy` | 预取策略 | `best_effort` / `wait_complete` / `timeout` |
| `--hicache-write-policy` | 回写策略 | `write_through` / `write_back` / `write_through_selective` |
| `--hicache-mem-layout` | 内存布局 | `layer_first` / `page_first` / `page_first_direct` |
| `--hicache-io-backend` | I/O 后端 | `direct` / `kernel` |

### 4.3 配置文件方式

```json
// /etc/sglang/mooncake_config.json
{
    "local_hostname": "192.168.1.100",
    "metadata_server": "http://192.168.1.1:8080/metadata",
    "global_segment_size": "4gb",
    "protocol": "rdma",
    "device_name": "mlx5_0",
    "master_server_address": "192.168.1.1:50051",
    "check_server": true,
    "standalone_storage": false
}
```

```bash
# 使用配置文件
export SGLANG_HICACHE_MOONCAKE_CONFIG_PATH=/etc/sglang/mooncake_config.json
python -m sglang.launch_server --enable-hierarchical-cache ...
```

### 4.4 Extra Config 方式

```bash
python -m sglang.launch_server \
    --enable-hierarchical-cache \
    --hicache-storage-backend mooncake \
    --hicache-storage-backend-extra-config '{
        "master_server_address": "192.168.1.1:50051",
        "protocol": "rdma",
        "device_name": "mlx5_0",
        "global_segment_size": "4gb",
        "prefetch_threshold": 512,
        "prefetch_timeout_base": 0.5,
        "prefetch_timeout_per_ki_token": 0.25
    }'
```

---

## 五、使用方式

### 5.1 单节点部署

```bash
#!/bin/bash
# start_single_node.sh

# 1. 启动 Mooncake Master
mooncake_master --enable_http_metadata_server=true &
MASTER_PID=$!
sleep 2

# 2. 启动 SGLang
MOONCAKE_MASTER=127.0.0.1:50051 \
python -m sglang.launch_server \
    --model-path meta-llama/Llama-2-7b-hf \
    --tp-size 1 \
    --page-size 64 \
    --enable-hierarchical-cache \
    --hicache-ratio 2 \
    --hicache-storage-backend mooncake \
    --hicache-storage-prefetch-policy timeout \
    --hicache-write-policy write_through \
    --hicache-mem-layout page_first \
    --mem-fraction-static 0.85

echo "Server started. Press Ctrl+C to stop."
wait $MASTER_PID
```

### 5.2 多节点部署（PD 分离）

```bash
#!/bin/bash
# start_pd_disaggregation.sh

# ============ Node 1: Prefill ============
# Terminal 1
MOONCAKE_MASTER=192.168.1.100:50051 \
python -m sglang.launch_server \
    --model-path meta-llama/Llama-2-7b-hf \
    --page-size 64 \
    --enable-hierarchical-cache \
    --hicache-storage-prefetch-policy timeout \
    --hicache-storage-backend mooncake \
    --disaggregation-mode prefill \
    --disaggregation-ib-device "mlx5_1" \
    --base-gpu-id 0 \
    --port 30000

# ============ Node 2: Decode ============
# Terminal 2
python -m sglang.launch_server \
    --model-path meta-llama/Llama-2-7b-hf \
    --page-size 64 \
    --disaggregation-mode decode \
    --disaggregation-ib-device "mlx5_1" \
    --base-gpu-id 0 \
    --port 30001

# ============ Router ============
# Terminal 3
python -m sglang_router.launch_router \
    --pd-disaggregation \
    --prefill "http://192.168.1.100:30000" \
    --decode "http://192.168.1.101:30001" \
    --host 0.0.0.0 \
    --port 8000
```

### 5.3 使用 Python API

```python
import sglang as sgl

# 创建 runtime
runtime = sgl.Runtime(
    model_path="meta-llama/Llama-2-7b-hf",
    enable_hierarchical_cache=True,
    hicache_ratio=2,
    hicache_storage_backend="mooncake",
    hicache_storage_prefetch_policy="timeout",
)

# 发送请求
@sgl.function
def text_qa(s, question):
    s += sgl.system("You are a helpful assistant.")
    s += sgl.user(question)
    s += sgl.assistant(sgl.gen("answer"))

# 第一次请求（冷启动，可能需要从 L3 加载）
result = text_qa.run(
    question="What is the capital of France?",
    temperature=0.7
)
print(result["answer"])

# 第二次请求（相同前缀，命中缓存）
result = text_qa.run(
    question="What is the capital of France? Tell me more.",
    temperature=0.7
)
print(result["answer"])
```

---

## 六、PD 分离场景

### 6.1 PD 分离架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Prefill-Decode 分离架构                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────┐     ┌─────────────────────────┐       │
│  │      Prefill Node       │     │      Decode Node        │       │
│  │   (处理输入/计算 KV)     │     │   (生成输出 Token)       │       │
│  │                         │     │                         │       │
│  │  ┌─────────────────┐    │     │  ┌─────────────────┐    │       │
│  │  │ Compute KV Cache│    │     │  │ Reuse KV Cache  │    │       │
│  │  │ for Prompt      │────┼─────►│  │ for Generation  │    │       │
│  │  └─────────────────┘    │     │  └─────────────────┘    │       │
│  │                         │     │                         │       │
│  │  ┌─────────────────┐    │     │  ┌─────────────────┐    │       │
│  │  │ Store to L3     │    │     │  │ Load from L3    │    │       │
│  │  │ (Mooncake)      │    │     │  │ (Mooncake)      │    │       │
│  │  └─────────────────┘    │     │  └─────────────────┘    │       │
│  └─────────────────────────┘     └─────────────────────────┘       │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      Router (sglang_router)                  │   │
│  │         自动路由：Prefill 请求 → Prefill Node                │   │
│  │                    Decode 请求 → Decode Node                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 PD 分离 + HiCache 配置

```bash
# Prefill 节点配置（启用 HiCache）
MOONCAKE_MASTER=192.168.1.100:50051 \
python -m sglang.launch_server \
    --model-path meta-llama/Llama-2-7b-hf \
    --page-size 64 \
    --enable-hierarchical-cache \
    --hicache-ratio 2 \
    --hicache-storage-backend mooncake \
    --hicache-storage-prefetch-policy timeout \
    --disaggregation-mode prefill \
    --disaggregation-ib-device "mlx5_1" \
    --port 30000

# Decode 节点配置（可选启用 HiCache 回写）
python -m sglang.launch_server \
    --model-path meta-llama/Llama-2-7b-hf \
    --page-size 64 \
    --disaggregation-mode decode \
    --disaggregation-ib-device "mlx5_1" \
    --disaggregation-decode-enable-offload-kvcache \
    --hicache-ratio 2 \
    --hicache-storage-backend mooncake \
    --port 30001
```

### 6.3 Decode 回写优化

启用 `--disaggregation-decode-enable-offload-kvcache` 后：
- Decode 生成的 KV Cache 也会写回 Mooncake L3
- 多轮对话场景下，历史 KV 可被复用
- 显著提升多轮对话性能

```
Round 1:
  User: "Tell me a story" 
  → Prefill 计算 KV → Store to L3 → Decode 生成

Round 2:
  User: "Continue the story"
  → 从 L3 加载历史 KV → 仅需计算新 token 的 KV → Decode 生成
  → 大幅提速！
```

---

## 七、性能优化

### 7.1 预取策略选择

| 策略 | 延迟 | 命中率 | 适用场景 |
|------|------|--------|----------|
| `best_effort` | 最低 | 较低 | 延迟敏感，短序列 |
| `wait_complete` | 最高 | 100% | 吞吐量优先，长序列 |
| `timeout` | 可调 | 可调 | **推荐**，平衡方案 |

### 7.2 内存布局优化

```bash
# 场景 1: 短序列，高并发 → page_first
--hicache-mem-layout page_first

# 场景 2: 长序列，逐层计算 → page_first_direct
--hicache-mem-layout page_first_direct

# 场景 3: 与 GPU 计算完全对齐 → layer_first
--hicache-mem-layout layer_first
```

### 7.3 RDMA 优化

```bash
# 多 RDMA 设备绑定
export MOONCAKE_DEVICE="mlx5_0,mlx5_1,mlx5_2,mlx5_3"
export MOONCAKE_PROTOCOL="rdma"

# 启动 SGLang
python -m sglang.launch_server \
    --enable-hierarchical-cache \
    --hicache-storage-backend mooncake \
    ...
```

### 7.4 性能监控

```python
# 获取 Mooncake 统计信息
from mooncake import store

client = store.MooncakeDistributedStore()
# ... 初始化 ...

# 获取性能指标
stats = client.get_stats()
print(f"Prefetch pages: {stats.prefetch_pgs}")
print(f"Prefetch bandwidth: {stats.prefetch_bandwidth} GB/s")
print(f"Backup bandwidth: {stats.backup_bandwidth} GB/s")
```

---

## 八、故障排查

### 8.1 常见问题

#### Q1: Mooncake 连接失败
```
RuntimeError: Failed to setup Mooncake store, error code: -900 (RPC_FAIL)
```
**解决方案：**
```bash
# 1. 检查 Master 服务
ps aux | grep mooncake_master

# 2. 检查端口连通性
telnet 127.0.0.1 50051

# 3. 检查环境变量
echo $MOONCAKE_MASTER
echo $MOONCAKE_PROTOCOL
```

#### Q2: RDMA 设备找不到
```
Error: IB device mlx5_0 not found
```
**解决方案：**
```bash
# 查看可用 RDMA 设备
ibstat
ibv_devices

# 使用 TCP 回退
export MOONCAKE_PROTOCOL="tcp"
export MOONCAKE_DEVICE=""
```

#### Q3: 内存分配失败
```
Error: NO_AVAILABLE_HANDLE (-200)
```
**解决方案：**
```bash
# 增大全局内存段
export MOONCAKE_GLOBAL_SEGMENT_SIZE="8589934592"  # 8GB

# 或减小 hicache-ratio
--hicache-ratio 1.5
```

### 8.2 调试日志

```bash
# 启用详细日志
export SGLANG_LOG_LEVEL=DEBUG
export MOONCAKE_LOG_LEVEL=DEBUG

# 运行测试
python -m sglang.launch_server ... 2>&1 | tee sglang.log
```

### 8.3 健康检查

```python
# Python 健康检查示例
from mooncake import store

client = store.MooncakeDistributedStore()

# 检查连接状态
status = client.health_check()
if status == 0:
    print("✅ Mooncake 连接正常")
elif status == 1:
    print("⚠️ 未初始化")
elif status == 2:
    print("❌ Master 不可达")

# 检查存储状态
segments = client.get_all_segments()
for seg in segments:
    capacity, used = client.query_segments(seg)
    print(f"Segment {seg}: {used/(1<<30):.2f}GB / {capacity/(1<<30):.2f}GB")
```

---

## 附录

### A. 完整参数速查表

```bash
# ============ Mooncake 环境变量 ============
export MOONCAKE_MASTER="127.0.0.1:50051"
export MOONCAKE_PROTOCOL="tcp"  # 或 "rdma"
export MOONCAKE_DEVICE="mlx5_0"  # RDMA 设备名
export MOONCAKE_TE_META_DATA_SERVER="http://127.0.0.1:8080/metadata"
export MOONCAKE_GLOBAL_SEGMENT_SIZE="4294967296"
export MOONCAKE_LOCAL_HOSTNAME="192.168.1.100"
export MOONCAKE_MASTER_METRICS_PORT="50052"
export MOONCAKE_CHECK_SERVER="true"
export MOONCAKE_STANDALONE_STORAGE="false"
export MOONCAKE_CLIENT=""

# ============ SGLang 参数 ============
python -m sglang.launch_server \
    --enable-hierarchical-cache \
    --hicache-ratio 2 \
    --hicache-size 30 \
    --page-size 64 \
    --hicache-storage-backend mooncake \
    --hicache-storage-prefetch-policy timeout \
    --hicache-write-policy write_through \
    --hicache-mem-layout page_first \
    --hicache-io-backend kernel \
    --hicache-storage-backend-extra-config '{...}'
```

### B. 参考链接

- [Mooncake GitHub](https://github.com/kvcache-ai/Mooncake)
- [SGLang 文档](https://docs.sglang.io/)
- [HiCache 设计文档](https://github.com/sgl-project/sglang/blob/main/docs/advanced_features/hicache_design.md)

---

> **文档结束**  
> 如有问题，请参考官方文档或提交 Issue。
