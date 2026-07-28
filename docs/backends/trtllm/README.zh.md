---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: TensorRT-LLM
---

## 使用最新版本

我们建议使用 Dynamo 的[最新稳定发布版本](https://github.com/ai-dynamo/dynamo/releases/latest)以避免 breaking 变更。

---

Dynamo TensorRT-LLM 将 [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM) 引擎集成到 Dynamo 的分布式运行时中，支持分离式服务、KV 感知路由、多节点部署以及请求取消。它支持 LLM 推理、多模态模型、视频扩散，以及推测解码、注意力数据并行等高级特性。

## 特性支持矩阵

### 核心 Dynamo 特性

| 特性 | TensorRT-LLM | 备注 |
|---------|--------------|-------|
| [**分离式服务**](../../design-docs/disagg-serving.md) | ✅ |  |
| [**条件式分离**](../../design-docs/disagg-serving.md) | 🚧 | 暂未支持 |
| [**KV 感知路由**](../../components/router/README.md) | ✅ |  |
| [**SLA 驱动 Planner**](../../components/planner/planner-guide.md) | ✅ |  |
| [**基于负载的 Planner**](../../components/planner/README.md) | 🚧 | 计划中 |
| [**KVBM**](../../components/kvbm/README.md) | ✅ | |

### 大规模 P/D 与 WideEP 特性

| 特性            | TensorRT-LLM | 备注                                                           |
|--------------------|--------------|-----------------------------------------------------------------|
| **WideEP**         | ✅           |                                                                 |
| **DP Rank Routing**| ✅           |                                                                 |
| **GB200 Support**  | ✅           |                                                                 |

## 快速开始

**步骤 1（宿主机终端）：** 启动基础设施服务：

```bash
docker compose -f deploy/docker-compose.yml up -d
```

**步骤 2（宿主机终端）：** 拉取并运行预构建容器：

```bash
DYNAMO_VERSION=1.0.0
docker pull nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:$DYNAMO_VERSION
docker run --gpus all -it --network host --ipc host \
  nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:$DYNAMO_VERSION
```

> [!NOTE]
> 上述 `DYNAMO_VERSION` 变量可设置为该容器的任意可用版本。
> 要查找 Dynamo 可用的 `tensorrtllm-runtime` 版本，请访问 [NVIDIA NGC Catalog for Dynamo TensorRT-LLM Runtime](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/tensorrtllm-runtime)。

**步骤 3（容器内）：** 启动一个聚合式服务部署（默认使用 `Qwen/Qwen3-0.6B`）：

```bash
cd $DYNAMO_HOME/examples/backends/trtllm
./launch/agg.sh
```

启动脚本将自动下载模型并启动 TensorRT-LLM 引擎。你可以在运行脚本前通过设置 `MODEL_PATH` 与 `SERVED_MODEL_NAME` 环境变量来覆盖所用模型。

**步骤 4（宿主机终端）：** 验证部署：

```bash
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [{"role": "user", "content": "Explain why Roger Federer is considered one of the greatest tennis players of all time"}],
    "stream": true,
    "max_tokens": 30
  }'
```

### Kubernetes 部署

你可以在 Kubernetes 上通过 `DynamoGraphDeployment` 部署带 Dynamo 的 TensorRT-LLM。详情请见 [TensorRT-LLM Kubernetes 部署指南](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/trtllm/deploy/README.md)。

## 后续步骤

- **[参考指南](trtllm-reference-guide.md)**：特性、配置与运维细节
- **[示例](trtllm-examples.md)**：所有部署模式与启动脚本
- **[KV 缓存传输](trtllm-kv-cache-transfer.md)**：分离式服务下的 KV 缓存传输方法
- **[可观测性](trtllm-observability.md)**：指标与监控
- **[多节点示例](multinode/trtllm-multinode-examples.md)**：使用 SLURM 的多节点部署
- **[在 Kubernetes 上部署 TensorRT-LLM 与 Dynamo](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/trtllm/deploy/README.md)**：Kubernetes 部署指南
