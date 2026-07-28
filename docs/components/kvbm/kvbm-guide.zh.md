---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: KVBM Guide
subtitle: Enable KV offloading using KV Block Manager (KVBM) for Dynamo deployments
---

Dynamo KV Block Manager（KVBM）是一个可扩展的运行时（runtime）组件，用于在异构、分布式环境下处理推理任务的 KV（Key-Value）block 的内存分配、管理与远程共享。它充当 vLLM、TensorRT-LLM 等框架的统一内存层与写穿透（write-through）缓存。

KVBM 是模块化的，既可以通过 `pip install kvbm` 独立使用，也可以作为完整 Dynamo 栈中的内存管理组件。本指南介绍 Dynamo KV Block Manager（KVBM）以及其他 KV 缓存（KV cache）管理系统的安装、配置与部署。

## 独立运行 KVBM

KVBM 可以在不使用 Dynamo 其他部分的情况下独立使用：

```bash
pip install kvbm
```

版本兼容性参见 [support matrix](../../reference/support-matrix.md)。

### 从源码构建

要从源码构建 KVBM，请参见 [KVBM bindings README](https://github.com/ai-dynamo/dynamo/tree/main/lib/bindings/kvbm/README.md#build-from-source) 中的详细说明。

## 在 Dynamo 中结合 vLLM 使用 KVBM

### Docker 准备

```bash
# Start up etcd for KVBM leader/worker registration and discovery
docker compose -f deploy/docker-compose.yml up -d
```

下面两种方式任选其一即可获得集成了 KVBM 的 Dynamo vLLM 容器。后续的服务命令两种方式相同。

**方案 A：预构建的 NGC 容器（推荐快速上手）**

```bash
docker run --gpus all --network host --rm -it nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
```

完整搭建说明参见 [本地安装指南](../../getting-started/local-installation.md)；可用版本参见 [Release Artifacts](../../reference/release-artifacts.md#container-images)。

**方案 B：从源码构建**

```bash
# Build a dynamo vLLM container (KVBM is built in by default)
# NOTE: render.py defaults to --platform linux/amd64. On ARM64 hosts, pass --platform linux/arm64.
python container/render.py --framework vllm --target runtime --output-short-filename
docker build -t dynamo:latest-vllm-runtime -f container/rendered.Dockerfile .

# Launch the container
container/run.sh --image dynamo:latest-vllm-runtime -it --mount-workspace --use-nixl-gds
```

### 聚合（Aggregated）服务

```bash
cd $DYNAMO_HOME/examples/backends/vllm
./launch/agg_kvbm.sh
```

#### 验证部署

```bash
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [{"role": "user", "content": "Hello, how are you?"}],
    "stream": false,
    "max_tokens": 10
  }'
```

#### 替代方案：直接使用 vllm serve

也可以直接通过 `vllm serve` 启用 KVBM：

```bash
vllm serve --kv-transfer-config '{"kv_connector":"DynamoConnector","kv_role":"kv_both", "kv_connector_module_path": "kvbm.vllm_integration.connector"}' Qwen/Qwen3-0.6B
```

## 在 Dynamo 中结合 TensorRT-LLM 使用 KVBM

> [!NOTE]
> **前置条件：**
> - 启动前确保 `etcd` 与 `nats` 已在运行
> - KVBM 仅支持 TensorRT-LLM 的 PyTorch 后端（backend）
> - 关闭部分复用（`enable_partial_reuse: false`）以提升 offloading 缓存命中
> - KVBM 需要 TensorRT-LLM v1.2.0rc2 或更新版本

### Docker 准备

```bash
# Start up etcd for KVBM leader/worker registration and discovery
docker compose -f deploy/docker-compose.yml up -d
```

下面两种方式任选其一即可获得集成了 KVBM 的 Dynamo TensorRT-LLM 容器。后续的服务命令两种方式相同。

**方案 A：预构建的 NGC 容器（推荐快速上手）**

```bash
docker run --gpus all --network host --rm -it nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:1.1.1
```

完整搭建说明参见 [本地安装指南](../../getting-started/local-installation.md)；可用版本参见 [Release Artifacts](../../reference/release-artifacts.md#container-images)。

**方案 B：从源码构建**

```bash
# Build a dynamo TRTLLM container (KVBM is built in by default)
# NOTE: render.py defaults to --platform linux/amd64. On ARM64 hosts, pass --platform linux/arm64.
python container/render.py --framework trtllm --target runtime --output-short-filename
docker build -t dynamo:latest-trtllm-runtime -f container/rendered.Dockerfile .

# Launch the container
container/run.sh --image dynamo:latest-trtllm-runtime -it --mount-workspace --use-nixl-gds
```

### 聚合服务

```bash
# Write the LLM API config
cat > "/tmp/kvbm_llm_api_config.yaml" <<EOF
backend: pytorch
cuda_graph_config: null
kv_cache_config:
  enable_partial_reuse: false
  free_gpu_memory_fraction: 0.80
kv_connector_config:
  connector_module: kvbm.trtllm_integration.connector
  connector_scheduler_class: DynamoKVBMConnectorLeader
  connector_worker_class: DynamoKVBMConnectorWorker
EOF

# Start dynamo frontend
python3 -m dynamo.frontend --http-port 8000 &

# Serve the model with KVBM
python3 -m dynamo.trtllm \
  --model-path Qwen/Qwen3-0.6B \
  --served-model-name Qwen/Qwen3-0.6B \
  --extra-engine-args /tmp/kvbm_llm_api_config.yaml &
```

#### 验证部署

```bash
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [{"role": "user", "content": "Hello, how are you?"}],
    "stream": false,
    "max_tokens": 30
  }'
```

#### 替代方案：使用 trtllm-serve

```bash
trtllm-serve Qwen/Qwen3-0.6B --host localhost --port 8000 --backend pytorch --extra_llm_api_options /tmp/kvbm_llm_api_config.yaml
```

## 在 Dynamo 中使用 SGLang HiCache

SGLang 的层次化缓存（Hierarchical Cache，HiCache）把 KV 缓存存储从 GPU 显存扩展到主机 CPU 内存。当使用 NIXL 作为存储后端时，HiCache 与 Dynamo 的内存基础设施集成。

### 快速开始

```bash
# Start SGLang worker with HiCache enabled
python -m dynamo.sglang \
  --model-path Qwen/Qwen3-0.6B \
  --host 0.0.0.0 --port 8000 \
  --enable-hierarchical-cache \
  --hicache-ratio 2 \
  --hicache-write-policy write_through \
  --hicache-storage-backend nixl

# In a separate terminal, start the frontend
python -m dynamo.frontend --http-port 8000

# Send a test request
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": false,
    "max_tokens": 30
  }'
```

> **了解更多：** 详细配置、部署示例与故障排查，请参见 [SGLang HiCache 集成指南](../../integrations/sglang-hicache.md)。

## 与 KVBM 配合的解耦（Disaggregated）服务

KVBM 支持解耦服务 —— 把预填充（prefill）与解码（decode）操作分别运行在不同的 worker 上。在 prefill worker 上启用 KVBM 来卸载（offload） KV 缓存。

### 解耦服务（vLLM）

```bash
# 1P1D - one prefill worker and one decode worker
# NOTE: requires at least 2 GPUs
cd $DYNAMO_HOME/examples/backends/vllm
./launch/disagg_kvbm.sh

# 2P2D - two prefill workers and two decode workers
# NOTE: requires at least 4 GPUs
cd $DYNAMO_HOME/examples/backends/vllm
./launch/disagg_kvbm_2p2d.sh
```

### 解耦服务（TRT-LLM）

```bash
# Launch prefill worker with KVBM
python3 -m dynamo.trtllm \
  --model-path Qwen/Qwen3-0.6B \
  --served-model-name Qwen/Qwen3-0.6B \
  --extra-engine-args /tmp/kvbm_llm_api_config.yaml \
  --disaggregation-mode prefill &
```

## 配置

### 缓存层级配置

通过环境变量配置 KVBM 的缓存层级：

```bash
# Option 1: CPU cache only (GPU -> CPU offloading)
export DYN_KVBM_CPU_CACHE_GB=4  # 4GB of pinned CPU memory

# Option 2: Both CPU and Disk cache (GPU -> CPU -> Disk tiered offloading)
export DYN_KVBM_CPU_CACHE_GB=4
export DYN_KVBM_DISK_CACHE_GB=8  # 8GB of disk

# [Experimental] Option 3: Disk cache only (GPU -> Disk direct offloading)
# NOTE: Experimental, may not provide optimal performance
# NOTE: Disk offload filtering not supported with this option
export DYN_KVBM_DISK_CACHE_GB=8
```

也可以指定具体的 block 数而不是 GB：
- `DYN_KVBM_CPU_CACHE_OVERRIDE_NUM_BLOCKS`
- `DYN_KVBM_DISK_CACHE_OVERRIDE_NUM_BLOCKS`

> [!NOTE] KVBM 是写穿透缓存，配置时容易踩坑。容量随启用更多层级应当**逐级增大**。例如，如果 GPU 设备上为 KV cache 存储分配了 100GB，则需要配置 `DYN_KVBM_CPU_CACHE_GB >= 100`；磁盘缓存同理：`DYN_KVBM_DISK_CACHE_GB >= DYN_KVBM_CPU_CACHE_GB`。如果 CPU 缓存被配置得比设备缓存还小，那么使用 KVBM **不会带来任何收益**，许多情况下还会因为 KVBM 在每次前向传播后都把 block 从 GPU 卸到 CPU 而出现性能劣化。要确定 `DYN_KVBM_CPU_CACHE_GB` 的最低值，请参考你 LLM 引擎的 KV cache 配置。

### SSD 寿命保护

启用磁盘 offloading 时，磁盘 offload 过滤（disk offload filtering）默认开启，以延长 SSD 寿命。当前策略只在 block 频率（frequency）≥ 2 时把 KV block 从 CPU 卸到磁盘。频率会在缓存命中时翻倍（初值为 1），并在每个时间衰减步骤里递减 1。

要关闭磁盘 offload 过滤：

```bash
export DYN_KVBM_DISABLE_DISK_OFFLOAD_FILTER=true
```

### 适用 MLA 模型的 NCCL 复制模式

对于 MLA（Multi-Layer Attention）模型（如 DeepSeek），KVBM 可以使用 **NCCL 复制模式**：仅由 rank 0 从 G2/G3 存储加载 KV block，再通过 NCCL 广播给所有 GPU。这避免了重复加载，能在多 GPU 共享同一份复制的 KV cache 时提升性能。

**启用 NCCL MLA 模式：**

```bash
export DYN_KVBM_NCCL_MLA_MODE=true
```

**要求：**

- 必须初始化 MPI（例如使用 `mpirun` 或等价方式启动），以便 NCCL 能拿到 rank 与 world size。
- 为获得基于广播的最优复制效果，构建 KVBM 时启用 NCCL feature：`cargo build -p kvbm --features nccl`。否则 connector 会回退到 worker 级复制（每个 GPU 各自加载）。

未启用（默认）时，每个 GPU 独立加载 KV block。在使用 KVBM 跑 MLA 模型时，建议设置 `DYN_KVBM_NCCL_MLA_MODE=true` 以使用 NCCL 广播优化。

## 启用并查看 KVBM 指标

### 搭建监控栈

```bash
# Start basic services (etcd & natsd), along with Prometheus and Grafana
docker compose -f deploy/docker-observability.yml up -d
```

### 为 vLLM 启用指标

```bash
DYN_KVBM_METRICS=true \
DYN_KVBM_CPU_CACHE_GB=20 \
python -m dynamo.vllm \
    --model Qwen/Qwen3-0.6B \
    --enforce-eager \
    --kv-transfer-config '{"kv_connector":"DynamoConnector","kv_connector_module_path":"kvbm.vllm_integration.connector","kv_role":"kv_both"}'
```

### 为 TensorRT-LLM 启用指标

```bash
DYN_KVBM_METRICS=true \
DYN_KVBM_CPU_CACHE_GB=20 \
python3 -m dynamo.trtllm \
  --model-path Qwen/Qwen3-0.6B \
  --served-model-name Qwen/Qwen3-0.6B \
  --extra-engine-args /tmp/kvbm_llm_api_config.yaml &
```

### 防火墙配置（可选）

```bash
# If firewall blocks KVBM metrics ports
sudo ufw allow 6880/tcp
```

### 查看指标

访问 http://localhost:3000 进入 Grafana（默认登录：`dynamo`/`dynamo`），找到 **KVBM Dashboard**。

### 可用指标

| 指标 | 描述 |
|--------|-------------|
| `kvbm_matched_tokens` | 命中的 token 数 |
| `kvbm_offload_blocks_d2h` | 设备 → 主机的 offload block 数 |
| `kvbm_offload_blocks_h2d` | 主机 → 磁盘的 offload block 数 |
| `kvbm_offload_blocks_d2d` | 设备 → 磁盘的 offload block 数（绕过主机） |
| `kvbm_onboard_blocks_d2d` | 磁盘 → 设备的 onboard block 数 |
| `kvbm_onboard_blocks_h2d` | 主机 → 设备的 onboard block 数 |
| `kvbm_host_cache_hit_rate` | 主机缓存命中率（0.0-1.0） |
| `kvbm_disk_cache_hit_rate` | 磁盘缓存命中率（0.0-1.0） |

## KVBM 基准测试

使用 [LMBenchmark](https://github.com/LMCache/LMBenchmark) 评估 KVBM 性能。

### 准备

```bash
git clone https://github.com/LMCache/LMBenchmark.git
cd LMBenchmark/synthetic-multi-round-qa
```

### 运行基准

```bash
# Synthetic multi-turn chat dataset
# Arguments: model, endpoint, output prefix, qps
./long_input_short_output_run.sh \
    "Qwen/Qwen3-0.6B" \
    "http://localhost:8000" \
    "benchmark_kvbm" \
    1
```

平均 TTFT 等性能数据会出现在输出中。

> **提示：** 启用了指标的话，可以在 Grafana 仪表板上观察 KV 的卸载与 onboard 行为。

### 基线对比

#### vLLM 基线（不启用 KVBM）

```bash
vllm serve Qwen/Qwen3-0.6B
```

#### TensorRT-LLM 基线（不启用 KVBM）

```bash
# Create config without kv_connector_config
cat > "/tmp/llm_api_config.yaml" <<EOF
backend: pytorch
cuda_graph_config: null
kv_cache_config:
  enable_partial_reuse: false
  free_gpu_memory_fraction: 0.80
EOF

trtllm-serve Qwen/Qwen3-0.6B --host localhost --port 8000 --backend pytorch --extra_llm_api_options /tmp/llm_api_config.yaml
```

## 故障排查

### TTFT 没有性能提升

**症状：** 启用 KVBM 后没有 TTFT 改善，甚至出现性能劣化。

**原因：** KVBM 上的前缀（prefix）缓存命中不足，无法复用已被卸载的 KV block。

**解决：** 启用 KVBM 指标，到 Grafana 仪表板查看 `Onboard Blocks - Host to Device` 与 `Onboard Blocks - Disk to Device`。大量 onboard 的 KV block 表明缓存复用情况良好：

![Grafana Example](../../assets/img/kvbm-metrics-grafana.png)

### KVBM Worker 初始化超时

**症状：** 在分配大块内存或磁盘存储时 KVBM 启动失败。

**解决：** 增大 leader-worker 初始化超时（默认 1800 秒）：

```bash
export DYN_KVBM_LEADER_WORKER_INIT_TIMEOUT_SECS=3600  # 1 hour
```

### 磁盘 offload 启动失败

**症状：** 启用磁盘 offloading 时 KVBM 启动失败。

**原因：** 文件系统不支持 `fallocate()`（如 Lustre、某些网络文件系统），或存储后端需要不同的方式来设置 `O_DIRECT`。

**解决：**

1. 如果不支持 `fallocate()`，启用 zerofill 回退：

```bash
export DYN_KVBM_DISK_ZEROFILL_FALLBACK=true
```

2. 如果你的文件系统忽略 `fcntl(F_SETFL, O_DIRECT)`（如 IBM Storage Scale），改在 open 时传入 `O_DIRECT`：

```bash
export DYN_KVBM_DISK_ALLOCATOR_TYPE=open-direct
```

`DYN_KVBM_DISK_ALLOCATOR_TYPE` 的支持值：
- `default`：在文件创建后通过 `fcntl` 设置 `O_DIRECT`。在大多数 POSIX 文件系统（ext4、XFS、Lustre 等）上有效。
- `open-direct`：在 `mkostemp` 打开文件时直接传入 `O_DIRECT`。在 `fcntl(F_SETFL, O_DIRECT)` 被忽略的文件系统（如 IBM Storage Scale）上必需。

3. 如果遇到 "write all error" 或 EINVAL（errno 22），或希望在不带 `O_DIRECT` 的情况下调试：

```bash
export DYN_KVBM_DISK_DISABLE_O_DIRECT=true
```

## 本地开发

在 Dynamo 容器内修改了 KVBM 相关代码（Rust 和/或 Python）后：

```bash
cd /workspace/lib/bindings/kvbm
uv pip install maturin[patchelf]
maturin build --release --out /workspace/dist
uv pip install --upgrade --force-reinstall --no-deps /workspace/dist/kvbm*.whl
```

要使用 [Nsight Systems](https://developer.nvidia.com/nsight-systems) 做性能分析，可参考下面的步骤（以 vLLM 为例）。KVBM 在顶层 KV Connector API 上添加了 NVTX 注解（搜索 `@nvtx_annotate`）。如需更多注解，请添加后再重建。
```bash
# build and run local-dev container, which contains nsys
python container/render.py --framework=vllm --target=local-dev --output-short-filename
docker build --build-arg USER_UID=$(id -u) --build-arg USER_GID=$(id -g) -f container/rendered.Dockerfile -t dynamo:latest-vllm-local-dev .

container/run.sh --image dynamo:latest-vllm-local-dev -it --mount-workspace --use-nixl-gds

# export nsys to PATH
# NOTE: change the version accordingly
export PATH=/opt/nvidia/nsight-systems/2025.5.1/bin:$PATH

# example usage of nsys: delay 30 seconds and then capture 60 seconds
python -m dynamo.frontend &

DYN_KVBM_CPU_CACHE_GB=10 \
nsys profile -o /tmp/kvbm-nsys --trace-fork-before-exec=true --cuda-graph-trace=node --delay 30 --duration 60 \
python -m dynamo.vllm --model Qwen/Qwen3-0.6B --kv-transfer-config '{"kv_connector":"DynamoConnector","kv_connector_module_path":"kvbm.vllm_integration.connector","kv_role":"kv_both"}'
```

## 另见

- [KVBM 概览](README.md) —— 对 KV Caching、KVBM 及其架构的快速介绍
- [KVBM 设计](../../design-docs/kvbm-design.md) —— 深入 KVBM 架构
- [LMCache 集成](../../integrations/lmcache-integration.md)
- [FlexKV 集成](../../integrations/flexkv-integration.md)
- [SGLang HiCache](../../integrations/sglang-hicache.md)
