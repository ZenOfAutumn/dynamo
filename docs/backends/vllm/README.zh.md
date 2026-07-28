---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: vLLM
---

# 使用 vLLM 进行 LLM 部署

Dynamo vLLM 将 [vLLM](https://github.com/vllm-project/vllm) 引擎集成到 Dynamo 的分布式运行时中，支持分离式服务、KV 感知路由与请求取消，同时与 vLLM 原生引擎参数保持完整兼容。Dynamo 利用 vLLM 原生的 KV 缓存事件、基于 NIXL 的传输机制以及指标上报，实现 KV 感知路由与 P/D 分离。

## 安装

### 安装最新发布版

我们建议使用 [uv](https://github.com/astral-sh/uv) 安装：

```bash
uv venv --python 3.12 --seed
uv pip install "ai-dynamo[vllm]"
```

这将以兼容版本的 vLLM 安装 Dynamo。

---

### 容器

公共镜像可在 [NGC Catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/collections/ai-dynamo/artifacts) 获取：

```bash
docker pull nvcr.io/nvidia/ai-dynamo/vllm-runtime:<version>
./container/run.sh -it --framework VLLM --image nvcr.io/nvidia/ai-dynamo/vllm-runtime:<version>
```

<Accordion title="从源码构建">

```bash
python container/render.py --framework vllm --output-short-filename
docker build -f container/rendered.Dockerfile -t dynamo:latest-vllm .
```

```bash
./container/run.sh -it --framework VLLM [--mount-workspace]
```

</Accordion>

### 开发环境

进行开发时，建议使用预安装了所有依赖的 [devcontainer](https://github.com/ai-dynamo/dynamo/tree/main/.devcontainer)。

## 特性支持矩阵

| 特性 | 状态 | 备注 |
|---------|--------|-------|
| [**分离式服务**](../../design-docs/disagg-serving.md) | ✅ | Prefill/decode 分离，使用 NIXL 进行 KV 传输 |
| [**KV 感知路由**](../../components/router/README.md) | ✅ | |
| [**SLA 驱动 Planner**](../../components/planner/planner-guide.md) | ✅ | |
| [**KVBM**](../../components/kvbm/README.md) | ✅ | |
| [**LMCache**](../../integrations/lmcache-integration.md) | ✅ | |
| [**FlexKV**](../../integrations/flexkv-integration.md) | ✅ | |
| [**多模态支持**](vllm-omni.md) | ✅ | 通过 vLLM-Omni 集成 |
| [**可观测性**](vllm-observability.md) | ✅ | 指标与监控 |
| **WideEP** | ✅ | 支持 DeepEP |
| **DP Rank Routing** | ✅ | 通过外部 DP rank 控制实现的[混合负载均衡](https://docs.vllm.ai/en/stable/serving/data_parallel_deployment/?h=external+dp#hybrid-load-balancing) |
| [**LoRA**](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/launch/lora/README.md) | ✅ | 从 S3 兼容存储动态加载/卸载 |
| **GB200 Support** | ✅ | 主分支上的容器可正常工作 |

## 快速开始

为本地开发启动基础设施服务：

```bash
docker compose -f deploy/docker-compose.yml up -d
```

启动一个聚合式服务部署：

```bash
cd $DYNAMO_HOME/examples/backends/vllm
bash launch/agg.sh
```

## 后续步骤

- **[参考指南](vllm-reference-guide.md)**：配置、参数与运维细节
- **[示例](vllm-examples.md)**：所有部署模式与启动脚本
- **[KV 缓存卸载](vllm-kv-offloading.md)**：KVBM、LMCache 与 FlexKV 集成
- **[可观测性](vllm-observability.md)**：指标与监控
- **[vLLM-Omni](vllm-omni.md)**：多模态模型服务
- **[Kubernetes 部署](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/deploy/README.md)**：Kubernetes 部署指南
- **[vLLM 文档](https://docs.vllm.ai/en/stable/)**：上游 vLLM serve 参数
