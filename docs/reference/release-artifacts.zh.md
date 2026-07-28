---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Release Artifacts
subtitle: Container images, Python wheels, Helm charts, Rust crates, and release history
---

本文档对所有 Dynamo 发布产物（容器镜像、Python wheel、Helm chart、Rust crate）进行了完整盘点。

> **另请参阅：** [Support Matrix](support-matrix.md) 了解硬件与平台兼容性 | [Feature Matrix](feature-matrix.md) 了解后端特性支持

本文中的发布历史从 v0.6.0 起。

## 当前版本：Dynamo v1.1.1

- **GitHub Release：** [v1.1.1](https://github.com/ai-dynamo/dynamo/releases/tag/v1.1.1)
- **文档：** [v1.1.1](https://docs.dynamo.nvidia.com/dynamo)
- **NGC Collection：** [ai-dynamo](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/collections/ai-dynamo)

> **实验性：** [v1.2.0-deepseek-v4-dev.3](#v120-deepseek-v4-dev3) *(Blackwell 上的 DeepSeek-V4-Flash / V4-Pro，仅 vLLM + SGLang 容器)* 作为实验性预览版本提供。带标签的 **Pre-Release** 与实验版本列于 [Pre-Release Artifacts](#pre-release-artifacts)。

### 容器镜像

| Image:Tag | 说明 | Backend | CUDA | 架构 | NGC | 备注 |
|-----------|-------------|---------|------|------|-----|-------|
| `vllm-runtime:1.1.1` | vLLM 后端 runtime 容器 | vLLM `v0.19.0` | `v12.9` | AMD64/ARM64 | [NGC: vllm-runtime 1.1.1](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/vllm-runtime?version=1.1.1) | |
| `vllm-runtime:1.1.1-cuda13` | vLLM 后端 runtime 容器（CUDA 13） | vLLM `v0.19.0` | `v13.0` | AMD64/ARM64 | [NGC: vllm-runtime 1.1.1-cuda13](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/vllm-runtime?version=1.1.1-cuda13) | |
| `vllm-runtime:1.1.1-efa-amd64` | 带 AWS EFA 的 vLLM runtime 容器 | vLLM `v0.19.0` | `v12.9` | AMD64 | [NGC: vllm-runtime 1.1.1-efa-amd64](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/vllm-runtime?version=1.1.1-efa-amd64) | 实验性 |
| `sglang-runtime:1.1.1` | SGLang 后端 runtime 容器 | SGLang `v0.5.10.post1` | `v12.9` | AMD64/ARM64 | [NGC: sglang-runtime 1.1.1](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/sglang-runtime?version=1.1.1) | |
| `sglang-runtime:1.1.1-cuda13` | SGLang 后端 runtime 容器（CUDA 13） | SGLang `v0.5.10.post1` | `v13.0` | AMD64/ARM64 | [NGC: sglang-runtime 1.1.1-cuda13](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/sglang-runtime?version=1.1.1-cuda13) | |
| `tensorrtllm-runtime:1.1.1` | TensorRT-LLM 后端 runtime 容器 | TRT-LLM `v1.3.0rc11` | `v13.1` | AMD64/ARM64 | [NGC: tensorrtllm-runtime 1.1.1](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/tensorrtllm-runtime?version=1.1.1) | |
| `tensorrtllm-runtime:1.1.1-efa-amd64` | 带 AWS EFA 的 TensorRT-LLM runtime 容器 | TRT-LLM `v1.3.0rc11` | `v13.1` | AMD64 | [NGC: tensorrtllm-runtime 1.1.1-efa-amd64](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/tensorrtllm-runtime?version=1.1.1-efa-amd64) | 实验性 |
| `dynamo-frontend:1.1.1` | 带 Endpoint Prediction Protocol（EPP）的 API 网关 | — | — | AMD64/ARM64 | [NGC: dynamo-frontend 1.1.1](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/dynamo-frontend?version=1.1.1) | |
| `dynamo-planner:1.1.1` | 由 Profiler 任务与 Planner pod 使用的独立 Planner 镜像 | — | — | AMD64/ARM64 | [NGC: dynamo-planner 1.1.1](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/dynamo-planner?version=1.1.1) | |
| `kubernetes-operator:1.1.1` | Dynamo 部署的 Kubernetes operator | — | — | AMD64/ARM64 | [NGC: kubernetes-operator 1.1.1](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/kubernetes-operator?version=1.1.1) | |
| `snapshot-agent:1.1.1` | 通过 CRIU 实现 GPU worker 快速恢复的 snapshot agent | — | — | AMD64/ARM64 | [NGC: snapshot-agent 1.1.1](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/snapshot-agent?version=1.1.1) | 预览 |

### Python Wheel

我们建议使用 TensorRT-LLM 的 NGC 容器，而非 `ai-dynamo[trtllm]` wheel。受支持的镜像见 [NGC container collection](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/collections/ai-dynamo)。

| 包 | 说明 | Python | 平台 | PyPI |
|---------|-------------|--------|----------|------|
| `ai-dynamo==1.1.1` | 主包，集成各后端（vLLM、SGLang、TRT-LLM） | `3.10`–`3.12` | Linux（glibc `v2.28+`） | [PyPI: ai-dynamo 1.1.1](https://pypi.org/project/ai-dynamo/1.1.1/) |
| `ai-dynamo-runtime==1.1.1` | Dynamo runtime 的核心 Python 绑定 | `3.10`–`3.12` | Linux（glibc `v2.28+`） | [PyPI: ai-dynamo-runtime 1.1.1](https://pypi.org/project/ai-dynamo-runtime/1.1.1/) |
| `kvbm==1.1.1` | 解耦 KV cache 的 KV Block Manager | `3.10`–`3.12` | Linux（glibc `v2.28+`） | [PyPI: kvbm 1.1.1](https://pypi.org/project/kvbm/1.1.1/) |

### Helm Chart

| Chart | 说明 | NGC |
|-------|-------------|-----|
| `dynamo-platform-1.1.1` | Dynamo 集群的平台服务（etcd、NATS）与 Dynamo Operator | [NGC Helm: dynamo-platform-1.1.1](https://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/dynamo-platform-1.1.1.tgz) |
| `snapshot-1.1.1` | 用于 GPU worker 快速恢复的 Snapshot DaemonSet | [NGC Helm: snapshot-1.1.1](https://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/snapshot-1.1.1.tgz) |

> [!NOTE]
> `dynamo-crds` Helm chart 自 v1.0.0 起已弃用；CRD 现由 Dynamo Operator 管理。`dynamo-graph` Helm chart 自 v0.9.0 起已弃用。

### Rust Crate

| Crate | 说明 | MSRV (Rust) | crates.io |
|-------|-------------|-------------|-----------|
| `dynamo-runtime@1.1.1` | 核心分布式 runtime 库 | `v1.82` | [crates.io: dynamo-runtime 1.1.1](https://crates.io/crates/dynamo-runtime/1.1.1) |
| `dynamo-llm@1.1.1` | LLM 推理引擎 | `v1.82` | [crates.io: dynamo-llm 1.1.1](https://crates.io/crates/dynamo-llm/1.1.1) |
| `dynamo-protocols@1.1.1` | 异步、OpenAI 兼容的 API 客户端 | `v1.82` | [crates.io: dynamo-protocols 1.1.1](https://crates.io/crates/dynamo-protocols/1.1.1) |
| `dynamo-async-openai@1.0.2` | 已弃用的旧 OpenAI 客户端；请使用 **`dynamo-protocols`** | `v1.82` | [crates.io: dynamo-async-openai 1.0.2](https://crates.io/crates/dynamo-async-openai/1.0.2) |
| `dynamo-parsers@1.1.1` | 协议解析器（SSE、JSON 流式） | `v1.82` | [crates.io: dynamo-parsers 1.1.1](https://crates.io/crates/dynamo-parsers/1.1.1) |
| `dynamo-memory@1.1.1` | 内存管理工具 | `v1.82` | [crates.io: dynamo-memory 1.1.1](https://crates.io/crates/dynamo-memory/1.1.1) |
| `dynamo-config@1.1.1` | 配置管理 | `v1.82` | [crates.io: dynamo-config 1.1.1](https://crates.io/crates/dynamo-config/1.1.1) |
| `dynamo-tokens@1.1.1` | LLM 推理用的 tokenizer 绑定 | `v1.82` | [crates.io: dynamo-tokens 1.1.1](https://crates.io/crates/dynamo-tokens/1.1.1) |
| `dynamo-mocker@1.1.1` | 用于基准测试的推理引擎模拟器 | `v1.82` | [crates.io: dynamo-mocker 1.1.1](https://crates.io/crates/dynamo-mocker/1.1.1) |
| `dynamo-kv-router@1.1.1` | KV 感知的请求路由库 | `v1.82` | [crates.io: dynamo-kv-router 1.1.1](https://crates.io/crates/dynamo-kv-router/1.1.1) |

## 快速安装命令

### 容器镜像（NGC）

> [!TIP]
> 详细的运行说明请参见对应后端指南：[vLLM](../backends/vllm/README.md) | [SGLang](../backends/sglang/README.md) | [TensorRT-LLM](../backends/trtllm/README.md)

```bash
# Runtime containers
docker pull nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
docker pull nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.1.1
docker pull nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:1.1.1

# CUDA 13 variants
docker pull nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1-cuda13
docker pull nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.1.1-cuda13

# EFA variants (AWS, AMD64 only, experimental)
docker pull nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1-efa-amd64
docker pull nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:1.1.1-efa-amd64

# Infrastructure containers
docker pull nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1
docker pull nvcr.io/nvidia/ai-dynamo/dynamo-planner:1.1.1
docker pull nvcr.io/nvidia/ai-dynamo/kubernetes-operator:1.1.1
docker pull nvcr.io/nvidia/ai-dynamo/snapshot-agent:1.1.1
```

### Python Wheel（PyPI）

> [!TIP]
> 详细安装说明见 README 中的 [Local Quick Start](https://github.com/ai-dynamo/dynamo#local-quick-start)。

```bash
# Install Dynamo with a specific backend (Recommended)
uv pip install "ai-dynamo[vllm]==1.1.1"
uv pip install --prerelease=allow "ai-dynamo[sglang]==1.1.1"
# TensorRT-LLM requires the NVIDIA PyPI index and pip
pip install --pre --extra-index-url https://pypi.nvidia.com "ai-dynamo[trtllm]==1.1.1"

# Install Dynamo core only
uv pip install ai-dynamo==1.1.1

# Install standalone KVBM
uv pip install kvbm==1.1.1
```

### Helm Chart（NGC）

> [!TIP]
> Kubernetes 部署说明见 [Kubernetes Installation Guide](../kubernetes/installation-guide.md)。

```bash
helm install dynamo-platform oci://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/dynamo-platform --version 1.1.1
helm install snapshot oci://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/snapshot --version 1.1.1
```

### Rust Crate（crates.io）

> [!TIP]
> API 文档请见各 crate 在 [docs.rs](https://docs.rs/) 上的页面。从源码构建 Dynamo 见 [Building from Source](https://github.com/ai-dynamo/dynamo#building-from-source)。

```bash
cargo add dynamo-runtime@1.1.1
cargo add dynamo-llm@1.1.1
cargo add dynamo-protocols@1.1.1
# Deprecated legacy crate name — pin only if a dependency requires it; new code should use dynamo-protocols:
# cargo add dynamo-async-openai@1.0.2
cargo add dynamo-parsers@1.1.1
cargo add dynamo-memory@1.1.1
cargo add dynamo-config@1.1.1
cargo add dynamo-tokens@1.1.1
cargo add dynamo-mocker@1.1.1
cargo add dynamo-kv-router@1.1.1
```

**CUDA 与驱动要求：** 各容器镜像所需的具体 CUDA toolkit 版本与最低驱动要求见 [Support Matrix](support-matrix.md#cuda-and-driver-requirements)。

## 已知问题

完整的已知问题列表请参见各版本的 release notes：
- [v1.1.1 Release Notes](https://github.com/ai-dynamo/dynamo/releases/tag/v1.1.1)
- [v1.1.0 Release Notes](https://github.com/ai-dynamo/dynamo/releases/tag/v1.1.0)
- [v1.0.2 Release Notes](https://github.com/ai-dynamo/dynamo/releases/tag/v1.0.2)
- [v1.0.1 Release Notes](https://github.com/ai-dynamo/dynamo/releases/tag/v1.0.1)
- [v1.0.0 Release Notes](https://github.com/ai-dynamo/dynamo/releases/tag/v1.0.0)
- [v0.9.0 Release Notes](https://github.com/ai-dynamo/dynamo/releases/tag/v0.9.0)
- [v0.8.1 Release Notes](https://github.com/ai-dynamo/dynamo/releases/tag/v0.8.1)

### 已知产物问题

| 版本 | 产物 | 问题 | 状态 |
|---------|----------|-------|--------|
| v0.9.0 | `dynamo-platform-0.9.0` | Helm chart 把 operator 镜像设成 `0.7.1` 而非 `0.9.0`。 | 已在 v0.9.0.post1 修复 |
| v0.8.1 | `vllm-runtime:0.8.1-cuda13` | 容器无法启动。 | 已知问题 |
| v0.8.1 | `sglang-runtime:0.8.1-cuda13`、`vllm-runtime:0.8.1-cuda13` | ARM64 上多模态预期不可用。AMD64 上可用。 | 已知限制 |
| v0.8.0 | `sglang-runtime:0.8.0-cuda13` | CuDNN 安装问题导致 PyTorch `v2.9.1` 与 `nn.Conv3d` 兼容性问题，进而在多模态工作负载下出现性能下降与显存占用过高。 | 已在 v0.8.1 修复（[#5461](https://github.com/ai-dynamo/dynamo/pull/5461)） |

---

## 发布产物历史

每条 bullet 表示相对 NGC / Helm / PyPI / crates.io 上发布的 **增量**：新增 crate、移除的 Helm chart，或在 registry 上 **拆分** 或 **新出现** 的镜像行。完整矩阵见上方的盘点表。

先列稳定版（按时间倒序）。**Pre-Release Git Tag**（`v*-dev.*`、实验分支）在下方做总结；按 tag 列出的具体镜像与 wheel 见 [Pre-Release Artifacts](#pre-release-artifacts)。

后端版本对应见上方 version-pins 表与下方的 [GitHub Releases](#github-releases) 表。

**稳定版**

- **v1.1.1**：补丁版本。后端版本与 v1.1.0 相同：SGLang `v0.5.10.post1`（NIXL `v1.0.1`）、TRT-LLM `v1.3.0rc11`（NIXL `v0.10.1`）、vLLM `v0.19.0`（NIXL `v0.10.1`）。
- **v1.1.0**：**镜像：** 把 Planner 拆分为独立的 `dynamo-planner` 镜像在 NGC 发布，供 Profiler 任务与 Planner pod 使用；worker 与 runtime 镜像不再打包 Planner（**产物边界变化**，并非新引擎能力）。**Crate：** **`dynamo-protocols`**（多协议类型）首次以 **`1.y.z`** 在 crates.io 发布；**`dynamo-async-openai`** 仍然弃用，最终版本为 **`1.0.2`**）。
- **v1.0.2 / v1.0.1**：相对 v1.0.0 无产物增删。
- **v1.0.0**：**镜像：** `snapshot-agent`，vLLM 与 TRT-LLM 的 EFA 变体（仅 AMD64）。**Crate：** 首发 `dynamo-mocker`、`dynamo-kv-router`。**Helm：** 新增 `snapshot`（预览）；从发布流中移除已弃用的 `dynamo-crds`（CRD 由 Operator 管理）。
- **v0.9.1**：相对 v0.9.0 无产物增删。
- **v0.9.0**：**Crate：** 首发 `dynamo-tokens`。**Helm：** 从发布流中移除已弃用的 `dynamo-graph`。
- **v0.8.0**：**镜像：** `dynamo-frontend`，vLLM 与 SGLang 的 CUDA 13 变体。**Crate：** 首发 `dynamo-memory`、`dynamo-config`。

**Dynamo Nightlies**

- **自 v1.1.0\* 起新增：** **`ai-dynamo`** 与 **`ai-dynamo-runtime`** —— 来自 **`main`** 的 nightly 构建发布带 **`*.devYYYYMMDD`** tag 的 wheel。可使用 **`pip`** 或 **`uv`** 加 **`--pre`** 与与 [Pre-Release Artifacts](#pre-release-artifacts) 相同的 NVIDIA extra-index 模式来安装。

\* nightly **`main`** wheel 的 **`*.devYYYYMMDD`** 版本号方案自 **2026 年 4 月 24 日** 开始。

**Pre-Release 与实验性 Git Tag**

- **v1.2.0-deepseek-v4-dev.3**：**镜像：** `vllm-runtime:*-deepseek-v4-cuda13-dev.3`、`sglang-runtime:*-deepseek-v4-cuda12-dev.3`、`sglang-runtime:*-deepseek-v4-cuda13-dev.3`。**Helm / PyPI：** 此 tag 不发布（见 [Pre-Release Artifacts](#v120-deepseek-v4-dev3)）。
- **v1.1.0-dev.3**：**镜像：** `tensorrtllm-runtime:1.1.0-dev.3`。**Wheel：** `ai-dynamo`、`ai-dynamo-runtime` 在 [pypi.nvidia.com](https://pypi.nvidia.com/)（见 [下文](#v110-dev3)）。
- **v1.1.0-dev.2**：**镜像：** `sglang-runtime:1.1.0-dev.2`、`tensorrtllm-runtime:1.1.0-dev.2`。**Wheel：** `ai-dynamo`、`ai-dynamo-runtime` 在 [pypi.nvidia.com](https://pypi.nvidia.com/)（见 [下文](#v110-dev2)）。
- **v1.1.0-dev.1**：**镜像：** vLLM、SGLang、TRT-LLM runtime 矩阵（CUDA 12 / 13 与列出的 EFA 变体）、`dynamo-frontend`、`kubernetes-operator`、`snapshot-agent`。**Wheel：** `ai-dynamo`、`ai-dynamo-runtime` 在 [pypi.nvidia.com](https://pypi.nvidia.com/)。**Helm：** `dynamo-platform`、`snapshot` 在 `1.1.0-dev.1`（见 [下文](#v110-dev1)）。

**仅 Helm 的补丁**

- **v0.9.0.post1**：仅重发 `dynamo-platform` Helm chart（修正 operator 镜像 tag）。

**仅后端的补丁链**

- **v0.8.1.post1 / .post2 / .post3**：仅重发 TRT-LLM runtime 镜像与 PyPI wheel。

### crates.io Rust 包

这些 crate 使用仓库 `https://github.com/ai-dynamo/dynamo.git`。下表列出每个 crate 在 crates.io 上的 **首个非占位** 发布版本（不计名为 `0.0.0-prerelease.0` 的预占上传）。日期来源于 crates.io registry index。

| Crate | 首发版本 | 日期（crates.io） |
|-------|-------------------------|------------------|
| `dynamo-runtime` | `0.1.0` | 2025-03-18 |
| `dynamo-llm` | `0.2.0` | 2025-05-01 |
| `dynamo-async-openai` | `0.4.1` | 2025-08-27 |
| `dynamo-parsers` | `0.5.0` | 2025-09-18 |
| `dynamo-memory` | `0.8.0` | 2026-01-15 |
| `dynamo-config` | `0.8.0` | 2026-01-15 |
| `dynamo-tokens` | `0.9.0` | 2026-02-12 |
| `dynamo-mocker` | `1.0.0` | 2026-03-13 |
| `dynamo-kv-router` | `1.0.0` | 2026-03-13 |
| `dynamo-protocols` | `1.1.0` | 2026-05-04 |

**`dynamo-async-openai`** 已 **弃用**；**`1.0.2`** 是其在 crates.io 上的最终版本。新依赖请使用 **`dynamo-protocols`**（[crate](https://crates.io/crates/dynamo-protocols)）。

**`dynamo-tokenizers`** 仅存在于 Dynamo workspace 内，**未** 在 crates.io 发布。

### GitHub Releases

| 版本 | 发布日期 | GitHub | 文档 | 备注 |
|---------|--------------|--------|------|-------|
| `v1.2.0-deepseek-v4-dev.3` | May 9, 2026 | [Tag](https://github.com/ai-dynamo/dynamo/releases/tag/v1.2.0-deepseek-v4-dev.3) | — | 实验性（DeepSeek-V4-Flash / V4-Pro Blackwell 预览；仅 vLLM + SGLang 容器） |
| `v1.2.0-deepseek-v4-dev.2` | May 1, 2026 | [Tag](https://github.com/ai-dynamo/dynamo/releases/tag/v1.2.0-deepseek-v4-dev.2) | — | 实验性（DeepSeek-V4-Flash / V4-Pro Blackwell 预览；仅 vLLM + SGLang 容器） |
| `v1.1.1` | May 5, 2026 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v1.1.1) | [Docs](https://docs.dynamo.nvidia.com/dynamo) | |
| `v1.1.0` | May 1, 2026 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v1.1.0) | [Docs](https://docs.dynamo.nvidia.com/dynamo) | |
| `v1.1.0-dev.3` | Apr 18, 2026 | [Tag](https://github.com/ai-dynamo/dynamo/releases/tag/v1.1.0-dev.3) | — | Pre-Release（TRT-LLM Runtime Image + Wheel；见 Pre-Release Artifacts） |
| `v1.1.0-dev.2` | Apr 9, 2026 | [Tag](https://github.com/ai-dynamo/dynamo/releases/tag/v1.1.0-dev.2) | — | Pre-Release（SGLang + TRT-LLM Runtime Images + Wheel；见 Pre-Release Artifacts） |
| `v1.1.0-dev.1` | Mar 17, 2026 | [Tag](https://github.com/ai-dynamo/dynamo/releases/tag/v1.1.0-dev.1) | — | 实验性 |
| `v1.0.2` | Apr 22, 2026 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v1.0.2) | [Docs](https://docs.dynamo.nvidia.com/dynamo) | |
| `v1.0.1` | Mar 16, 2026 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v1.0.1) | [Docs](https://docs.dynamo.nvidia.com/dynamo) | |
| `v1.0.0` | Mar 12, 2026 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v1.0.0) | [Docs](https://docs.dynamo.nvidia.com/dynamo) | |
| `v0.9.1` | Mar 4, 2026 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v0.9.1) | [Docs](https://docs.dynamo.nvidia.com/dynamo) |
| `v0.9.0` | Feb 11, 2026 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v0.9.0) | 归档文档不可用 |
| `v0.8.1` | Jan 23, 2026 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v0.8.1) | 归档文档不可用 |
| `v0.8.0` | Jan 15, 2026 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v0.8.0) | 归档文档不可用 |
| `v0.7.1` | Dec 15, 2025 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v0.7.1) | 归档文档不可用 |
| `v0.7.0` | Nov 26, 2025 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v0.7.0) | 归档文档不可用 |
| `v0.6.1` | Nov 6, 2025 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v0.6.1) | — |
| `v0.6.0` | Oct 28, 2025 | [Release](https://github.com/ai-dynamo/dynamo/releases/tag/v0.6.0) | — |

### 容器镜像

> **NGC Collection：** [ai-dynamo](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/collections/ai-dynamo)
>
> 访问特定版本，可在容器 URL 后追加 `?version=TAG`：
> `https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/{container}?version={tag}`

#### vllm-runtime

| Image:Tag | vLLM | 架构 | CUDA | 备注 |
|-----------|------|------|------|-------|
| `vllm-runtime:1.1.1` | `v0.19.0` | AMD64/ARM64 | `v12.9` | |
| `vllm-runtime:1.1.1-cuda13` | `v0.19.0` | AMD64/ARM64 | `v13.0` | |
| `vllm-runtime:1.1.1-efa-amd64` | `v0.19.0` | AMD64 | `v12.9` | 实验性 |
| `vllm-runtime:1.1.0` | `v0.19.0` | AMD64/ARM64 | `v12.9` | |
| `vllm-runtime:1.1.0-cuda13` | `v0.19.0` | AMD64/ARM64 | `v13.0` | |
| `vllm-runtime:1.1.0-efa-amd64` | `v0.19.0` | AMD64 | `v12.9` | 实验性 |
| `vllm-runtime:1.0.2` | `v0.16.0` | AMD64/ARM64 | `v12.9` | |
| `vllm-runtime:1.0.2-cuda13` | `v0.16.0` | AMD64/ARM64 | `v13.0` | |
| `vllm-runtime:1.0.2-efa-amd64` | `v0.16.0` | AMD64 | `v12.9` | 实验性 |
| `vllm-runtime:1.0.1` | `v0.16.0` | AMD64/ARM64 | `v12.9` | |
| `vllm-runtime:1.0.1-cuda13` | `v0.16.0` | AMD64/ARM64 | `v13.0` | |
| `vllm-runtime:1.0.1-efa-amd64` | `v0.16.0` | AMD64 | `v12.9` | 实验性 |
| `vllm-runtime:1.0.0` | `v0.16.0` | AMD64/ARM64 | `v12.9` | |
| `vllm-runtime:1.0.0-cuda13` | `v0.16.0` | AMD64/ARM64 | `v13.0` | |
| `vllm-runtime:1.0.0-efa-amd64` | `v0.16.0` | AMD64 | `v12.9` | 实验性 |
| `vllm-runtime:0.9.1` | `v0.14.1` | AMD64/ARM64 | `v12.9` | |
| `vllm-runtime:0.9.1-cuda13` | `v0.14.1` | AMD64/ARM64 | `v13.0` | 实验性 |
| `vllm-runtime:0.9.0` | `v0.14.1` | AMD64/ARM64 | `v12.9` | |
| `vllm-runtime:0.9.0-cuda13` | `v0.14.1` | AMD64/ARM64 | `v13.0` | 实验性 |
| `vllm-runtime:0.8.1` | `v0.12.0` | AMD64/ARM64 | `v12.9` | |
| `vllm-runtime:0.8.0` | `v0.12.0` | AMD64/ARM64 | `v12.9` | |
| `vllm-runtime:0.8.0-cuda13` | `v0.12.0` | AMD64/ARM64 | `v13.0` | 实验性 |
| `vllm-runtime:0.7.0.post2` | `v0.11.2` | AMD64/ARM64 | `v12.8` | 补丁 |
| `vllm-runtime:0.7.1` | `v0.11.0` | AMD64/ARM64 | `v12.8` | |
| `vllm-runtime:0.7.0.post1` | `v0.11.0` | AMD64/ARM64 | `v12.8` | 补丁 |
| `vllm-runtime:0.7.0` | `v0.11.0` | AMD64/ARM64 | `v12.8` | |
| `vllm-runtime:0.6.1.post1` | `v0.11.0` | AMD64/ARM64 | `v12.8` | 补丁 |
| `vllm-runtime:0.6.1` | `v0.11.0` | AMD64/ARM64 | `v12.8` | |
| `vllm-runtime:0.6.0` | `v0.11.0` | AMD64 | `v12.8` | |

#### sglang-runtime

| Image:Tag | SGLang | 架构 | CUDA | 备注 |
|-----------|--------|------|------|-------|
| `sglang-runtime:1.1.1` | `v0.5.10.post1` | AMD64/ARM64 | `v12.9` | |
| `sglang-runtime:1.1.1-cuda13` | `v0.5.10.post1` | AMD64/ARM64 | `v13.0` | |
| `sglang-runtime:1.1.0` | `v0.5.10.post1` | AMD64/ARM64 | `v12.9` | |
| `sglang-runtime:1.1.0-cuda13` | `v0.5.10.post1` | AMD64/ARM64 | `v13.0` | |
| `sglang-runtime:1.0.2` | `v0.5.9` | AMD64/ARM64 | `v12.9` | |
| `sglang-runtime:1.0.2-cuda13` | `v0.5.9` | AMD64/ARM64 | `v13.0` | |
| `sglang-runtime:1.0.1` | `v0.5.9` | AMD64/ARM64 | `v12.9` | |
| `sglang-runtime:1.0.1-cuda13` | `v0.5.9` | AMD64/ARM64 | `v13.0` | |
| `sglang-runtime:1.0.0` | `v0.5.9` | AMD64/ARM64 | `v12.9` | |
| `sglang-runtime:1.0.0-cuda13` | `v0.5.9` | AMD64/ARM64 | `v13.0` | |
| `sglang-runtime:0.9.1` | `v0.5.8` | AMD64/ARM64 | `v12.9` | |
| `sglang-runtime:0.9.1-cuda13` | `v0.5.8` | AMD64/ARM64 | `v13.0` | 实验性 |
| `sglang-runtime:0.9.0` | `v0.5.8` | AMD64/ARM64 | `v12.9` | |
| `sglang-runtime:0.9.0-cuda13` | `v0.5.8` | AMD64/ARM64 | `v13.0` | 实验性 |
| `sglang-runtime:0.8.1` | `v0.5.6.post2` | AMD64/ARM64 | `v12.9` | |
| `sglang-runtime:0.8.1-cuda13` | `v0.5.6.post2` | AMD64/ARM64 | `v13.0` | 实验性 |
| `sglang-runtime:0.8.0` | `v0.5.6.post2` | AMD64/ARM64 | `v12.9` | |
| `sglang-runtime:0.8.0-cuda13` | `v0.5.6.post2` | AMD64/ARM64 | `v13.0` | 实验性 |
| `sglang-runtime:0.7.1` | `v0.5.4.post3` | AMD64/ARM64 | `v12.9` | |
| `sglang-runtime:0.7.0.post1` | `v0.5.4.post3` | AMD64/ARM64 | `v12.9` | 补丁 |
| `sglang-runtime:0.7.0` | `v0.5.4.post3` | AMD64/ARM64 | `v12.9` | |
| `sglang-runtime:0.6.1.post1` | `v0.5.3.post2` | AMD64/ARM64 | `v12.9` | 补丁 |
| `sglang-runtime:0.6.1` | `v0.5.3.post2` | AMD64/ARM64 | `v12.9` | |
| `sglang-runtime:0.6.0` | `v0.5.3.post2` | AMD64 | `v12.8` | |

#### tensorrtllm-runtime

| Image:Tag | TRT-LLM | 架构 | CUDA | 备注 |
|-----------|---------|------|------|-------|
| `tensorrtllm-runtime:1.1.1` | `v1.3.0rc11` | AMD64/ARM64 | `v13.1` | |
| `tensorrtllm-runtime:1.1.1-efa-amd64` | `v1.3.0rc11` | AMD64 | `v13.1` | 实验性 |
| `tensorrtllm-runtime:1.1.0` | `v1.3.0rc11` | AMD64/ARM64 | `v13.1` | |
| `tensorrtllm-runtime:1.1.0-efa-amd64` | `v1.3.0rc11` | AMD64 | `v13.1` | 实验性 |
| `tensorrtllm-runtime:1.0.2` | `v1.3.0rc5.post1` | AMD64/ARM64 | `v13.1` | |
| `tensorrtllm-runtime:1.0.2-efa-amd64` | `v1.3.0rc5.post1` | AMD64 | `v13.1` | 实验性 |
| `tensorrtllm-runtime:1.0.1` | `v1.3.0rc5.post1` | AMD64/ARM64 | `v13.1` | |
| `tensorrtllm-runtime:1.0.1-efa-amd64` | `v1.3.0rc5.post1` | AMD64 | `v13.1` | 实验性 |
| `tensorrtllm-runtime:1.0.0` | `v1.3.0rc5.post1` | AMD64/ARM64 | `v13.1` | |
| `tensorrtllm-runtime:1.0.0-efa-amd64` | `v1.3.0rc5.post1` | AMD64 | `v13.1` | 实验性 |
| `tensorrtllm-runtime:0.9.1` | `v1.3.0rc3` | AMD64/ARM64 | `v13.0` | |
| `tensorrtllm-runtime:0.9.0` | `v1.3.0rc1` | AMD64/ARM64 | `v13.0` | |
| `tensorrtllm-runtime:0.8.1.post3` | `v1.2.0rc6.post3` | AMD64/ARM64 | `v13.0` | 补丁 |
| `tensorrtllm-runtime:0.8.1.post1` | `v1.2.0rc6.post2` | AMD64/ARM64 | `v13.0` | 补丁 |
| `tensorrtllm-runtime:0.8.1` | `v1.2.0rc6.post1` | AMD64/ARM64 | `v13.0` | |
| `tensorrtllm-runtime:0.8.0` | `v1.2.0rc6.post1` | AMD64/ARM64 | `v13.0` | |
| `tensorrtllm-runtime:0.7.0.post2` | `v1.2.0rc2` | AMD64/ARM64 | `v13.0` | 补丁 |
| `tensorrtllm-runtime:0.7.1` | `v1.2.0rc3` | AMD64/ARM64 | `v13.0` | |
| `tensorrtllm-runtime:0.7.0.post1` | `v1.2.0rc3` | AMD64/ARM64 | `v13.0` | 补丁 |
| `tensorrtllm-runtime:0.7.0` | `v1.2.0rc2` | AMD64/ARM64 | `v13.0` | |
| `tensorrtllm-runtime:0.6.1-cuda13` | `v1.2.0rc1` | AMD64/ARM64 | `v13.0` | 实验性 |
| `tensorrtllm-runtime:0.6.1.post1` | `v1.1.0rc5` | AMD64/ARM64 | `v12.9` | 补丁 |
| `tensorrtllm-runtime:0.6.1` | `v1.1.0rc5` | AMD64/ARM64 | `v12.9` | |
| `tensorrtllm-runtime:0.6.0` | `v1.1.0rc5` | AMD64/ARM64 | `v12.9` | |

#### dynamo-frontend

| Image:Tag | 架构 | 备注 |
|-----------|------|-------|
| `dynamo-frontend:1.1.1` | AMD64/ARM64 | |
| `dynamo-frontend:1.1.0` | AMD64/ARM64 | |
| `dynamo-frontend:1.0.2` | AMD64/ARM64 | |
| `dynamo-frontend:1.0.1` | AMD64/ARM64 | |
| `dynamo-frontend:1.0.0` | AMD64/ARM64 | |
| `dynamo-frontend:0.9.1` | AMD64/ARM64 | |
| `dynamo-frontend:0.9.0` | AMD64/ARM64 | |
| `dynamo-frontend:0.8.1` | AMD64/ARM64 | |
| `dynamo-frontend:0.8.0` | AMD64/ARM64 | 首发 |

#### kubernetes-operator

| Image:Tag | 架构 | 备注 |
|-----------|------|-------|
| `kubernetes-operator:1.1.1` | AMD64/ARM64 | |
| `kubernetes-operator:1.1.0` | AMD64/ARM64 | |
| `kubernetes-operator:1.0.2` | AMD64/ARM64 | |
| `kubernetes-operator:1.0.1` | AMD64/ARM64 | |
| `kubernetes-operator:1.0.0` | AMD64/ARM64 | |
| `kubernetes-operator:0.9.1` | AMD64/ARM64 | |
| `kubernetes-operator:0.9.0` | AMD64/ARM64 | |
| `kubernetes-operator:0.8.1` | AMD64/ARM64 | |
| `kubernetes-operator:0.8.0` | AMD64/ARM64 | |
| `kubernetes-operator:0.7.1` | AMD64/ARM64 | |
| `kubernetes-operator:0.7.0.post1` | AMD64/ARM64 | 补丁 |
| `kubernetes-operator:0.7.0` | AMD64/ARM64 | |
| `kubernetes-operator:0.6.1` | AMD64/ARM64 | |
| `kubernetes-operator:0.6.0` | AMD64/ARM64 | |

#### dynamo-planner

| Image:Tag | 架构 | 备注 |
|-----------|------|-------|
| `dynamo-planner:1.1.1` | AMD64/ARM64 | |
| `dynamo-planner:1.1.0` | AMD64/ARM64 | 新增 |

#### snapshot-agent

| Image:Tag | 架构 | 备注 |
|-----------|------|-------|
| `snapshot-agent:1.1.1` | AMD64/ARM64 | 预览 |
| `snapshot-agent:1.1.0` | AMD64/ARM64 | 预览 |
| `snapshot-agent:1.0.2` | AMD64/ARM64 | 预览 |
| `snapshot-agent:1.0.1` | AMD64/ARM64 | 预览 |
| `snapshot-agent:1.0.0` | AMD64/ARM64 | 预览 |

### Python Wheel

> **PyPI：** [ai-dynamo](https://pypi.org/project/ai-dynamo/) | [ai-dynamo-runtime](https://pypi.org/project/ai-dynamo-runtime/) | [kvbm](https://pypi.org/project/kvbm/)
>
> 访问特定版本：`https://pypi.org/project/{package}/{version}/`

#### ai-dynamo (wheel)

| 包 | Python | 平台 | 备注 |
|---------|--------|----------|-------|
| `ai-dynamo==1.1.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo==1.1.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo==1.0.2` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo==1.0.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo==1.0.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo==0.9.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo==0.9.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo==0.8.1.post3` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | TRT-LLM `v1.2.0rc6.post3` |
| `ai-dynamo==0.8.1.post1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | TRT-LLM `v1.2.0rc6.post2` |
| `ai-dynamo==0.8.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo==0.8.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo==0.7.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo==0.7.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo==0.6.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo==0.6.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |

#### ai-dynamo-runtime (wheel)

| 包 | Python | 平台 | 备注 |
|---------|--------|----------|-------|
| `ai-dynamo-runtime==1.1.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo-runtime==1.1.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo-runtime==1.0.2` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo-runtime==1.0.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo-runtime==1.0.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo-runtime==0.9.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo-runtime==0.9.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo-runtime==0.8.1.post3` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | TRT-LLM `v1.2.0rc6.post3` |
| `ai-dynamo-runtime==0.8.1.post1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | TRT-LLM `v1.2.0rc6.post2` |
| `ai-dynamo-runtime==0.8.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo-runtime==0.8.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo-runtime==0.7.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo-runtime==0.7.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo-runtime==0.6.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `ai-dynamo-runtime==0.6.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |

#### kvbm (wheel)

| 包 | Python | 平台 | 备注 |
|---------|--------|----------|-------|
| `kvbm==1.1.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `kvbm==1.1.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `kvbm==1.0.2` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `kvbm==1.0.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `kvbm==1.0.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `kvbm==0.9.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `kvbm==0.9.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `kvbm==0.8.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `kvbm==0.8.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `kvbm==0.7.1` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | |
| `kvbm==0.7.0` | `3.10`–`3.12` | Linux（glibc `v2.28+`） | 首发 |

### Helm Chart

> **NGC Helm Registry：** [ai-dynamo](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/collections/ai-dynamo)
>
> 直接下载：`https://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/{chart}-{version}.tgz`

#### dynamo-crds (Helm chart) -- 已弃用

> [!NOTE]
> `dynamo-crds` Helm chart 自 v1.0.0 起已弃用。CRD 现由 Dynamo Operator 管理。

| Chart | 备注 |
|-------|-------|
| `dynamo-crds-0.9.1` | 最后一版 |
| `dynamo-crds-0.9.0` | |
| `dynamo-crds-0.8.1` | |
| `dynamo-crds-0.8.0` | |
| `dynamo-crds-0.7.1` | |
| `dynamo-crds-0.7.0` | |
| `dynamo-crds-0.6.1` | |
| `dynamo-crds-0.6.0` | |

#### dynamo-platform (Helm chart)

| Chart | 备注 |
|-------|-------|
| `dynamo-platform-1.1.1` | |
| `dynamo-platform-1.1.0` | |
| `dynamo-platform-1.0.2` | |
| `dynamo-platform-1.0.1` | |
| `dynamo-platform-1.0.0` | |
| `dynamo-platform-0.9.1` | |
| `dynamo-platform-0.9.0-post1` | Helm 修复：operator 镜像 tag |
| `dynamo-platform-0.9.0` | |
| `dynamo-platform-0.8.1` | |
| `dynamo-platform-0.8.0` | |
| `dynamo-platform-0.7.1` | |
| `dynamo-platform-0.7.0` | |
| `dynamo-platform-0.6.1` | |
| `dynamo-platform-0.6.0` | |

#### snapshot (Helm chart)

| Chart | 备注 |
|-------|-------|
| `snapshot-1.1.1` | 预览 |
| `snapshot-1.1.0` | 预览 |
| `snapshot-1.0.2` | 预览 |
| `snapshot-1.0.1` | 预览 |
| `snapshot-1.0.0` | 预览 |

#### dynamo-graph (Helm chart) -- 已弃用

> [!NOTE]
> `dynamo-graph` Helm chart 自 v0.9.0 起已弃用。

| Chart | 备注 |
|-------|-------|
| `dynamo-graph-0.8.1` | 最后一版 |
| `dynamo-graph-0.8.0` | |
| `dynamo-graph-0.7.1` | |
| `dynamo-graph-0.7.0` | |
| `dynamo-graph-0.6.1` | |
| `dynamo-graph-0.6.0` | |

### Rust Crate

> **crates.io：** [dynamo-runtime](https://crates.io/crates/dynamo-runtime) | [dynamo-llm](https://crates.io/crates/dynamo-llm) | [dynamo-protocols](https://crates.io/crates/dynamo-protocols) | [dynamo-async-openai](https://crates.io/crates/dynamo-async-openai) *（已弃用）* | [dynamo-parsers](https://crates.io/crates/dynamo-parsers) | [dynamo-memory](https://crates.io/crates/dynamo-memory) | [dynamo-config](https://crates.io/crates/dynamo-config) | [dynamo-tokens](https://crates.io/crates/dynamo-tokens)
>
> 访问特定版本：`https://crates.io/crates/{crate}/{version}`

#### dynamo-runtime (crate)

| Crate | MSRV (Rust) | 备注 |
|-------|-------------|-------|
| `dynamo-runtime@1.1.1` | `v1.82` | |
| `dynamo-runtime@1.1.0` | `v1.82` | |
| `dynamo-runtime@1.0.2` | `v1.82` | |
| `dynamo-runtime@1.0.1` | `v1.82` | |
| `dynamo-runtime@1.0.0` | `v1.82` | |
| `dynamo-runtime@0.9.1` | `v1.82` | |
| `dynamo-runtime@0.9.0` | `v1.82` | |
| `dynamo-runtime@0.8.1` | `v1.82` | |
| `dynamo-runtime@0.8.0` | `v1.82` | |
| `dynamo-runtime@0.7.1` | `v1.82` | |
| `dynamo-runtime@0.7.0` | `v1.82` | |
| `dynamo-runtime@0.6.1` | `v1.82` | |
| `dynamo-runtime@0.6.0` | `v1.82` | |

#### dynamo-llm (crate)

| Crate | MSRV (Rust) | 备注 |
|-------|-------------|-------|
| `dynamo-llm@1.1.1` | `v1.82` | |
| `dynamo-llm@1.1.0` | `v1.82` | |
| `dynamo-llm@1.0.2` | `v1.82` | |
| `dynamo-llm@1.0.1` | `v1.82` | |
| `dynamo-llm@1.0.0` | `v1.82` | |
| `dynamo-llm@0.9.1` | `v1.82` | |
| `dynamo-llm@0.9.0` | `v1.82` | |
| `dynamo-llm@0.8.1` | `v1.82` | |
| `dynamo-llm@0.8.0` | `v1.82` | |
| `dynamo-llm@0.7.1` | `v1.82` | |
| `dynamo-llm@0.7.0` | `v1.82` | |
| `dynamo-llm@0.6.1` | `v1.82` | |
| `dynamo-llm@0.6.0` | `v1.82` | |

#### dynamo-protocols (crate)

在 crates.io 上，**`dynamo-protocols`** 列出的首个可安装版本是 **`1.1.0`**（如其他 **`0.0.0-prerelease.*`** 上传一样，占位预占的 **`0.0.0-prerelease.0`** 不计）。OpenAI 兼容客户端更早的 semver 线发布在 **`dynamo-async-openai`** 下——见下方 **`#### dynamo-async-openai (crate)`**。

| Crate | MSRV (Rust) | 备注 |
|-------|-------------|-------|
| `dynamo-protocols@1.1.1` | `v1.82` | |
| `dynamo-protocols@1.1.0` | `v1.82` | |

#### dynamo-async-openai (crate)

**已弃用。** 请改用 **`dynamo-protocols`**。该 crate 仍在 crates.io 上发布，供仍 pin 旧包名的 manifest 使用。

| Crate | MSRV (Rust) | 备注 |
|-------|-------------|-------|
| `dynamo-async-openai@1.0.2` | `v1.82` | crates.io 最终版本 |
| `dynamo-async-openai@1.0.1` | `v1.82` | |
| `dynamo-async-openai@1.0.0` | `v1.82` | |
| `dynamo-async-openai@0.9.1` | `v1.82` | |
| `dynamo-async-openai@0.9.0` | `v1.82` | |
| `dynamo-async-openai@0.8.1` | `v1.82` | |
| `dynamo-async-openai@0.8.0` | `v1.82` | |
| `dynamo-async-openai@0.7.1` | `v1.82` | |
| `dynamo-async-openai@0.7.0` | `v1.82` | |
| `dynamo-async-openai@0.7.0-post1` | `v1.82` | |
| `dynamo-async-openai@0.6.1` | `v1.82` | |
| `dynamo-async-openai@0.6.0` | `v1.82` | |
| `dynamo-async-openai@0.5.1` | `v1.82` | |
| `dynamo-async-openai@0.5.0` | `v1.82` | |
| `dynamo-async-openai@0.4.1` | `v1.82` | |

#### dynamo-parsers (crate)

| Crate | MSRV (Rust) | 备注 |
|-------|-------------|-------|
| `dynamo-parsers@1.1.1` | `v1.82` | |
| `dynamo-parsers@1.1.0` | `v1.82` | |
| `dynamo-parsers@1.0.2` | `v1.82` | |
| `dynamo-parsers@1.0.1` | `v1.82` | |
| `dynamo-parsers@1.0.0` | `v1.82` | |
| `dynamo-parsers@0.9.1` | `v1.82` | |
| `dynamo-parsers@0.9.0` | `v1.82` | |
| `dynamo-parsers@0.8.1` | `v1.82` | |
| `dynamo-parsers@0.8.0` | `v1.82` | |
| `dynamo-parsers@0.7.1` | `v1.82` | |
| `dynamo-parsers@0.7.0` | `v1.82` | |
| `dynamo-parsers@0.6.1` | `v1.82` | |
| `dynamo-parsers@0.6.0` | `v1.82` | |

#### dynamo-memory (crate)

| Crate | MSRV (Rust) | 备注 |
|-------|-------------|-------|
| `dynamo-memory@1.1.1` | `v1.82` | |
| `dynamo-memory@1.1.0` | `v1.82` | |
| `dynamo-memory@1.0.2` | `v1.82` | |
| `dynamo-memory@1.0.1` | `v1.82` | |
| `dynamo-memory@1.0.0` | `v1.82` | |
| `dynamo-memory@0.9.1` | `v1.82` | |
| `dynamo-memory@0.9.0` | `v1.82` | |
| `dynamo-memory@0.8.1` | `v1.82` | |
| `dynamo-memory@0.8.0` | `v1.82` | 首发 |

#### dynamo-config (crate)

| Crate | MSRV (Rust) | 备注 |
|-------|-------------|-------|
| `dynamo-config@1.1.1` | `v1.82` | |
| `dynamo-config@1.1.0` | `v1.82` | |
| `dynamo-config@1.0.2` | `v1.82` | |
| `dynamo-config@1.0.1` | `v1.82` | |
| `dynamo-config@1.0.0` | `v1.82` | |
| `dynamo-config@0.9.1` | `v1.82` | |
| `dynamo-config@0.9.0` | `v1.82` | |
| `dynamo-config@0.8.1` | `v1.82` | |
| `dynamo-config@0.8.0` | `v1.82` | 首发 |

#### dynamo-tokens (crate)

| Crate | MSRV (Rust) | 备注 |
|-------|-------------|-------|
| `dynamo-tokens@1.1.1` | `v1.82` | |
| `dynamo-tokens@1.1.0` | `v1.82` | |
| `dynamo-tokens@1.0.2` | `v1.82` | |
| `dynamo-tokens@1.0.1` | `v1.82` | |
| `dynamo-tokens@1.0.0` | `v1.82` | |
| `dynamo-tokens@0.9.1` | `v1.82` | |
| `dynamo-tokens@0.9.0` | `v1.82` | 首发 |

#### dynamo-mocker (crate)

| Crate | MSRV (Rust) | 备注 |
|-------|-------------|-------|
| `dynamo-mocker@1.1.1` | `v1.82` | |
| `dynamo-mocker@1.1.0` | `v1.82` | |
| `dynamo-mocker@1.0.2` | `v1.82` | |
| `dynamo-mocker@1.0.1` | `v1.82` | |
| `dynamo-mocker@1.0.0` | `v1.82` | 首发 |

#### dynamo-kv-router (crate)

| Crate | MSRV (Rust) | 备注 |
|-------|-------------|-------|
| `dynamo-kv-router@1.1.1` | `v1.82` | |
| `dynamo-kv-router@1.1.0` | `v1.82` | |
| `dynamo-kv-router@1.0.2` | `v1.82` | |
| `dynamo-kv-router@1.0.1` | `v1.82` | |
| `dynamo-kv-router@1.0.0` | `v1.82` | 首发 |

---

## Pre-Release 产物

> [!WARNING]
> **Pre-Release 产物未经 QA 验证。** 预发布版本是面向早期测试与反馈的实验预览，可能存在 bug、破坏性改动或不完整特性。生产工作负载请使用稳定版。

**Pre-Release Python wheel** 发布在 NVIDIA 包索引 [pypi.nvidia.com](https://pypi.nvidia.com/) 上，而非公共 [PyPI](https://pypi.org/)。它们与稳定 wheel 一样，是面向 [Support Matrix](support-matrix.md) 中所列 Python 版本的 **Linux（manylinux）构建**；macOS 或 Windows 上的 `pip`/`uv` 不会找到匹配 wheel。请在受支持的 Linux 主机或 Linux 容器中安装。

通过把该 URL 加为 extra index 并允许预发布版（PEP 440 dev 版本）来安装：

```bash
# uv (recommended in other Dynamo docs)
uv pip install --pre --extra-index-url https://pypi.nvidia.com/ ai-dynamo==1.1.0.dev2

# pip
pip install --pre --extra-index-url https://pypi.nvidia.com ai-dynamo==1.1.0.dev2
```

GitHub 或容器 tag `v1.1.0-dev.N` 对应 wheel 版本 `1.1.0.devN`（例如 `v1.1.0-dev.2` → `==1.1.0.dev2`）。`ai-dynamo[vllm]` 等可选 extra 使用相同 flag；从下方各小节中 pin 你想要的版本。

### v1.2.0-deepseek-v4-dev.3

- **分支：** [release/1.2.0-deepseek-v4-dev.3](https://github.com/ai-dynamo/dynamo/tree/release/1.2.0-deepseek-v4-dev.3)
- **GitHub Tag：** [v1.2.0-deepseek-v4-dev.3](https://github.com/ai-dynamo/dynamo/releases/tag/v1.2.0-deepseek-v4-dev.3)
- **Backend：** vLLM `v0.20.1`（在 `v0.20.0` 原生 DSv4 支持上的 DSv4 稳定化补丁） | SGLang 上游 `lmsysorg/sglang:deepseek-v4-blackwell` 预览（dev.3 已刷新） | NIXL `v0.10.1`
- **覆盖范围：** 部分——仅 DeepSeek-V4-Flash 与 V4-Pro。vLLM 与 SGLang 容器为 Blackwell（B200 + GB200）发布；不包含 TensorRT-LLM 容器、不包含其他组件容器、不包含 Helm chart、不包含 wheel。提供 V4 模型早期访问的快照 dev 构建；未经 QA 验证。

#### 容器镜像

| Image:Tag | Backend | CUDA | 架构 |
|-----------|---------|------|------|
| `vllm-runtime:1.2.0-deepseek-v4-cuda13-dev.3` | vLLM `v0.20.1` | `v13.0` | AMD64/ARM64 |
| `sglang-runtime:1.2.0-deepseek-v4-cuda12-dev.3` | SGLang upstream DSv4 preview | `v12.9` | AMD64 |
| `sglang-runtime:1.2.0-deepseek-v4-cuda13-dev.3` | SGLang upstream DSv4 preview | `v13.0` | ARM64 |

#### Python Wheel

此 dev release 不发布。请使用 `v1.1.1` wheel 或 [pypi.nvidia.com](https://pypi.nvidia.com/) 上的 `v1.1.0-dev.3`。

#### Helm Chart

此 dev release 不发布。平台安装请使用 `v1.1.1` chart。

#### Rust Crate

预发布版本不发布。

### v1.2.0-deepseek-v4-dev.2

- **分支：** [release/1.2.0-deepseek-v4-dev.2](https://github.com/ai-dynamo/dynamo/tree/release/1.2.0-deepseek-v4-dev.2)
- **GitHub Tag：** [v1.2.0-deepseek-v4-dev.2](https://github.com/ai-dynamo/dynamo/releases/tag/v1.2.0-deepseek-v4-dev.2)
- **Backend：** vLLM `v0.20.0`（原生 DeepSeek-V4 支持） | SGLang 上游 `lmsysorg/sglang:deepseek-v4-blackwell` 预览 | NIXL `v0.10.1`
- **覆盖范围：** 仅 DeepSeek-V4-Flash 与 V4-Pro。vLLM 与 SGLang 容器为 Blackwell 发布。此 tag 不发布 TensorRT-LLM 容器、其他组件容器、Helm chart 与 wheel。提供 V4 模型早期访问的快照 dev 构建；未经 QA 验证。

#### 容器镜像

| Image:Tag | Backend | CUDA | 架构 |
|-----------|---------|------|------|
| `vllm-runtime:1.2.0-deepseek-v4-cuda13-dev.2` | vLLM `v0.20.0` | `v13.0` | AMD64/ARM64 |
| `sglang-runtime:1.2.0-deepseek-v4-cuda12-dev.2` | SGLang upstream DSv4 preview | `v12.9` | AMD64 |
| `sglang-runtime:1.2.0-deepseek-v4-cuda13-dev.2` | SGLang upstream DSv4 preview | `v13.0` | ARM64 |

#### Python Wheel

此 dev release 不发布。请使用 `v1.1.0` wheel 或 [pypi.nvidia.com](https://pypi.nvidia.com/) 上的 `v1.1.0-dev.3`。

#### Helm Chart

此 dev release 不发布。平台安装请使用 `v1.1.0` chart。

#### Rust Crate

预发布版本不发布。

### v1.1.0-dev.3

- **分支：** [release/1.1.0-dev.3](https://github.com/ai-dynamo/dynamo/tree/release/1.1.0-dev.3)
- **GitHub Tag：** [v1.1.0-dev.3](https://github.com/ai-dynamo/dynamo/releases/tag/v1.1.0-dev.3)
- **Backend（分支 ToT）：** SGLang `v0.5.10.post1` | TensorRT-LLM `v1.3.0rc11` | vLLM `v0.19.0` | NIXL `v0.10.1`
- **覆盖范围：** TensorRT-LLM runtime 容器以及 [pypi.nvidia.com](https://pypi.nvidia.com/) 上的 **`ai-dynamo`** 与 **`ai-dynamo-runtime`** wheel。此 tag 不发布 SGLang 与 vLLM 容器、组件容器（`dynamo-frontend`、`dynamo-planner`、`kubernetes-operator`、`snapshot-agent`）、**`kvbm`** wheel 与 Helm chart。

#### 容器镜像

| Image:Tag | Backend | CUDA | 架构 |
|-----------|---------|------|------|
| `tensorrtllm-runtime:1.1.0-dev.3` | TRT-LLM `v1.3.0rc11` | `v13.1` | AMD64/ARM64 |

#### Python Wheel

可从 [pypi.nvidia.com](https://pypi.nvidia.com/)（预发布索引）安装：

```bash
uv pip install --pre --extra-index-url https://pypi.nvidia.com/ ai-dynamo==1.1.0.dev3
uv pip install --pre --extra-index-url https://pypi.nvidia.com/ ai-dynamo-runtime==1.1.0.dev3
```

`kvbm==1.1.0.dev3` 尚未发布。

#### Helm Chart

此 dev release 不发布。平台安装请使用最新稳定版（`v1.1.0`）。

#### Rust Crate

预发布版本不发布。

### v1.1.0-dev.2

- **分支：** [release/1.1.0-dev.2](https://github.com/ai-dynamo/dynamo/tree/release/1.1.0-dev.2)
- **GitHub Tag：** [v1.1.0-dev.2](https://github.com/ai-dynamo/dynamo/releases/tag/v1.1.0-dev.2)
- **Backend（分支 ToT）：** SGLang `v0.5.9` | TensorRT-LLM `v1.3.0rc9` | vLLM `v0.19.0` | NIXL `v0.10.1`
- **覆盖范围：** SGLang 与 TensorRT-LLM runtime 容器以及 [pypi.nvidia.com](https://pypi.nvidia.com/) 上的 **`ai-dynamo`** 与 **`ai-dynamo-runtime`** wheel。此 tag 不发布 vLLM runtime 容器、组件容器（`dynamo-frontend`、`dynamo-planner`、`kubernetes-operator`、`snapshot-agent`）、**`kvbm`** wheel 与 Helm chart。

#### 容器镜像

| Image:Tag | Backend | CUDA | 架构 |
|-----------|---------|------|------|
| `sglang-runtime:1.1.0-dev.2` | SGLang `v0.5.9` | `v12.9` | AMD64/ARM64 |
| `tensorrtllm-runtime:1.1.0-dev.2` | TRT-LLM `v1.3.0rc9` | `v13.1` | AMD64/ARM64 |

#### Python Wheel

可从 [pypi.nvidia.com](https://pypi.nvidia.com/)（预发布索引）安装：

```bash
uv pip install --pre --extra-index-url https://pypi.nvidia.com/ ai-dynamo==1.1.0.dev2
uv pip install --pre --extra-index-url https://pypi.nvidia.com/ ai-dynamo-runtime==1.1.0.dev2
```

#### Helm Chart

此 dev release 不发布。平台安装请使用最新稳定版（`v1.1.0`）。

#### Rust Crate

预发布版本不发布。

### v1.1.0-dev.1

- **分支：** [release/1.1.0-dev.1](https://github.com/ai-dynamo/dynamo/tree/release/1.1.0-dev.1)
- **GitHub Tag：** [v1.1.0-dev.1](https://github.com/ai-dynamo/dynamo/releases/tag/v1.1.0-dev.1)
- **Backend：** SGLang `v0.5.9` | TensorRT-LLM `v1.3.0rc5.post1` | vLLM `v0.17.1` | NIXL `v0.10.1`

#### 容器镜像

| Image:Tag | Backend | CUDA | 架构 |
|-----------|---------|------|------|
| `vllm-runtime:1.1.0-dev.1` | vLLM `v0.17.1` | `v12.9` | AMD64/ARM64 |
| `vllm-runtime:1.1.0-dev.1-cuda13` | vLLM `v0.17.1` | `v13.0` | AMD64/ARM64 |
| `vllm-runtime:1.1.0-dev.1-efa-amd64` | vLLM `v0.17.1` | `v12.9` | AMD64 |
| `sglang-runtime:1.1.0-dev.1` | SGLang `v0.5.9` | `v12.9` | AMD64/ARM64 |
| `sglang-runtime:1.1.0-dev.1-cuda13` | SGLang `v0.5.9` | `v13.0` | AMD64/ARM64 |
| `tensorrtllm-runtime:1.1.0-dev.1` | TRT-LLM `v1.3.0rc5.post1` | `v13.1` | AMD64/ARM64 |
| `tensorrtllm-runtime:1.1.0-dev.1-efa-amd64` | TRT-LLM `v1.3.0rc5.post1` | `v13.1` | AMD64 |
| `dynamo-frontend:1.1.0-dev.1` | — | — | AMD64/ARM64 |
| `kubernetes-operator:1.1.0-dev.1` | — | — | AMD64/ARM64 |
| `snapshot-agent:1.1.0-dev.1` | — | — | AMD64/ARM64 |

#### Python Wheel

可从 [pypi.nvidia.com](https://pypi.nvidia.com/)（预发布索引）安装：

```bash
uv pip install --pre --extra-index-url https://pypi.nvidia.com/ ai-dynamo==1.1.0.dev1
uv pip install --pre --extra-index-url https://pypi.nvidia.com/ ai-dynamo-runtime==1.1.0.dev1
```

#### Helm Chart

| Chart | NGC |
|-------|-----|
| `dynamo-platform-1.1.0-dev.1` | [NGC Helm: dynamo-platform 1.1.0-dev.1](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/helm-charts/dynamo-platform?version=1.1.0-dev.1) |
| `snapshot-1.1.0-dev.1` | [NGC Helm: snapshot 1.1.0-dev.1](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/helm-charts/snapshot?version=1.1.0-dev.1) |

#### Rust Crate

预发布版本不发布。
