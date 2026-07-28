---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: FlexKV
---

## 简介

[FlexKV](https://github.com/taco-project/FlexKV) 是由腾讯云 TACO 团队与 NVIDIA 联合社区共同开发的、可扩展的分布式 KV 缓存（KV cache）卸载（offloading）运行时。它作为 SGLang、TensorRT-LLM、vllm 等推理（inference）引擎的统一 KV 缓存层。

### 关键特性

- **多级缓存**：支持将 KV 缓存卸载到 CPU 内存、本地 SSD 与可扩展存储（云存储）
- **分布式 KV 缓存复用**：通过分布式 RadixTree 在多节点间共享 KV 缓存
- **高性能 I/O**：支持 io_uring 和 GPU Direct Storage（GDS），加速数据传输
- **异步操作**：通过预取（prefetch），get/put 操作可与计算重叠


## 前置条件

1. **已安装支持 vLLM 的 Dynamo**
2. **基础设施服务已运行**：
   ```bash
   docker compose -f deploy/docker-compose.yml up -d
   ```
3. **已安装 FlexKV**：
   ```bash
   git clone https://github.com/taco-project/FlexKV.git
   cd FlexKV
   ./build.sh
   ```
4. **可选：SSD 卸载相关依赖**（仅在使用 CPU + SSD 分级卸载时需要）：
   ```bash
   apt install liburing-dev libxxhash-dev
   ```

## 快速开始

### 启用 FlexKV

设置 `DYNAMO_USE_FLEXKV` 环境变量并使用 `--kv-transfer-config` 参数：

```bash
export DYNAMO_USE_FLEXKV=1
python -m dynamo.vllm --model Qwen/Qwen3-0.6B --kv-transfer-config '{"kv_connector":"FlexKVConnectorV1","kv_role":"kv_both"}'
```

## 聚合（Aggregated）部署

### 基础搭建

```bash
# Terminal 1: Start frontend
python -m dynamo.frontend &

# Terminal 2: Start vLLM worker with FlexKV
DYNAMO_USE_FLEXKV=1 \
FLEXKV_CPU_CACHE_GB=32 \
  python -m dynamo.vllm --model Qwen/Qwen3-0.6B --kv-transfer-config '{"kv_connector":"FlexKVConnectorV1","kv_role":"kv_both"}'
```

### 启用 KV 感知路由（KV-Aware Routing）

针对多 worker 部署，开启 KV 感知路由以最大化缓存复用：

```bash
# Terminal 1: Start frontend with KV router
python -m dynamo.frontend \
    --router-mode kv \
    --router-reset-states &

# Terminal 2: Worker 1
DYNAMO_USE_FLEXKV=1 \
FLEXKV_CPU_CACHE_GB=32 \
FLEXKV_SERVER_RECV_PORT="ipc:///tmp/flexkv_server_0" \
CUDA_VISIBLE_DEVICES=0 \
python -m dynamo.vllm \
    --model Qwen/Qwen3-0.6B \
    --kv-transfer-config '{"kv_connector":"FlexKVConnectorV1","kv_role":"kv_both"}' \
    --gpu-memory-utilization 0.2 \
    --kv-events-config '{"publisher":"zmq","topic":"kv-events","endpoint":"tcp://*:20080","enable_kv_cache_events":true}' &

# Terminal 3: Worker 2
DYNAMO_USE_FLEXKV=1 \
FLEXKV_CPU_CACHE_GB=32 \
FLEXKV_SERVER_RECV_PORT="ipc:///tmp/flexkv_server_1" \
CUDA_VISIBLE_DEVICES=1 \
python -m dynamo.vllm \
    --model Qwen/Qwen3-0.6B \
    --kv-transfer-config '{"kv_connector":"FlexKVConnectorV1","kv_role":"kv_both"}' \
    --gpu-memory-utilization 0.2 \
    --kv-events-config '{"publisher":"zmq","topic":"kv-events","endpoint":"tcp://*:20081","enable_kv_cache_events":true}'
```

## 解耦（Disaggregated）部署

> **注意：** 解耦模式下的 FlexKV 服务为实验性。预填充（prefill）worker 必须使用 `PdConnector` 并嵌套两个子 connector：`FlexKVConnectorV1`（KV 缓存卸载）与 `NixlConnector`（P/D KV 传输）。在解耦模式下单独使用 `FlexKVConnectorV1` 作为顶层 connector **不被支持**，会导致 `TypeError`。

FlexKV 可与解耦的 prefill/decode 部署一起使用。预填充 worker 使用 FlexKV 进行 KV 缓存卸载，而 NIXL 负责 prefill 与 decode worker 之间的 KV 传输。`PdConnector` 将两者包装在一起协同工作。

### 受支持的 connector 配置

| 角色 | Connector | 说明 |
|------|-----------|-------------|
| Decode worker | `NixlConnector` | 通过 NIXL 从 prefill worker 拉取 KV 块 |
| Prefill worker | `PdConnector` 包装 `[FlexKVConnectorV1, NixlConnector]` | FlexKV 进行 KV 块卸载/上线；NIXL 将其交付给 decode |

```bash
# Terminal 1: Start frontend
python -m dynamo.frontend &

# Terminal 2: Decode worker (without FlexKV)
CUDA_VISIBLE_DEVICES=0 python -m dynamo.vllm --model Qwen/Qwen3-0.6B \
  --kv-transfer-config '{"kv_connector":"NixlConnector","kv_role":"kv_both"}' &

# Terminal 3: Prefill worker (with FlexKV + NIXL via PdConnector)
DYN_VLLM_KV_EVENT_PORT=20081 \
VLLM_NIXL_SIDE_CHANNEL_PORT=20097 \
DYNAMO_USE_FLEXKV=1 \
FLEXKV_CPU_CACHE_GB=32 \
CUDA_VISIBLE_DEVICES=1 \
  python -m dynamo.vllm \
  --model Qwen/Qwen3-0.6B \
  --is-prefill-worker \
  --kv-transfer-config '{"kv_connector":"PdConnector","kv_role":"kv_both","kv_connector_extra_config":{"connectors":[{"kv_connector":"FlexKVConnectorV1","kv_role":"kv_both"},{"kv_connector":"NixlConnector","kv_role":"kv_both"}]},"kv_connector_module_path":"kvbm.vllm_integration.connector"}' \
  --kv-events-config '{"publisher":"zmq","topic":"kv-events","endpoint":"tcp://*:20081","enable_kv_cache_events":true}'
```

也可以直接使用提供的启动脚本：

```bash
examples/backends/vllm/launch/disagg_flexkv.sh
```

## 配置

### 环境变量

| 变量 | 说明 | 默认值 |
|----------|-------------|---------|
| `DYNAMO_USE_FLEXKV` | 启用 FlexKV 集成 | `0`（禁用） |
| `FLEXKV_CPU_CACHE_GB` | CPU 内存缓存大小（GB） | 必填 |
| `FLEXKV_CONFIG_PATH` | FlexKV YAML 配置文件路径 | 未设置 |
| `FLEXKV_SERVER_RECV_PORT` | FlexKV 服务的 IPC 端口 | 自动 |

### 仅 CPU 卸载

简单的 CPU 内存卸载：

```bash
unset FLEXKV_CONFIG_PATH
export FLEXKV_CPU_CACHE_GB=32
```

### CPU + SSD 分级卸载

如需 SSD 存储的多级卸载，可创建配置文件：

```bash
cat > ./flexkv_config.yml <<EOF
cpu_cache_gb: 32
ssd_cache_gb: 1024
ssd_cache_dir: /data0/flexkv_ssd/;/data1/flexkv_ssd/
enable_gds: false
EOF

export FLEXKV_CONFIG_PATH="./flexkv_config.yml"
```

### 配置选项

| 选项 | 说明 |
|--------|-------------|
| `cpu_cache_gb` | CPU 内存缓存大小（GB） |
| `ssd_cache_gb` | SSD 缓存大小（GB） |
| `ssd_cache_dir` | SSD 缓存目录（多个 SSD 用分号分隔） |
| `enable_gds` | 启用 SSD I/O 的 GPU Direct Storage |

> **注意：** 完整配置选项参见 [FlexKV 配置参考](https://github.com/taco-project/FlexKV/blob/main/docs/flexkv_config_reference/README_en.md)。

## 分布式 KV 缓存复用

FlexKV 支持在多节点之间分布式复用 KV 缓存。具备：

- **分布式 RadixTree**：每个节点维护全局索引的本地快照
- **租约（Lease）机制**：保证跨节点传输时的数据有效性
- **基于 RDMA 的传输**：使用 Mooncake Transfer Engine 实现高性能 KV 缓存传输

搭建说明参见 [FlexKV 分布式复用指南](https://github.com/taco-project/FlexKV/blob/main/docs/dist_reuse/README_en.md)。

## 架构

FlexKV 由三大核心模块组成：

### StorageEngine

初始化三级缓存（GPU → CPU → SSD/Cloud）。它将多个 token 组成块（block），按块存储 KV 缓存，KV 形状与 GPU 内存中的形状保持一致。

### GlobalCacheEngine

控制平面（control plane），决定数据传输方向并识别源/目的块 ID。包含：
- 用于前缀匹配的 RadixTree
- 用于跟踪空间使用并触发淘汰的内存池

### TransferEngine

数据平面（data plane），执行实际数据传输：
- 多线程并行传输
- 高性能 I/O（io_uring、GDS）
- 与计算重叠的异步操作

## 验证部署

```bash
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": false,
    "max_tokens": 30
  }'
```

## 参见

- [FlexKV GitHub 仓库](https://github.com/taco-project/FlexKV)
- [FlexKV vLLM 适配器文档](https://github.com/taco-project/FlexKV/blob/main/docs/vllm_adapter/README_en.md)
