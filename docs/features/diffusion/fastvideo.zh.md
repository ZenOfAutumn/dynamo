---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
sidebar-title: FastVideo
---

# FastVideo

本指南介绍如何在 Dynamo 上部署 [FastVideo](https://github.com/hao-ai-lab/FastVideo) 文本到视频生成，使用一个自定义 worker（`worker.py`）通过 `/v1/videos` 端点暴露。

> [!NOTE]
> Dynamo 还通过内置后端支持 diffusion：[SGLang Diffusion](../../backends/sglang/sglang-diffusion.md)（LLM diffusion、图像、视频）、[vLLM-Omni](../../backends/vllm/vllm-omni.md)（文生图、文生视频）以及 [TRT-LLM Diffusion](../../backends/trtllm/trtllm-diffusion.md)（文生图、文生视频）。完整支持矩阵请参阅 [Diffusion 总览](README.md)。

## 概述

- **默认模型：** `FastVideo/LTX2-Distilled-Diffusers` —— LTX-2 Diffusion Transformer（Lightricks）的蒸馏变体，把推理（inference）从 50+ 步降到仅 5 步。
- **两阶段管线：** 第 1 阶段在目标分辨率生成视频；第 2 阶段使用蒸馏 LoRA 进一步提升保真度与纹理。
- **优化推理：** 通过 `--enable-optimizations` 可启用 FP4 量化与 `torch.compile`；attention 后端通过 `--attention-backend` 单独控制。
- **响应格式：** 每个请求返回一个完整的 MP4 负载，作为 `data[0].b64_json`（非流式）。
- **并发：** 每个 worker 一次只处理一个请求（VideoGenerator 不可重入）。通过运行多个 worker 来扩展吞吐。

> [!IMPORTANT]
> 为了在更多 GPU（包括 H100 等系统）上有更广的兼容性，`worker.py` 默认使用 `--attention-backend TORCH_SDPA`。在面向 B200/B300 的路径上，使用 `--enable-optimizations` 启用 FP4/compile，并按需通过 `--attention-backend FLASH_ATTN` 显式启用 flash-attention。

## Docker 镜像构建

本地 Docker 工作流会基于 [`Dockerfile`](https://github.com/ai-dynamo/dynamo/tree/main/examples/diffusers/Dockerfile) 构建运行时镜像：

- 基础镜像：`nvidia/cuda:13.1.1-devel-ubuntu24.04`
- 从 GitHub 安装 [FastVideo](https://github.com/hao-ai-lab/FastVideo)
- 从 `release/1.0.0` 分支安装 Dynamo（用于 `/v1/videos` 支持）
- 从源码编译 [flash-attention](https://github.com/RandNMR73/flash-attention) 的 fork

Dockerfile 暴露 `TORCH_CUDA_ARCH_LIST` 作为构建参数（默认 `10.0 10.0a`，对应 Blackwell）。要面向不同架构构建，请用 `--build-arg`：

```bash
# Blackwell (default)
docker build examples/diffusers/ --build-arg TORCH_CUDA_ARCH_LIST="10.0 10.0a"

# Hopper
docker build examples/diffusers/ --build-arg TORCH_CUDA_ARCH_LIST="9.0 9.0a"
```

`MAX_JOBS`（默认 `4`）控制 flash-attention 的并行编译任务数。如果构建因内存不足失败，请调小：

```bash
docker build examples/diffusers/ --build-arg MAX_JOBS=2
```

使用 Docker Compose 时，在执行 `docker compose up --build` 前把它们设为环境变量：

```bash
# Hopper on a memory-constrained builder
TORCH_CUDA_ARCH_LIST="9.0 9.0a" MAX_JOBS=2 COMPOSE_PROFILES=4 docker compose up --build
```

> [!WARNING]
> 第一次 Docker 镜像构建可能耗时 **20–40+ 分钟**，因为 FastVideo 与 CUDA 相关组件会在构建期被编译。如果 Docker 层缓存被保留，后续构建会快很多。编译 `flash-attention` 可能消耗大量 RAM——内存吃紧的构建机可能会 OOM。若发生这种情况，请在 Dockerfile 中调小 `MAX_JOBS` 以减少并行编译时的内存占用。[flash-attn 安装说明](https://pypi.org/project/flash-attn/) 在内存少于 96 GB 且 CPU 核数较多时也专门推荐这一做法。

## 预热时间

worker 第一次启动时会下载模型权重。启用 `--enable-optimizations` 时，编译/预热步骤可能让首次就绪时间长达约 **10–20 分钟**（取决于硬件）。第一次成功响应之后，第二次请求仍可能耗时约 **35 秒**，因为运行时缓存还在预热；从第三次请求开始通常可达到稳态性能。

> [!TIP]
> 在 Kubernetes 上，挂载共享 Hugging Face 缓存 PVC（参见 [Kubernetes 部署](#kubernetes-deployment)），这样模型权重只需下载一次，pod 重启后可以复用。

## 本地部署

### 前置条件

**对于 Docker Compose：**

- Docker Engine 26.0+
- Docker Compose v2
- NVIDIA Container Toolkit

**对于宿主机本地脚本：**

- 已安装 Dynamo + FastVideo 依赖的 Python 环境
- 宿主机上可用的 CUDA 兼容 GPU 运行时

### 选项 1：Docker Compose

```bash
cd <dynamo-root>/examples/diffusers/local

# Start 4 workers on GPUs 0..3
COMPOSE_PROFILES=4 docker compose up --build
```

Compose 文件基于 Dockerfile 构建，并将 API 暴露在 `http://localhost:8000`。构建时间预期请参见 [Docker 镜像构建](#docker-image-build) 一节。

### 选项 2：宿主机本地脚本

```bash
cd <dynamo-root>/examples/diffusers/local
./run_local.sh
```

环境变量：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `PYTHON_BIN` | `python3` | Python 解释器 |
| `MODEL` | `FastVideo/LTX2-Distilled-Diffusers` | HuggingFace 模型路径 |
| `NUM_GPUS` | `1` | GPU 数量 |
| `HTTP_PORT` | `8000` | 前端 HTTP 端口 |
| `WORKER_EXTRA_ARGS` | — | `worker.py` 的额外参数（例如 `--enable-optimizations --attention-backend FLASH_ATTN`） |
| `FRONTEND_EXTRA_ARGS` | — | `dynamo.frontend` 的额外参数 |

示例：

```bash
MODEL=FastVideo/LTX2-Distilled-Diffusers \
NUM_GPUS=1 \
HTTP_PORT=8000 \
WORKER_EXTRA_ARGS="--enable-optimizations --attention-backend FLASH_ATTN" \
./run_local.sh
```

> [!NOTE]
> `--enable-optimizations` 与 `--attention-backend` 是 `worker.py` 的参数，不是 `dynamo.frontend` 的参数；要使用非默认 worker 配置时通过 `WORKER_EXTRA_ARGS` 传递。

脚本把日志写到：

- `.runtime/logs/worker.log`
- `.runtime/logs/frontend.log`

## Kubernetes 部署

### 文件

| 文件 | 说明 |
|---|---|
| `agg.yaml` | 基础聚合（aggregated）部署（前端 + `FastVideoWorker`） |
| `agg_user_workload.yaml` | 同样的部署，附带 `user-workload` tolerations 和 `imagePullSecrets` |
| `huggingface-cache-pvc.yaml` | 用于模型权重的共享 HF 缓存 PVC |
| `dynamo-platform-values-user-workload.yaml` | 适用于带 `user-workload` taint 节点的可选 Helm values |

### 前置条件

1. 已安装 Dynamo Kubernetes Platform
2. 启用了 GPU 的 Kubernetes 集群
3. FastVideo runtime 镜像已推送到你的镜像仓库
4. 可选 HF token secret（针对受限模型）

如果需要，创建 Hugging Face token secret：

```bash
export NAMESPACE=<your-namespace>
export HF_TOKEN=<your-hf-token>
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n ${NAMESPACE}
```

### 部署

```bash
cd <dynamo-root>/examples/diffusers/deploy
export NAMESPACE=<your-namespace>

kubectl apply -f huggingface-cache-pvc.yaml -n ${NAMESPACE}
kubectl apply -f agg.yaml -n ${NAMESPACE}
```

对于带 `user-workload` taint 的集群以及私有镜像仓库拉取：

1. 在 `agg_user_workload.yaml` 中设置你的 pull secret 名称与镜像。
2. 应用：

```bash
kubectl apply -f huggingface-cache-pvc.yaml -n ${NAMESPACE}
kubectl apply -f agg_user_workload.yaml -n ${NAMESPACE}
```

### 快速更新镜像

```bash
export DEPLOYMENT_FILE=agg.yaml
export FASTVIDEO_IMAGE=<my-registry/fastvideo-runtime:my-tag>

yq '.spec.services.[].extraPodSpec.mainContainer.image = env(FASTVIDEO_IMAGE)' \
  ${DEPLOYMENT_FILE} > ${DEPLOYMENT_FILE}.generated

kubectl apply -f ${DEPLOYMENT_FILE}.generated -n ${NAMESPACE}
```

### 验证与访问

```bash
kubectl get dgd -n ${NAMESPACE}
kubectl get pods -n ${NAMESPACE}
kubectl logs -n ${NAMESPACE} -l nvidia.com/dynamo-component=FastVideoWorker
```

```bash
kubectl port-forward -n ${NAMESPACE} svc/fastvideo-agg-frontend 8000:8000
```

## 测试请求

> [!NOTE]
> 如果是启动后第一次请求，预期会因预热完成而耗时较长。详见 [预热时间](#warmup-time)。

发送一次请求并解码响应：

```bash
curl -s -X POST http://localhost:8000/v1/videos \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "FastVideo/LTX2-Distilled-Diffusers",
    "prompt": "A cinematic drone shot over a snowy mountain range at sunrise",
    "size": "1920x1088",
    "seconds": 5,
    "nvext": {
      "fps": 24,
      "num_frames": 121,
      "num_inference_steps": 5,
      "guidance_scale": 1.0,
      "seed": 10
    }
  }' > response.json

# Linux
jq -r '.data[0].b64_json' response.json | base64 --decode > output.mp4

# macOS
jq -r '.data[0].b64_json' response.json | base64 -D > output.mp4
```

## Worker 配置参考

### CLI 参数

| 参数 | 默认值 | 说明 |
|---|---|---|
| `--model` | `FastVideo/LTX2-Distilled-Diffusers` | HuggingFace 模型路径 |
| `--num-gpus` | `1` | 用于分布式推理的 GPU 数量 |
| `--enable-optimizations` | off | 启用 FP4 量化与 `torch.compile` |
| `--attention-backend` | `TORCH_SDPA` | 设置 `FASTVIDEO_ATTENTION_BACKEND`；可选项：`FLASH_ATTN`、`TORCH_SDPA`、`SAGE_ATTN`、`SAGE_ATTN_THREE`、`VIDEO_SPARSE_ATTN`、`VMOBA_ATTN`、`SLA_ATTN`、`SAGE_SLA_ATTN` |

### 请求参数（`nvext`）

| 字段 | 默认值 | 说明 |
|---|---|---|
| `fps` | `24` | 帧率 |
| `num_frames` | `121` | 总帧数；设置时会覆盖 `fps * seconds` |
| `num_inference_steps` | `5` | Diffusion 推理步数 |
| `guidance_scale` | `1.0` | Classifier-free guidance 系数 |
| `seed` | `10` | 用于复现的随机种子 |
| `negative_prompt` | — | 生成中要避免的文本 |

### 环境变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `FASTVIDEO_VIDEO_CODEC` | `libx264` | MP4 编码使用的视频编解码器 |
| `FASTVIDEO_X264_PRESET` | `ultrafast` | x264 编码速度预设 |
| `FASTVIDEO_ATTENTION_BACKEND` | `TORCH_SDPA` | attention 后端；`worker.py` 会根据 `--attention-backend` 设置该值，并校验 `FLASH_ATTN`、`TORCH_SDPA`、`SAGE_ATTN`、`SAGE_ATTN_THREE`、`VIDEO_SPARSE_ATTN`、`VMOBA_ATTN`、`SLA_ATTN` 与 `SAGE_SLA_ATTN` |
| `FASTVIDEO_STAGE_LOGGING` | `1` | 启用按阶段的耗时日志 |
| `FASTVIDEO_LOG_LEVEL` | — | 设为 `DEBUG` 可获得详细日志 |

## 故障排查

| 现象 | 原因 | 解决 |
|---|---|---|
| Docker 构建期间 OOM | `flash-attention` 编译占用过多 RAM | 在构建时传入 `--build-arg MAX_JOBS=2`（或更小） |
| 运行时报 `no kernel image available for this GPU` 或 CUDA 架构错误 | 镜像是为不同 GPU 架构构建的 | 用正确的 `TORCH_CUDA_ARCH_LIST` 重建（例如 Hopper 用 `9.0 9.0a`） |
| 启用优化后首次启动等待 10–20 分钟 | 模型下载 + `torch.compile` 预热 | 这是预期行为；权重缓存命中后后续启动更快 |
| 第二次请求约 35 秒 | 运行时缓存还在预热 | 从第三次请求开始进入稳态性能 |
| B200/B300 上的吞吐低于预期 | FP4/compile 与 flash-attention 是分别配置的 | 传入 `--enable-optimizations`，并按需 `--attention-backend FLASH_ATTN` |
| 启用优化或更换 attention 后端后启动/导入失败 | FP4 与某些 attention 后端依赖特定的硬件/软件支持 | 不带 `--enable-optimizations` 重新运行 `worker.py`，或使用 `--attention-backend TORCH_SDPA` |

## 源代码

示例源码位于 Dynamo 仓库的 [`examples/diffusers/`](https://github.com/ai-dynamo/dynamo/tree/main/examples/diffusers)。

## 另见

- [vLLM-Omni Text-to-Video](../../backends/vllm/vllm-omni.md#text-to-video) —— vLLM-Omni 通过 `/v1/videos` 的视频生成
- [vLLM-Omni Text-to-Image](../../backends/vllm/vllm-omni.md#text-to-image) —— vLLM-Omni 图像生成
- [SGLang Video Generation](../../backends/sglang/sglang-diffusion.md#video-generation) —— SGLang 视频生成 worker
- [SGLang Image Diffusion](../../backends/sglang/sglang-diffusion.md#image-diffusion) —— SGLang 图像 diffusion worker
- [TRT-LLM Diffusion](../../backends/trtllm/trtllm-diffusion.md#quick-start) —— TensorRT-LLM diffusion 快速开始
- [Diffusion 总览](README.md) —— 完整的后端支持矩阵
