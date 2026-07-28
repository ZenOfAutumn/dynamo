---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
sidebar-title: 本地安装
description: 在本地机器或虚拟机上通过容器或 PyPI 安装并运行 Dynamo
---

# 本地安装

本指南介绍在本地机器或带有一块及以上 GPU 的虚拟机上安装并运行 Dynamo。读完本文后，你将获得一个可工作的 OpenAI 兼容 endpoint，并由它对外提供模型服务。

如果是生产级的多节点集群，请参考 [Kubernetes 部署指南](../kubernetes/README.md)。如需从源码构建用于开发，请参考 [从源码构建](building-from-source.md)。

## 系统要求

| 项 | 受支持范围 |
|---|---|
| **GPU** | NVIDIA Ampere、Ada Lovelace、Hopper、Blackwell |
| **OS** | Ubuntu 22.04、Ubuntu 24.04 |
| **架构** | x86_64、ARM64（ARM64 需要 Ubuntu 24.04） |
| **CUDA** | 12.9+ 或 13.0+（B300/GB300 需要 CUDA 13） |
| **Python** | 3.10、3.12 |
| **驱动** | 575.51.03+（CUDA 12）或 580.00.03+（CUDA 13） |

TensorRT-LLM 不支持 Python 3.11。

完整的兼容性矩阵（含各后端框架版本）请参见 [Support Matrix](../reference/support-matrix.md)。

## 安装 Dynamo

### 方式 A：容器（推荐）

容器已预装所有依赖，无需额外配置。

```bash
# SGLang
docker run --gpus all --network host --rm -it nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.1.1

# TensorRT-LLM
docker run --gpus all --network host --rm -it nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:1.1.1

# vLLM
docker run --gpus all --network host --rm -it nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
```

如果想在同一个容器内同时运行 frontend 与 worker，可以：

- 用 `&` 把进程放到后台运行（参见下文 "运行 Dynamo" 一节），或
- 另开一个终端，使用 `docker exec -it <container_id> bash`

可用版本请参见 [Release Artifacts](../reference/release-artifacts.md#container-images)；运行说明请参考各后端指南：
[SGLang](../backends/sglang/README.md) | [TensorRT-LLM](../backends/trtllm/README.md) | [vLLM](../backends/vllm/README.md)

### 方式 B：通过 PyPI 安装

```bash
# 安装 uv（推荐的 Python 包管理器）
curl -LsSf https://astral.sh/uv/install.sh | sh

# 创建虚拟环境
uv venv venv
source venv/bin/activate
uv pip install pip
```

安装系统依赖以及与你所选后端对应的 Dynamo wheel：

**SGLang**

```bash
sudo apt install python3-dev
uv pip install --prerelease=allow "ai-dynamo[sglang]"
```

对于 CUDA 13（B300/GB300），推荐使用容器。详见 [SGLang 安装文档](https://docs.sglang.io/get_started/install.html)。

**TensorRT-LLM**

```bash
sudo apt install python3-dev
pip install torch==2.9.0 torchvision --index-url https://download.pytorch.org/whl/cu130
pip install --pre --extra-index-url https://pypi.nvidia.com "ai-dynamo[trtllm]"
```

TensorRT-LLM 必须使用 `pip`，因为它有一个传递依赖来自 Git URL，`uv` 当前无法解析。为获得更好的兼容性，建议使用 TensorRT-LLM 容器。详见 [TRT-LLM 后端指南](../backends/trtllm/README.md)。

**vLLM**

```bash
sudo apt install python3-dev libxcb1
uv pip install --prerelease=allow "ai-dynamo[vllm]"
```

## 运行 Dynamo

### 服务发现后端（Discovery Backend）

Dynamo 各组件通过共享的 discovery backend 互相发现，目前提供两种选择：

| Backend | 适用场景 | 配置方式 |
|---|---|---|
| **File** | 单机本地开发 | 无需配置 —— 给所有组件传 `--discovery-backend file` 即可。event plane 会自动回退为 ZMQ（无需 NATS） |
| **etcd** | 多节点、生产环境 | 需要一个运行中的 etcd 实例（不指定参数时即默认）。event plane 默认使用 NATS |

本指南使用 `--discovery-backend file`。etcd 的搭建请参见 [服务发现](../kubernetes/service-discovery.md)。

### 验证安装（可选）

确认 CLI 已正确安装并可调用：

```bash
python3 -m dynamo.frontend --help
```

如果你已克隆仓库，还可以运行更完整的系统检查脚本：

```bash
python3 deploy/sanity_check.py
```

### 启动 Frontend

```bash
# 启动 OpenAI 兼容的 frontend（默认端口 8000）
python3 -m dynamo.frontend --discovery-backend file
```

如果想在单个终端里运行（在容器内尤其有用），追加 `> logfile.log 2>&1 &` 把进程放到后台：

```bash
python3 -m dynamo.frontend --discovery-backend file > dynamo.frontend.log 2>&1 &
```

### 启动一个 Worker

另开一个终端（如果用了后台模式也可以同终端），按所选后端启动 worker：

**SGLang**

```bash
python3 -m dynamo.sglang --model-path Qwen/Qwen3-0.6B --discovery-backend file
```

**TensorRT-LLM**

```bash
python3 -m dynamo.trtllm --model-path Qwen/Qwen3-0.6B --discovery-backend file
```

`Cannot connect to ModelExpress server/transport error. Using direct download.` 这条警告在本地部署中是预期行为，可以安全忽略。

**vLLM**

```bash
python3 -m dynamo.vllm --model Qwen/Qwen3-0.6B --discovery-backend file \
  --kv-events-config '{"enable_kv_cache_events": false}'
```

### KV Events 配置

如果想在本地开发时不依赖 NATS，可以关闭 KV 事件发布：

- **vLLM：** 加上 `--kv-events-config '{"enable_kv_cache_events": false}'`
- **SGLang：** 无需任何参数（KV 事件默认就是关闭的）
- **TensorRT-LLM：** 无需任何参数（KV 事件默认就是关闭的）

所有后端的 KV 事件**默认都是关闭的**，只有当你想启用 KV 事件发布时才需要显式加上 `--kv-events-config`。

## 测试你的部署

```bash
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-0.6B",
       "messages": [{"role": "user", "content": "Hello!"}],
       "max_tokens": 50}'
```

## 故障排查

**CUDA / 驱动版本不匹配**

执行 `nvidia-smi` 查看当前驱动版本。Dynamo 要求驱动 575.51.03+（CUDA 12）或 580.00.03+（CUDA 13）。B300/GB300 需要 CUDA 13。完整要求详见 [Support Matrix](../reference/support-matrix.md)。

**模型放不下 GPU（OOM）**

默认模型 `Qwen/Qwen3-0.6B` 大约需要 2GB 显存。更大的模型需要更多 VRAM：

| 模型规模 | 大致 VRAM |
|---|---|
| 7B | 14–16 GB |
| 13B | 26–28 GB |
| 70B | 140+ GB（多卡） |

建议先用小模型起步，再根据硬件能力逐步放大。

**TensorRT-LLM 配 Python 3.11**

TensorRT-LLM 不支持 Python 3.11。如果安装 TensorRT-LLM 时报错，先用 `python3 --version` 查看版本。请改用 Python 3.10 或 3.12。

**容器跑起来了，但检测不到 GPU**

确认 `docker run` 时加了 `--gpus all`。不加这个参数，容器无法访问 GPU：

```bash
# 正确
docker run --gpus all --network host --rm -it nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.1.1

# 错误 —— 没有 GPU 访问权限
docker run --network host --rm -it nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.1.1
```

## 下一步

- [后端指南](../backends/sglang/README.md) —— 各后端的配置与特性
- [Disaggregated Serving](../features/disaggregated-serving/README.md) —— prefill 与 decode 独立扩缩容
- [KV Cache 感知路由](../components/router/router-guide.md) —— 智能的请求路由
- [Kubernetes 部署](../kubernetes/README.md) —— 生产级多节点部署

