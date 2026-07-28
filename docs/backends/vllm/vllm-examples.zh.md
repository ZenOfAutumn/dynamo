---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Examples
---

# vLLM 示例

快速上手说明请参见 [vLLM README](README.md)。本文档汇总了在 Dynamo 上运行 vLLM
的所有部署形态，包括聚合（aggregated）、解耦（disaggregated）、KV 路由
（KV-routed）以及专家并行（expert-parallel）等配置。

## 目录

- [基础设施准备](#infrastructure-setup)
- [LLM 服务化](#llm-serving)
- [进阶示例](#advanced-examples)
- [Kubernetes 部署](#kubernetes-deployment)
- [故障排查](#troubleshooting)

## 基础设施准备

对于本地 / 裸机开发，使用 Docker Compose 启动 etcd 与（可选的）NATS：

```bash
docker compose -f deploy/docker-compose.yml up -d
```

<Note>
- **etcd** 是可选的，但它是默认的本地服务发现后端（discovery backend）。也可使用基于文件的发现方式（运行 `python -m dynamo.vllm --help` 查看 `--discovery-backend` 选项）。
- **NATS** 仅在使用基于事件的 KV 路由（KV routing with events）时才需要。基于预测的路由不需要 NATS。
- **在 Kubernetes 上**，使用 Dynamo operator 时上述两者都不需要。
</Note>

<Tip>
每个启动脚本会在同一个终端中同时启动前端（frontend）与 worker。也可以将各条命令分别放在不同终端运行，便于查看日志。对于使用 Dynamo 的 AI agent，可以把启动脚本放到后台运行，并使用 `curl` 命令测试部署。
</Tip>

## LLM 服务化

### 聚合（Aggregated）服务

最简单的部署形态：单个 worker 同时承担预填充（prefill）与解码（decode）。需要
1 张 GPU。

在 CUDA 设备上运行：

```bash
cd $DYNAMO_HOME/examples/backends/vllm
bash launch/agg.sh
```

在 XPU 上运行：

```bash
cd $DYNAMO_HOME/examples/backends/vllm
bash launch/xpu/agg_xpu.sh
```

### 聚合服务 + KV 路由

在 [KV-aware 路由器](../../components/router/README.md) 后挂两个 worker，以最大化
缓存复用。需要 2 张 GPU。

在 CUDA 设备上运行：

```bash
cd $DYNAMO_HOME/examples/backends/vllm
bash launch/agg_router.sh
```
在 XPU 上运行：
```bash
cd $DYNAMO_HOME/examples/backends/vllm
bash launch/xpu/agg_router_xpu.sh
```


这会以 KV 路由模式启动前端，并启动两个通过 ZMQ 发布 KV 事件的 worker。

### 解耦（Disaggregated）服务

将 prefill 与 decode 拆成各自独立的 worker，并通过 NIXL 进行 KV 缓存（KV cache）
传输。需要 2 张 GPU。

```bash
cd $DYNAMO_HOME/examples/backends/vllm
bash launch/disagg.sh
```

### 解耦服务 + KV 路由

扩展到 2 个 prefill + 2 个 decode worker，并在两侧池上同时使用 KV-aware 路由。
需要 4 张 GPU。

```bash
cd $DYNAMO_HOME/examples/backends/vllm
bash launch/disagg_router.sh
```

前端运行在 KV 路由模式，会自动发现 prefill worker 并激活内部的 prefill 路由器。

### 数据并行 / 专家并行

启动 4 个数据并行（data-parallel）worker，使用专家并行（expert parallelism），
后挂 KV-aware 路由器。使用一个 Mixture-of-Experts 模型（`Qwen/Qwen3-30B-A3B`）。
需要 4 张 GPU。

```bash
cd $DYNAMO_HOME/examples/backends/vllm
bash launch/dep.sh
```

<Tip>
跑一个解耦示例，并在它运行起来后再加一个 prefill worker 试试看！系统会自动发现并使用新的 worker。
</Tip>

## 进阶示例

### 推测解码（Speculative Decoding）

以 **Eagle3** 作为草稿模型（draft model），运行 **Meta-Llama-3.1-8B-Instruct**，
在保持精度的同时加速推理。

**指南：** [Speculative Decoding Quickstart](../../features/speculative-decoding/speculative-decoding-vllm.md)

> **另请参见：** [Speculative Decoding Feature Overview](../../features/speculative-decoding/README.md) 了解跨后端文档。

### 多模态

通过 vLLM-Omni 集成提供多模态模型服务。

**指南：** [vLLM-Omni](vllm-omni.md)

### 多节点

借助 Dynamo 的分布式能力将 vLLM 部署在多个节点上。多节点部署要求节点间网络互通，
且防火墙允许 NATS / ETCD 通信。

在 head 节点上启动 NATS / ETCD，使所有 worker 节点都能访问：

```bash
# 在 head 节点上
docker compose -f deploy/docker-compose.yml up -d

# 在所有节点上设置
export HEAD_NODE_IP="<your-head-node-ip>"
export NATS_SERVER="nats://${HEAD_NODE_IP}:4222"
export ETCD_ENDPOINTS="${HEAD_NODE_IP}:2379"
```

对于多节点的张量 / 流水并行（TP × PP 超过单节点 GPU 数量的情况），参见
[`launch/multi_node_tp.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/multi_node_tp.sh)。
关于分布式执行的更多细节，参见
[vLLM 多进程文档](https://docs.vllm.ai/en/stable/serving/parallelism_scaling/#running-vllm-with-multiprocessing)。

### DeepSeek-R1

Dynamo 通过数据并行 attention 与 wide expert parallelism 支持 DeepSeek R1。每个
DP attention rank 都是一个独立的 Dynamo 组件，会发出自己的 KV 事件与指标
（metrics）。

在 2 个节点（16 张 GPU，dp=16）上运行：

```bash
# Node 0
cd $DYNAMO_HOME/examples/backends/vllm
./launch/dsr1_dep.sh --num-nodes 2 --node-rank 0 --gpus-per-node 8 --master-addr <node-0-addr>

# Node 1
./launch/dsr1_dep.sh --num-nodes 2 --node-rank 1 --gpus-per-node 8 --master-addr <node-0-addr>
```

可配置选项见 [`launch/dsr1_dep.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/dsr1_dep.sh)。

## Kubernetes 部署

完整的 Kubernetes 部署说明、配置与故障排查，请参见
[vLLM Kubernetes 部署指南](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/deploy/README.md)。

也可参考 [Kubernetes 部署指南](../../kubernetes/README.md) 获取通用的 Dynamo K8s
文档。

## 故障排查

### Worker 启动失败并出现 NIXL 错误

确保 NIXL 已安装，且 side-channel 端口未冲突。在多 worker 配置下，每个 worker
都需要一个唯一的 `VLLM_NIXL_SIDE_CHANNEL_PORT`。

### KV 路由结果不正确

使用 KV-aware 路由时，确保所有 vLLM 进程都设置了 `PYTHONHASHSEED=0`。详情参见
[Hashing Consistency](vllm-reference-guide.md#hashing-consistency-for-kv-events)。

### 启动时 GPU OOM

如果上一次运行残留了僵尸 GPU 进程，下一次启动可能会 OOM。检查僵尸进程：

```bash
nvidia-smi  # 查找残留的 python 进程
kill -9 <PID>
```

## 另请参见

- **[vLLM README](README.md)**：快速上手与功能概览
- **[参考指南](vllm-reference-guide.md)**：配置、参数与运维细节
- **[可观测性（Observability）](vllm-observability.md)**：指标与监控
- **[基准测试](../../benchmarks/benchmarking.md)**：性能基准测试工具
- **[解耦性能调优](../../performance/tuning.md)**：P/D 调优指南
