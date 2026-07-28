---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Examples
---

如需快速开始，请参考 [TensorRT-LLM README](README.md)。本文档提供使用 Dynamo 运行 TensorRT-LLM 的全部部署模式，包括单节点、多节点和 Kubernetes 部署。

## 目录

- [基础设施准备](#infrastructure-setup)
- [单节点示例](#single-node-examples)
- [进阶示例](#advanced-examples)
- [客户端](#client)
- [基准测试](#benchmarking)

## 基础设施准备

对于本地/裸金属开发，使用 Docker Compose 启动 etcd，并按需启动 NATS：

```bash
docker compose -f deploy/docker-compose.yml up -d
```

<Note>
- **etcd** 是可选的，但默认作为本地服务发现后端。也可以使用 `--discovery-backend file` 切换到基于文件系统的服务发现。
- **NATS** 是可选的 —— 仅在使用基于事件的 KV 路由时才需要。Worker 必须显式配置以发布事件。在 frontend 上使用 `--no-router-kv-events` 可启用基于预测的路由（不依赖事件）。
- **在 Kubernetes 上**，使用 Dynamo operator 时两者都不需要：operator 会显式设置 `DYN_DISCOVERY_BACKEND=kubernetes`，启用原生 K8s 服务发现（DynamoWorkerMetadata CRD）。
</Note>

<Tip>
每个启动脚本会在单个终端中同时运行 frontend 和 worker。你也可以在不同终端中分别执行各条命令以便于测试。每个 shell 脚本本质上就是用 `python3 -m dynamo.frontend <args>` 启动入口，用 `python3 -m dynamo.trtllm <args>` 启动 worker。
</Tip>

关于 KV 感知路由的详细行为，请参考 [Routing Concepts](../../components/router/router-concepts.md)。关于部署模式，请参考 [Router Guide](../../components/router/router-guide.md)。

## 单节点示例

### Aggregated（聚合）

```bash
cd $DYNAMO_HOME/examples/backends/trtllm
./launch/agg.sh
```

### Aggregated 配合 KV 路由

```bash
cd $DYNAMO_HOME/examples/backends/trtllm
./launch/agg_router.sh
```

### Disaggregated（分离式）

```bash
cd $DYNAMO_HOME/examples/backends/trtllm
./launch/disagg.sh
```

### Disaggregated 配合 KV 路由

<Note>
在分离式工作流中，请求会被路由到 prefill worker 以最大化 KV cache 复用。
</Note>

```bash
cd $DYNAMO_HOME/examples/backends/trtllm
./launch/disagg_router.sh
```

### Aggregated 配合 Multi-Token Prediction (MTP) 与 DeepSeek R1

```bash
cd $DYNAMO_HOME/examples/backends/trtllm

export AGG_ENGINE_ARGS=./engine_configs/deepseek-r1/agg/mtp/mtp_agg.yaml
export SERVED_MODEL_NAME="nvidia/DeepSeek-R1-FP4"
# nvidia/DeepSeek-R1-FP4 is a large model
export MODEL_PATH="nvidia/DeepSeek-R1-FP4"
./launch/agg.sh
```

<Note>
- 前两个推理请求会有显著的延迟。请在开始基准测试前先发送预热请求。
- MTP 性能取决于预测 token 的接受率，而该接受率与基准测试所用数据集或查询有关。此外，使用 MTP 时通常应忽略 `ignore_eos`，或将其设为 `false`，以避免推测出垃圾输出并得到不切实际的接受率。
</Note>

## 进阶示例

### 多节点部署

关于多节点部署的完整说明，请参考 [Multinode Examples](./multinode/trtllm-multinode-examples.md) 指南。其中提供了在多节点上运行 Dynamo + TensorRT-LLM 的逐步部署示例与配置建议。该指南以 DeepSeek-R1 作为模型进行演示，但你可以通过修改相关配置文件来轻松适配任何受支持的模型。如果单个 worker 可以放进单节点，也可以参考 [Llama4 + Eagle](./trtllm-llama4-plus-eagle.md) 来了解这些脚本的使用方式。

### Speculative Decoding（投机解码）

- **[Llama 4 Maverick Instruct + Eagle Speculative Decoding](./trtllm-llama4-plus-eagle.md)**

### 模型专属指南

- **[Gemma3 with Sliding Window Attention](./trtllm-gemma3-sliding-window-attention.md)**
- **[GPT-OSS-120b](./trtllm-gpt-oss.md)** —— 支持工具调用的推理模型

### Kubernetes 部署

完整的 Kubernetes 部署说明、配置以及故障排查请参考 [TensorRT-LLM Kubernetes Deployment Guide](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/trtllm/deploy/README.md)。

## 客户端

请参考 [client](../sglang/README.md#testing-the-deployment) 章节了解如何向部署发送请求。

<Note>
向多节点部署发送请求时，请将请求发送到运行 `python3 -m dynamo.frontend <args>` 的节点。
</Note>

## 基准测试

如需使用 AIPerf 对部署进行基准测试，请参考此工具脚本，并根据你的部署调整 `model` 名称与 `host`：[perf.sh](https://github.com/ai-dynamo/dynamo/blob/main/benchmarks/llm/perf.sh)
