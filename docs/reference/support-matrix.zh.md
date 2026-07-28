---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Support Matrix
subtitle: Hardware, software, and build compatibility for Dynamo
---

**另请参阅：** [发布产物](release-artifacts.md)（容器镜像、wheel、Helm chart、crate）| [特性矩阵](feature-matrix.md)（后端特性支持）

## 概览

**最新稳定发布：** [v1.1.1](https://github.com/ai-dynamo/dynamo/releases/tag/v1.1.1) -- SGLang `0.5.10.post1` (NIXL `1.0.1`) | TensorRT-LLM `1.3.0rc11` (NIXL `0.10.1`) | vLLM `0.19.0` (NIXL `0.10.1`)

**实验性发布：** [v1.2.0-deepseek-v4-dev.3](https://github.com/ai-dynamo/dynamo/releases/tag/v1.2.0-deepseek-v4-dev.3) *(Blackwell 上的 DeepSeek-V4-Flash / V4-Pro，仅 vLLM + SGLang 容器)* -- vLLM `0.20.1` | SGLang 上游 `deepseek-v4-blackwell` 预览版 | NIXL `0.10.1`

| 要求 | 支持 |
| :--- | :--- |
| **GPU** | NVIDIA Ampere、Ada Lovelace、Hopper、Blackwell |
| **OS** | Ubuntu 22.04、Ubuntu 24.04、CentOS Stream 9（实验性）|
| **架构** | x86_64、ARM64（ARM64 需 Ubuntu 24.04）|
| **CUDA 12** | SGLang 与 vLLM 的容器镜像（CUDA 12.9）|
| **CUDA 13** | TensorRT-LLM 的容器镜像（CUDA 13.1）；SGLang 与 vLLM 的容器镜像（CUDA 13.0）|

**本页内容：** [后端依赖](#后端依赖) | [CUDA 与驱动](#cuda-与驱动要求) | [硬件](#硬件兼容性) | [平台](#平台架构兼容性) | [云](#云服务提供商兼容性) | [构建支持](#构建支持)

## 后端依赖

> 各后端的驱动要求不同 —— 详见下文 [CUDA 与驱动要求](#cuda-与驱动要求)。

下表列出每个 Dynamo 发布版本所携带的后端框架版本：

| **Dynamo** | **SGLang** | **TensorRT-LLM** | **vLLM** | **NIXL** |
| :--- | :--- | :--- | :--- | :--- |
| **main (ToT)** | `0.5.10.post1` | `1.3.0rc14` | `0.20.1` | `0.10.1`（TRT-LLM、vLLM）；`1.0.1`（SGLang）|
| **v1.2.0-deepseek-v4-dev.3** *（实验性，部分）* | 上游 DSv4 预览 | — | `0.20.1` | `0.10.1` |
| **v1.2.0-deepseek-v4-dev.2** *（实验性，部分）* | 上游 DSv4 预览 | — | `0.20.0` | `0.10.1` |
| **v1.1.1** | `0.5.10.post1` | `1.3.0rc11` | `0.19.0` | `0.10.1`（TRT-LLM、vLLM）；`1.0.1`（SGLang）|
| **v1.1.0** | `0.5.10.post1` | `1.3.0rc11` | `0.19.0` | `0.10.1`（TRT-LLM、vLLM）；`1.0.1`（SGLang）|
| **v1.1.0-dev.3** *（实验性，部分）* | `0.5.10.post1` | `1.3.0rc11` | `0.19.0` | `0.10.1` |
| **v1.1.0-dev.2** *（实验性，部分）* | `0.5.9` | `1.3.0rc9` | `0.19.0` | `0.10.1` |
| **v1.1.0-dev.1** *（实验性）* | `0.5.9` | `1.3.0rc5.post1` | `0.17.1` | `0.10.1` |
| **v1.0.2** | `0.5.9` | `1.3.0rc5.post1` | `0.16.0` | `0.10.1` |
| **v1.0.1** | `0.5.9` | `1.3.0rc5.post1` | `0.16.0` | `0.10.1` |
| **v1.0.0** | `0.5.9` | `1.3.0rc5.post1` | `0.16.0` | `0.10.1` |
| **v0.9.1** | `0.5.8` | `1.3.0rc3` | `0.14.1` | `0.9.0` |
| **v0.9.0** | `0.5.8` | `1.3.0rc1` | `0.14.1` | `0.9.0` |
| **v0.8.1.post3** | `0.5.6.post2` | `1.2.0rc6.post3` | `0.12.0` | `0.8.0` |
| **v0.8.1.post2** | `0.5.6.post2` | `1.2.0rc6.post2` | `0.12.0` | `0.8.0` |
| **v0.8.1.post1** | `0.5.6.post2` | `1.2.0rc6.post1` | `0.12.0` | `0.8.0` |
| **v0.8.1** | `0.5.6.post2` | `1.2.0rc6.post1` | `0.12.0` | `0.8.0` |
| **v0.8.0** | `0.5.6.post2` | `1.2.0rc6.post1` | `0.12.0` | `0.8.0` |
| **v0.7.1** | `0.5.4.post3` | `1.2.0rc3` | `0.11.0` | `0.8.0` |
| **v0.7.0.post1** | `0.5.4.post3` | `1.2.0rc3` | `0.11.0` | `0.8.0` |
| **v0.7.0** | `0.5.4.post3` | `1.2.0rc2` | `0.11.0` | `0.8.0` |
| **v0.6.1.post1** | `0.5.3.post2` | `1.1.0rc5` | `0.11.0` | `0.6.0` |
| **v0.6.1** | `0.5.3.post2` | `1.1.0rc5` | `0.11.0` | `0.6.0` |
| **v0.6.0** | `0.5.3.post2` | `1.1.0rc5` | `0.11.0` | `0.6.0` |

对于 **v1.1.0-dev.2**、**v1.1.0-dev.3**、**v1.2.0-deepseek-v4-dev.2** 与 **v1.2.0-deepseek-v4-dev.3**，上表中的内容与对应发布分支上的 `container/context.yaml` 一致（用于构建镜像的 pin 版本）。这些是**部分发布**：并非每个后端都有针对该 tag 发布的 Dynamo 运行时容器。实际发布产物请见[预发布产物](release-artifacts.md#pre-release-artifacts)。`v1.2.0-deepseek-v4-dev.2` 与 `v1.2.0-deepseek-v4-dev.3` 的 SGLang 容器基于上游 `lmsysorg/sglang:deepseek-v4-blackwell` 预览镜像构建，而非某个标记的 SGLang 发布；TensorRT-LLM 不在这些 dev 版本中。

### 版本标识

- **main (ToT)** 反映当前开发分支的状态。
- 标注为*（实验性，部分）* 的发布是预发布：表中的版本是分支构建时的 pin，可能包含尚无对应 dev tag 的 NGC 镜像的后端。
- 标注为*（进行中）* 或*（计划中）* 的发布展示的是目标版本，最终发布前可能变化。

### 版本兼容性

- 表中列出的后端版本是每个发布版本测试与支持的唯一版本。
- TensorRT-LLM 不支持 Python 3.11；在 Python 3.11 上安装 `ai-dynamo[trtllm]` wheel 会失败。

### CUDA 与驱动要求

Dynamo 容器镜像内置 CUDA 工具包库。宿主机必须安装兼容的 NVIDIA GPU 驱动。

| Dynamo 版本 | 后端 | CUDA 工具包 | 最低驱动 | 备注 |
| :--- | :--- | :--- | :--- | :--- |
| **1.1.1** | **SGLang** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | |
| | **TensorRT-LLM** | 13.1 | 580.xx+ | |
| | **vLLM** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | |
| **1.1.0** | **SGLang** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | |
| | **TensorRT-LLM** | 13.1 | 580.xx+ | |
| | **vLLM** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | |
| **1.0.2** | **SGLang** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | |
| | **TensorRT-LLM** | 13.1 | 580.xx+ | |
| | **vLLM** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | |
| **1.0.1** | **SGLang** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | |
| | **TensorRT-LLM** | 13.1 | 580.xx+ | |
| | **vLLM** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | |
| **1.0.0** | **SGLang** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | |
| | **TensorRT-LLM** | 13.1 | 580.xx+ | |
| | **vLLM** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | |
| **0.9.1** | **SGLang** | 12.9 | 575.xx+ | |
| | **TensorRT-LLM** | 13.0 | 580.xx+ | |
| | **vLLM** | 12.9 | 575.xx+ | |
| **0.9.0** | **SGLang** | 12.9 | 575.xx+ | |
| | **TensorRT-LLM** | 13.0 | 580.xx+ | |
| | **vLLM** | 12.9 | 575.xx+ | |
| **0.8.1** | **SGLang** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | 实验性 |
| | **TensorRT-LLM** | 13.0 | 580.xx+ | |
| | **vLLM** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | 实验性 |
| **0.8.0** | **SGLang** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | 实验性 |
| | **TensorRT-LLM** | 13.0 | 580.xx+ | |
| | **vLLM** | 12.9 | 575.xx+ | |
| | | 13.0 | 580.xx+ | 实验性 |
| **0.7.1** | **SGLang** | 12.8 | 570.xx+ | |
| | **TensorRT-LLM** | 13.0 | 580.xx+ | |
| | **vLLM** | 12.9 | 575.xx+ | |
| **0.7.0** | **SGLang** | 12.9 | 575.xx+ | |
| | **TensorRT-LLM** | 13.0 | 580.xx+ | |
| | **vLLM** | 12.8 | 570.xx+ | |

补丁版本（如 v0.8.1.post1、v0.7.0.post1）的 CUDA 支持与其基础版本相同。

实验性的 `v1.1.0-dev.*` 镜像采用与 `v1.0.2` 相同的 CUDA 矩阵。`v1.2.0-deepseek-v4-dev.3` 的 vLLM 容器是 CUDA 13.0 多架构；SGLang 容器按架构区分（`amd64` 用 CUDA 12.9，`arm64` 用 CUDA 13.0）。

并非所有版本都发布实验性 CUDA 13 镜像。可用性请查阅[发布产物](release-artifacts.md)。

具体产物版本与 NGC 链接（包括容器镜像、Python wheel、Helm chart 与 Rust crate）请参阅[发布产物](release-artifacts.md)。

#### CUDA 兼容性资源

CUDA 驱动兼容性、向前兼容性与故障排查的详细信息：

- [CUDA 兼容性概览](https://docs.nvidia.com/deploy/cuda-compatibility/)
- [Why CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/why-cuda-compatibility.html)
- [Minor Version Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/minor-version-compatibility.html)
- [Forward Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/forward-compatibility.html)
- [FAQ](https://docs.nvidia.com/deploy/cuda-compatibility/frequently-asked-questions.html)

如需在最低驱动版本之外扩展兼容性，可考虑在宿主机使用 `cuda-compat` 包。详见 [Forward Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/forward-compatibility.html)。

## 硬件兼容性

| **CPU 架构** | **状态** |
| :------------------- | :----------- |
| **x86_64**           | 支持    |
| **ARM64**            | 支持    |

Dynamo 提供同时支持 AMD64（x86_64）与 ARM64 架构的多架构容器镜像。可用镜像参见[发布产物](release-artifacts.md)。

### GPU 兼容性

如果你使用 **GPU**，支持以下 GPU 型号与架构：

| **GPU 架构**                 | **状态** |
| :----------------------------------- | :--------- |
| **NVIDIA Blackwell 架构**    | 支持  |
| **NVIDIA Hopper 架构**       | 支持  |
| **NVIDIA Ada Lovelace 架构** | 支持  |
| **NVIDIA Ampere 架构**       | 支持  |

## 平台架构兼容性

**Dynamo** 兼容以下平台：

| **操作系统** | **版本** | **架构** | **状态**   |
| :------------------- | :---------- | :--------------- | :----------- |
| **Ubuntu**           | 22.04       | x86_64           | 支持    |
| **Ubuntu**           | 24.04       | x86_64           | 支持    |
| **Ubuntu**           | 24.04       | ARM64            | 支持    |
| **CentOS Stream**    | 9           | x86_64           | 实验性 |

Wheel 在 manylinux_2_28 兼容的环境中构建，并在 CentOS Stream 9 与 Ubuntu（22.04、24.04）上验证。其它 Linux 发行版的兼容性预期可用，但未经官方验证。

## 云服务提供商兼容性

### AWS

| **宿主机操作系统** | **版本** | **架构** | **状态** |
| :------------------------ | :---------- | :--------------- | :--------- |
| **Amazon Linux**          | 2023        | x86_64           | 支持  |

> [!Caution]
> **AL2023 TensorRT-LLM 限制：** 在本地用 `docker run --network host ...` 运行 AL2023 容器时，由于 mpi4py 的[一个 bug](https://github.com/mpi4py/mpi4py/discussions/491#discussioncomment-12660609)，TensorRT-LLM 框架存在已知问题。规避方法是用更精确的网络配置代替 `--network host`，仅映射必要端口（如 NATS 4222、etcd 2379/2380、frontend 8000）。

## 构建支持

各版本的产物详情、安装命令与发布历史，请参见[发布产物](release-artifacts.md)。

**Dynamo** 当前提供以下构建支持方式：

- **Wheel**：发布 Dynamo 与 KV Block Manager 的 Python wheel：
  - [ai-dynamo](https://pypi.org/project/ai-dynamo/)
  - [ai-dynamo-runtime](https://pypi.org/project/ai-dynamo-runtime/)
  - [kvbm](https://pypi.org/project/kvbm/) —— 独立实现。

- **Dynamo 容器镜像**：在 [NGC](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/collections/ai-dynamo) 上发布多架构镜像（同时兼容 x86 与 ARM64）：
  - [Dynamo Frontend](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/dynamo-frontend) *(v0.8.0 新增)*
  - [SGLang Runtime](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/sglang-runtime)
  - [SGLang Runtime (CUDA 13)](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/sglang-runtime-cu13)
  - [TensorRT-LLM Runtime](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/tensorrtllm-runtime)
  - [TensorRT-LLM Runtime (EFA)](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/tensorrtllm-runtime) *(v1.0.0 新增，实验性，仅 AMD64)*
  - [vLLM Runtime](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/vllm-runtime)
  - [vLLM Runtime (CUDA 13)](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/vllm-runtime-cu13)
  - [vLLM Runtime (EFA)](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/vllm-runtime) *(v1.0.0 新增，实验性，仅 AMD64)*
  - [Kubernetes Operator](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/kubernetes-operator)
  - [Snapshot Agent](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/snapshot-agent) *(v1.0.0 新增，预览)*

- **Helm Chart**：[NGC](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/collections/ai-dynamo) 托管支持 Dynamo Kubernetes 部署的 Helm chart：
  - [Dynamo Platform](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/helm-charts/dynamo-platform)（现已含 CRD）
  - [Snapshot](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/helm-charts/snapshot) *(v1.0.0 新增，预览)*
  - [Dynamo CRDs](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/helm-charts/dynamo-crds) *(v1.0.0 起弃用，由 Operator 管理 CRD)*
  - [Dynamo Graph](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/helm-charts/dynamo-graph) *(v0.9.0 起弃用)*

- **Rust Crate**：
  - [dynamo-runtime](https://crates.io/crates/dynamo-runtime/)
  - [dynamo-llm](https://crates.io/crates/dynamo-llm/)
  - [dynamo-protocols](https://crates.io/crates/dynamo-protocols/)
  - [dynamo-parsers](https://crates.io/crates/dynamo-parsers/)
  - [dynamo-config](https://crates.io/crates/dynamo-config/) *(v0.8.0 新增)*
  - [dynamo-memory](https://crates.io/crates/dynamo-memory/) *(v0.8.0 新增)*
  - [dynamo-tokens](https://crates.io/crates/dynamo-tokens/) *(v0.9.0 新增)*
  - [dynamo-mocker](https://crates.io/crates/dynamo-mocker/) *(v1.0.0 新增)*
  - [dynamo-kv-router](https://crates.io/crates/dynamo-kv-router/) *(v1.0.0 新增)*

确认你的平台与架构兼容后，可按 README 中的[本地快速开始](https://github.com/ai-dynamo/dynamo/blob/main/README.md#local-quick-start)安装 **Dynamo**。
