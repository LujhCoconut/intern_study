# Mooncake 作为 L3 KV 缓存

本文档描述如何使用 Mooncake 作为 SGLang 的 L3 KV 缓存。

相关文档：
* [快速入门：Mooncake 后端的 SGLang HiCache](https://kvcache-ai.github.io/Mooncake/getting_started/examples/sglang-integration/hicache-quick-start.html)
* [完整指南：Mooncake 后端的 SGLang HiCache](https://kvcache-ai.github.io/Mooncake/getting_started/examples/sglang-integration/hicache-integration-v1.html)
* [Mooncake x SGLang HiCache 系统设计](https://kvcache-ai.github.io/Mooncake/design/hicache-design.html)
* [HiCache 系统设计与优化](https://docs.sglang.io/advanced_features/hicache_design.html)
* [Mooncake 后端的 SGLang HiCache 基准测试](https://kvcache-ai.github.io/Mooncake/performance/sglang-hicache-benchmark-results-v1.html)

## 关于 Mooncake

Mooncake 旨在通过在高速互联的 DRAM/SSD 资源上构建多级缓存池，提升大语言模型（LLM）的推理效率，特别是在慢速对象存储环境中。与传统缓存系统相比，Mooncake 利用（GPUDirect）RDMA 技术以零拷贝方式直接传输数据，同时最大化利用单机多网卡资源。

有关 Mooncake 的更多详情，请参考 [Mooncake 项目](https://github.com/kvcache-ai/Mooncake) 和 [Mooncake 文档](https://kvcache-ai.github.io/Mooncake/)。

### Mooncake 与 SGLang HiCache

Mooncake 作为 SGLang HiCache 的高性能 L3 存储后端，通过 RDMA 加速数据传输，实现跨多台服务器的分布式 KV 缓存存储。这种集成通过分布式内存池提供几乎无限的缓存存储，解决了传统仅 GPU 或 GPU+CPU 缓存的容量限制问题。

当 L1 和 L2 发生缓存未命中时，HiCache 会自动从 Mooncake 的分布式内存池中获取所需的 KV 缓存。系统使用智能预取策略来最小化延迟，并利用 RDMA 技术和零拷贝技术确保 SGLang 实例与 Mooncake 存储节点之间的高带宽、低延迟数据传输。

**主要优势：**

- **可扩展容量**：将整个集群的内存聚合到大型分布式池中。
- **缓存共享**：集群中的所有 SGLang 实例可以共享 KV 缓存。
- **RDMA 加速**：直接内存访问消除 CPU 开销并降低延迟。
- **零拷贝**：在 L2 和 Mooncake 之间直接传输数据，无需中间拷贝，最大化吞吐量。
- **容错性**：分布式架构提供对单个节点故障的弹性。

这种集成对于涉及长上下文模型、多轮对话和高吞吐量服务场景的生产部署特别有价值，因为在这些场景中传统缓存方法会遇到容量限制。

## 安装 Mooncake

**方法 1：使用 pip**

```bash
pip install mooncake-transfer-engine
```

**方法 2：从源码安装**

克隆 Mooncake 项目：

```bash
git clone https://github.com/kvcache-ai/Mooncake --recursive
```

安装依赖：

```bash
cd Mooncake
bash dependencies.sh
```

构建项目：

```bash
mkdir build
cd build
cmake ..
make -j
```

安装 Mooncake：

```bash
sudo make install
```

更多详情，请参考 [Mooncake 官方安装指南](https://kvcache-ai.github.io/Mooncake/getting_started/build.html)。

## 部署

**Mooncake** 是一个分布式系统，可高效聚合多台服务器的内存资源。它也可以在单台服务器上部署，用于更简单的设置。

与 **SGLang** 集成时，系统在概念上由四个关键组件组成：`主服务（master service）`、`元数据服务（metadata service）`（可选）、`存储服务（store service）`（可选）和 `SGLang 服务器`。其中，`主服务`和`元数据服务`负责对象和元数据维护。`存储服务`管理为分布式 KV 缓存贡献的连续内存段，使其内存可被本地和远程的 `SGLang 服务器`访问。数据传输直接发生在 `存储服务` 和 `SGLang 服务器` 之间，绕过 `主服务`。

### 单服务器部署

**启动 Mooncake `元数据服务`（可选）：**

```bash
python -m mooncake.http_metadata_server
```

该服务负责集中式元数据管理，包括内部连接状态和相关元数据。

在以下情况下可以跳过 `元数据服务` 的部署：
* Mooncake 支持通过 P2P 握手机制进行非集中式元数据管理来交换元数据。使用此模式时，可以跳过 `元数据服务` 的部署。
* Mooncake 还支持将 `元数据服务` 嵌入到 `主服务` 中。在这种情况下，只需启动 `主服务`。

**启动 Mooncake `主服务`：**

`主服务` 编排整个集群的逻辑存储空间池，管理 KV 缓存空间分配和驱逐。

启动 `mooncake_master`：

```bash
mooncake_master --eviction_high_watermark_ratio=0.95
```

启动带有嵌入式 `元数据服务` 的 `mooncake_master`（这样可以跳过单独的 `元数据服务` 部署）：

```bash
mooncake_master --enable_http_metadata_server=true --http_metadata_server_port=8080 --eviction_high_watermark_ratio=0.95
```

**理解 `eviction_high_watermark_ratio`：**

当 `PutStart` 请求因内存不足而失败，或当驱逐线程检测到空间使用率达到配置的高水位比例时，会触发驱逐任务以释放空间。

由于内存碎片，即使内存使用率尚未达到 100%，也可能发生分配失败。实际阈值取决于工作负载。此[基准测试文档](https://kvcache-ai.github.io/Mooncake/performance/allocator-benchmark-result.html)提供了不同场景下的内存分配效率结果。如果观察到过多的分配失败，请考虑相应降低此参数。

**启动 Mooncake `存储服务`（可选）：**

首先，创建并保存一个 JSON 格式的配置文件。例如：

```json
{
    "local_hostname": "localhost",
    "metadata_server": "http://127.0.0.1:8080/metadata",
    "master_server_address": "127.0.0.1:50051",
    "protocol": "rdma",
    "device_name": "",
    "global_segment_size": "4gb",
    "local_buffer_size": 0
}
```

注意：如果未部署 `元数据服务`，请将此字段设置为：

```json
    "metadata_server": "P2PHANDSHAKE",
```

然后启动 `存储服务`：

```bash
python -m mooncake.mooncake_store_service --config=[config_path] --port=8081
```

Mooncake `存储服务` 配置也可以通过环境变量提供：

```bash
MOONCAKE_LOCAL_HOSTNAME="localhost" \
MOONCAKE_TE_META_DATA_SERVER="http://127.0.0.1:8080/metadata" \
MOONCAKE_MASTER="127.0.0.1:50051" \
MOONCAKE_PROTOCOL="rdma" \
MOONCAKE_DEVICE="" \
MOONCAKE_GLOBAL_SEGMENT_SIZE="4gb" \
MOONCAKE_LOCAL_BUFFER_SIZE=0 \
python -m mooncake.mooncake_store_service --port=8081
```

**参数说明：**

* `local_hostname`、`MOONCAKE_LOCAL_HOSTNAME`：`存储服务` 的主机名。
* `metadata_server`、`MOONCAKE_TE_META_DATA_SERVER`：`元数据服务` 的网络地址。默认端口是 8080。如果未部署 `元数据服务`，请将此字段设置为：`"metadata_server": "P2PHANDSHAKE"`。
* `master_server_address`、`MOONCAKE_MASTER`：`主服务` 的网络地址。默认端口是 50051。
* `protocol`、`MOONCAKE_PROTOCOL`：Mooncake 使用的协议。支持的值为 `"rdma"` 或 `"tcp"`。为获得最佳性能，建议使用 `"rdma"`。
* `device_name`、`MOONCAKE_DEVICE`：Mooncake 使用的 RDMA 设备。此字段通常可以留空，因为 Mooncake 默认会自动发现可用的网卡。仅当协议设置为 `"rdma"` **且**需要使用特定网卡组时才需要此参数。示例：`"device_name": "mlx5_0,mlx5_1"`。要列出可用设备，请运行 `ibv_devices`。**注意：** 如果环境变量 `MC_MS_AUTO_DISC` 设置为 `1`，任何 `device_name` 或 `MOONCAKE_DEVICE` 配置都将被覆盖，Mooncake 将切换到自动发现模式。
  - 对于张量并行部署，不同 rank 应使用不同设备，您可以使用 JSON 格式指定设备配置：
    ```json
    {
    "device_name": "{0: \"ib0,ib1\", 1: \"ib2,ib3\", 2: \"ib4,ib5\"}"
    }
    ```
  - 或在环境变量中：
    ```bash
    MOONCAKE_DEVICE="{\"0\": \"ib0,ib1\", \"1\": \"ib2,ib3\", \"2\": \"ib4,ib5\"}"
    ```
* `global_segment_size`、`MOONCAKE_GLOBAL_SEGMENT_SIZE`：贡献给全局内存池的内存量。接受字节（整数）或带有 `gb` 后缀的字符串，例如 `"4294967296"` 或 `"4gb"`。较大的值允许 Mooncake 缓存更多的 KV 张量。
* `local_buffer_size`、`MOONCAKE_LOCAL_BUFFER_SIZE`：本地缓冲区用于执行请求操作，如 `Get` 或 `Put`。在这种情况下，它设置为 0，因为该实例仅作为存储服务器，向全局池贡献内存而不发出任何请求操作。

**重要：理解全局段大小**

`global_segment_size` 和 `MOONCAKE_GLOBAL_SEGMENT_SIZE`：此参数指定每个实例贡献给分布式内存池的内存量。集群中可用于 KV 缓存存储的总内存是所有实例贡献内存的总和。

根据系统的可用内存和预期的缓存需求调整此值。

注意：如果在启动 `SGLang 服务器` 时将 `MOONCAKE_GLOBAL_SEGMENT_SIZE` 设置为非零值，可以跳过 `存储服务` 的启动。在这种情况下，`SGLang 服务器` 也承担 `存储服务` 的角色，这简化了部署但将两个组件耦合在一起。用户可以选择最适合其需求的部署方式。

**启动启用 Mooncake 的 `SGLang 服务器`：**

有三种方式配置 Mooncake：

1. 通过 sglang 参数传递的额外配置
2. 使用 JSON 配置文件
3. 使用环境变量

Mooncake 按以下优先级顺序加载配置：

1. 如果在 `--hicache-storage-backend-extra-config` 中提供了 Mooncake 特定选项，则优先使用。
2. 如果没有，Mooncake 检查是否设置了环境变量 `DEFAULT_MOONCAKE_CONFIG_PATH_ENV`，并从该路径加载 JSON 配置文件。
3. 如果上述两者都未提供，Mooncake 将回退到环境变量。

**使用 sglang 参数的 extra-config 配置 Mooncake**

```bash
python -m sglang.launch_server \
    --enable-hierarchical-cache \
    --hicache-storage-backend mooncake \
    --model-path [model_path] \
    --hicache-storage-backend-extra-config '{"master_server_address": "127.0.0.1:50051", "local_hostname": "localhost", "metadata_server": "http://127.0.0.1:8080/metadata", "global_segment_size": "4gb", "protocol": "rdma", "device_name": ""}'
```

**使用 JSON 文件配置 Mooncake**

SGLang 服务器可以从 `SGLANG_HICACHE_MOONCAKE_CONFIG_PATH` 加载 Mooncake 配置。

```bash
export SGLANG_HICACHE_MOONCAKE_CONFIG_PATH=/sgl-workspace/sglang/benchmark/hicache/mooncake_config.json

echo '{
    "local_hostname": "localhost",
    "metadata_server": "http://127.0.0.1:8080/metadata",
    "master_server_address": "127.0.0.1:50051",
    "protocol": "rdma",
    "device_name": "",
    "global_segment_size": "4gb"
}' > ${SGLANG_HICACHE_MOONCAKE_CONFIG_PATH}

python -m sglang.launch_server \
    --enable-hierarchical-cache \
    --hicache-storage-backend mooncake \
    --model-path [model_path]
```

**使用环境变量配置 Mooncake**

```bash
MOONCAKE_TE_META_DATA_SERVER="http://127.0.0.1:8080/metadata" \
MOONCAKE_MASTER="127.0.0.1:50051" \
MOONCAKE_PROTOCOL="rdma" \
MOONCAKE_DEVICE="" \
MOONCAKE_GLOBAL_SEGMENT_SIZE="4gb" \
python -m sglang.launch_server \
    --enable-hierarchical-cache \
    --hicache-storage-backend mooncake\
    --model-path [model_path]
```

**参数说明：**

此处使用的 Mooncake 参数与为 `存储服务` 配置的参数基本相同。

特别是对于 `全局段大小`，如果至少有一个 `存储服务` 实例正在运行，此值可以设置为 `0`。在这种情况下，SGLang 服务器不会向系统贡献任何内存。注意，存储在此贡献内存中的 KV 张量将在进程退出时丢失；但是，这**不会**导致任何系统错误。

**重要：** 当 `tp > 1` 时，每个张量并行（TP）rank 都会启动自己的 Mooncake 后端实例并贡献 `1/global_segment_size` 内存。因此，总内存消耗等于 `全局段大小`。

**SGLang 服务器的 HiCache 相关参数**

有关 HiCache 相关参数的完整概述，请参考[此文档](https://docs.sglang.io/advanced_features/hicache_design.html#related-parameters)。

注意，对于 `--hicache-mem-layout {layer_first,page_first,page_first_direct}`，它指定主机内存池的内存布局，如果使用 Mooncake 后端，则需要使用 `page_first` 或 `page_first_direct`。

### 分布式部署

Mooncake 的分布式部署很简单。类似于单节点设置，为集群启动一个 `元数据服务` 和一个 `主服务`。然后在每台服务器上启动一个 `存储服务`。

Mooncake 还支持高可用模式。此模式通过将 `主服务` 作为由 `etcd` 集群协调的多个主节点集群运行来增强容错性。主节点使用 `etcd` 选举领导者，负责处理客户端请求。有关如何在此模式下部署的更多详情，请参考我们的[文档](https://kvcache-ai.github.io/Mooncake/)。

### 使用虚拟客户端部署（实验性）

除了 SGLang 作为完整 Mooncake 节点的标准部署外，您还可以使用 **虚拟客户端** 模式。在此模式下，SGLang 通过 RPC/IPC 连接到本地的 **Mooncake 存储服务**（真实客户端）。这将 SGLang 进程与繁重的 RDMA 和内存管理解耦，可能提高稳定性并允许缓存即使在 SGLang 进程重启时也能持久化。

**架构：**
* **Mooncake 主服务**：管理集群拓扑（与标准方式相同）。
* **Mooncake 存储服务（真实客户端）**：管理实际的内存池和 RDMA 连接。必须在本地运行。
* **SGLang 服务器（虚拟客户端）**：连接到本地存储服务以访问缓存。

#### 1. 启动服务（主服务和存储服务）

首先，启动 `主服务` 和 `存储服务`。`存储服务` 充当真实客户端。

**启动主服务：**
```bash
mooncake_master --eviction_high_watermark_ratio=0.95
```

**启动存储服务（真实客户端）：** 关键的是，默认端口（50052）用于内部 RPC，虚拟客户端将连接到此端口。
```bash
mooncake_client --global_segment_size=4GB
```

**参数说明：**

- **`host`**：（字符串，默认值："0.0.0.0"）：客户端的主机名。

- **`port`**：（整数，默认值：50052）：客户端服务监听的端口号。

- **`global_segment_size`**：（字符串，默认值："4GB"）：客户端要分配的全局段大小。

- **`master_server_address`**：（字符串，默认值："localhost:50051"）：主服务的地址。

- **`metadata_server`**：（字符串，默认值："http://localhost:8080/metadata"）：元数据服务的地址。

- **`protocol`**：（字符串，默认值："tcp"）：传输引擎使用的协议。

- **`device_name`**：（字符串，默认值：""）：传输引擎使用的设备名称。

- **`threads`**：（整数，默认值：1）：客户端使用的线程数。

#### 2. 启动 SGLang（虚拟客户端）
配置 SGLang 使用 client_server_address 参数连接到真实客户端。

**使用 sglang 参数的 extra-config 配置 Mooncake**

```bash
python -m sglang.launch_server \
    --enable-hierarchical-cache \
    --hicache-storage-backend mooncake \
    --model-path [model_path] \
    --hicache-storage-backend-extra-config '{"standalone_storage": true, "client_server_address": "127.0.0.1:50052"}'
```

**使用 JSON 文件配置 Mooncake**

SGLang 服务器可以从 `SGLANG_HICACHE_MOONCAKE_CONFIG_PATH` 加载 Mooncake 配置。

```bash
export SGLANG_HICACHE_MOONCAKE_CONFIG_PATH=/sgl-workspace/sglang/benchmark/hicache/mooncake_config.json

echo '{
    "standalone_storage": true,
    "client_server_address": "127.0.0.1:50052"
}' > ${SGLANG_HICACHE_MOONCAKE_CONFIG_PATH}

python -m sglang.launch_server \
    --enable-hierarchical-cache \
    --hicache-storage-backend mooncake \
    --model-path [model_path]
```

**使用环境变量配置 Mooncake**

```bash
MOONCAKE_STANDALONE_STORAGE=1
MOONCAKE_CLIENT="127.0.0.1:50052"
python -m sglang.launch_server \
    --enable-hierarchical-cache \
    --hicache-storage-backend mooncake \
    --model-path [model_path]
```

### 预填充/解码解耦（PD 解耦）

在 **PD 解耦** 中，`元数据服务`、`mooncake 主服务` 和可选的 `存储服务` 的配置与上述相同。不同之处在于 SGLang 引入了三个不同的角色：`预填充工作节点（prefill worker）`、`解码工作节点（decode worker）` 和 `路由器（router）`。

其中，`预填充工作节点` 支持启用 **HiCache**。要运行 PD 解耦，请从 [PD 配置](https://kvcache-ai.github.io/Mooncake/getting_started/examples/sglang-integration-v1.html) 开始，并将 HiCache 相关参数（如前所述的 `SGLang 服务器` 参数）添加到 `预填充工作节点`。

在下面的示例中，启动了一个 `预填充工作节点`、一个 `解码工作节点` 和一个 `路由器`。在 `预填充工作节点` 上启用 HiCache 以优化预填充性能。

**预填充工作节点**：

```bash
MOONCAKE_TE_META_DATA_SERVER="http://127.0.0.1:8080/metadata" \
MOONCAKE_MASTER=127.0.0.1:50051 \
MOONCAKE_PROTOCOL="rdma" \
MOONCAKE_DEVICE="mlx5_1" \
MOONCAKE_GLOBAL_SEGMENT_SIZE=4294967296 \
python -m sglang.launch_server \
    --model-path [model_path] \
    --page-size 64 \
    --enable-hierarchical-cache \
    --hicache-storage-prefetch-policy timeout \
    --hicache-storage-backend mooncake \
    --disaggregation-mode prefill \
    --disaggregation-ib-device "mlx5_1" \
    --base-gpu-id 0 \
    --port 30000
```

**解码工作节点**：

```bash
python -m sglang.launch_server \
    --model-path [model_path] \
    --page-size 64 \
    --disaggregation-mode decode \
    --disaggregation-ib-device "mlx5_1" \
    --base-gpu-id 1 \
    --port 30001
```

**路由器**：

```bash
python -m sglang_router.launch_router \
    --pd-disaggregation \
    --prefill "http://127.0.0.1:30000" \
    --decode "http://127.0.0.1:30001" \
    --host 0.0.0.0 \
    --port 8000
```

## 故障排除

**RDMA 注册失败：**

* 在某些环境中，RDMA 注册可能需要 root 权限。在这种情况下，请尝试以 root 用户运行程序。
* 在某些环境（例如 eRDMA）中，可注册的 RDMA 内存总量有上限。一旦超过此限制，注册将失败。要解决此问题，您可以降低 `MOONCAKE_GLOBAL_SEGMENT_SIZE` 的值，或减少 `SGLang 服务器` 中分配给 HiCache 的主机内存（因为此内存已完全注册到 RDMA 以实现零拷贝）。

**HiCache CPU 内存使用：**

使用 HiCache 时，KV 缓存的默认 L2 主机 DRAM（CPU 内存）大小是 L1 设备内存（GPU 内存）的 **2 倍**。

如果模型很小但 GPU 内存很大——特别是在多 TP（张量并行）设置中——这可能导致 L1 KV 缓存变得非常大，进而消耗过多的 CPU DRAM。

在这种情况下，您应该根据硬件手动配置适当的 L2 缓存大小。这可以通过设置 `--hicache-ratio` 或 `--hicache-size` 来完成。

**更多信息：**

其他故障排除信息可以在[此处](https://kvcache-ai.github.io/Mooncake/troubleshooting/troubleshooting.html)找到。

## 测试 Mooncake 存储

此测试供开发人员快速验证 MooncakeStore 类接口是否正常工作。

首先，启动 `元数据服务` 和 `主服务`。然后运行 `test_mooncake_store.py`。16MB 的全局段大小足以运行此测试。

```bash
MOONCAKE_TE_META_DATA_SERVER="http://127.0.0.1:8080/metadata" \
MOONCAKE_MASTER=127.0.0.1:50051 \
MOONCAKE_PROTOCOL="rdma" \
MOONCAKE_GLOBAL_SEGMENT_SIZE=16777216 \
python3 [test_mooncake_store.py 的路径]
```

如果所有测试通过，最后将打印 "✅ All tests passed" 消息。





在 KV Cache 数据组织方面，HiCache 基于 RadixAttention 中提出的 RadixTree 结构，进一步设计了 HiRadixTree。在 RadixAttention 中，RadixTree 的每个节点对应 GPU 内存中一段连续 Token 的 KV Cache；从根节点到叶节点的路径代表一个请求的前缀，多个请求间共享的前缀可以复用相同的节点，从而避免冗余存储。

HiRadixTree 扩展了这一思想：每个节点仍对应一段连续 Token 的 KV Cache，但额外记录了该 KV Cache 的存储位置——可能存放在本地 GPU 内存、CPU 内存、L3 存储后端，或同时存在于多个层级中。对于本地存储的数据，HiRadixTree 会维护精确的元数据，包括具体的存储地址；但为了降低开销，它并不保存或持续同步 L3 层级 KV Cache 的元数据。取而代之的是，当需要访问 L3 数据时，HiRadixTree 会实时查询存储后端，以获取必要的元数据，例如数据是否存在、位于哪台服务器以及具体的存储位置等信息。
