---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: SGLang
---

## 使用最新版本

我们建议使用 Dynamo 的[最新稳定发布版本](https://github.com/ai-dynamo/dynamo/releases/latest)以避免破坏性变更。

---

Dynamo SGLang 将 [SGLang](https://github.com/sgl-project/sglang) 引擎集成到 Dynamo 的分布式运行时中，支持分离式服务、KV 感知路由与请求取消，同时与 SGLang 原生引擎参数完全兼容。它支持 LLM 推理、嵌入模型、多模态视觉模型，以及基于扩散的生成（LLM、图像、视频）。

## 安装

### 安装最新发布版

我们建议使用 [uv](https://github.com/astral-sh/uv) 安装：

```bash
uv venv --python 3.12 --seed
uv pip install --prerelease=allow "ai-dynamo[sglang]"
```

这将以兼容版本的 SGLang 安装 Dynamo。

### 开发安装

<Accordion title="开发安装">
需要 Rust 和 CUDA 工具链（`nvcc`）。

```bash
# install dynamo
uv venv --python 3.12 --seed
uv pip install maturin nixl
cd $DYNAMO_HOME/lib/bindings/python
maturin develop --uv
cd $DYNAMO_HOME
uv pip install -e .
# install sglang
git clone https://github.com/sgl-project/sglang.git
cd sglang && uv pip install -e "python"
```

这是 agent 进行开发的理想方式。你可以为它提供两个仓库与虚拟环境的路径，并在它进行更改时让它重新执行这些命令。
</Accordion>

### Docker

<Accordion title="构建并运行容器">
```bash
cd $DYNAMO_ROOT
python container/render.py --framework sglang --output-short-filename
docker build -f container/rendered.Dockerfile -t dynamo:latest-sglang .
```

```bash
docker run \
    --gpus all -it --rm \
    --network host --shm-size=10G \
    --ulimit memlock=-1 --ulimit stack=67108864 \
    --ulimit nofile=65536:65536 \
    --cap-add CAP_SYS_PTRACE --ipc host \
    dynamo:latest-sglang
```
</Accordion>

## 特性支持矩阵

| 特性 | 状态 | 备注 |
|---------|--------|-------|
| [**分离式服务**](../../design-docs/disagg-serving.md) | ✅ | Prefill/decode 分离，使用 NIXL 进行 KV 传输 |
| [**KV 感知路由**](../../components/router/README.md) | ✅ | |
| [**SLA 驱动 Planner**](../../components/planner/planner-guide.md) | ✅ | |
| [**多模态支持**](../../features/multimodal/multimodal-sglang.md) | ✅ | 通过 EPD、E/PD、E/P/D 模式支持图像 |
| [**扩散模型**](sglang-diffusion.md) | ✅ | LLM 扩散、图像与视频生成 |
| [**请求取消**](../../fault-tolerance/request-cancellation.md) | ✅ | 聚合式完整支持；分离式仅 decode |
| [**优雅关闭**](../../fault-tolerance/graceful-shutdown.md) | ✅ | 发现注销 + 宽限期 |
| [**可观测性**](sglang-observability.md) | ✅ | 指标、追踪与 Grafana 仪表盘 |
| [**KVBM**](../../components/kvbm/README.md) | ❌ | 计划中 |

## 快速开始

### Python / CLI 部署

为本地开发启动基础设施服务：

```bash
docker compose -f deploy/docker-compose.yml up -d
```


启动一个聚合式服务部署：

```bash
cd $DYNAMO_HOME/examples/backends/sglang
./launch/agg.sh
```

验证部署：

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

可通过 `DynamoGraphDeployment` 在 Kubernetes 上部署带 Dynamo 的 SGLang。详情见 [SGLang Kubernetes 部署指南](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/sglang/deploy)。

## 后续步骤

- **[参考指南](sglang-reference-guide.md)**：worker 类型、架构与配置
- **[示例](sglang-examples.md)**：所有部署模式的启动脚本
- **[分离式](sglang-disaggregation.md)**：P/D 架构与 KV 传输细节
- **[扩散](sglang-diffusion.md)**：LLM、图像与视频扩散模型
- **[可观测性](sglang-observability.md)**：指标、追踪与 Grafana 仪表盘
- **[在 Kubernetes 上部署带 Dynamo 的 SGLang](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/sglang/deploy)**：Kubernetes 部署指南
