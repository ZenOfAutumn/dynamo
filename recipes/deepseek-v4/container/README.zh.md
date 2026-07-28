<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# DeepSeek-V4 参考容器

DeepSeek-V4 系列共享的参考 Dockerfile —— 由 [`deepseek-v4-flash`](../deepseek-v4-flash/) 与 [`deepseek-v4-pro`](../deepseek-v4-pro/) 共同使用。两个镜像中没有任何与 recipe 相关的特化内容；模型在运行时通过 `--model-path`（SGLang）选择。

| 后端 | Dockerfile | 基础镜像 | 构建流程 |
|---------|-----------|-----------|------------|
| SGLang (B200)  | [`sglang/Dockerfile.dsv4.sglang.b200`](sglang/Dockerfile.dsv4.sglang.b200)   | `lmsysorg/sglang:deepseek-v4-blackwell`（按 digest 锁定，amd64）       | 两阶段构建；以 Dynamo runtime 镜像作为提供方 |
| SGLang (GB200) | [`sglang/Dockerfile.dsv4.sglang.gb200`](sglang/Dockerfile.dsv4.sglang.gb200) | `lmsysorg/sglang:deepseek-v4-grace-blackwell`（按 digest 锁定，arm64） | 两阶段构建；以 Dynamo runtime 镜像作为提供方 |

NVIDIA 同样发布了 manifest 直接拉取的 vLLM 与 SGLang 预构建镜像：
- `nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.0-deepseek-v4-cuda13-dev.3`（多架构）
- `nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.2.0-deepseek-v4-cuda13-dev.3`（仅 arm64）
- `nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.2.0-deepseek-v4-cuda12-dev.3`（仅 amd64）

`cudaXY` 后缀表示镜像内置的 CUDA 主版本号，而非硬件目标。

> **可选：** 用户也可以通过 `container/render.py` 构建标准的 Dynamo vLLM runtime 镜像。参见 [`<repo_root>/container/README.md`](../../../container/README.md)。

## SGLang (`sglang/Dockerfile.dsv4.sglang.b200`)

两阶段构建：以 Dynamo SGLang runtime 镜像作为提供方（提供 nats / etcd / UCX / NIXL，以及 Dynamo wheels 与 Python 源码），叠加在上游 SGLang dsv4 基础镜像之上。

### 第 1 步 —— 构建 Dynamo SGLang runtime

在**仓库根目录**下执行：

```bash
container/render.py --framework sglang --target runtime --output-short-filename
docker build -t dynamo:latest-sglang-runtime -f container/rendered.Dockerfile .
```

这会生成本地 tag `dynamo:latest-sglang-runtime`，第 2 步会以 `DYNAMO_SRC_IMAGE` 的形式引用它。提供方镜像必须包含 V4 的 tool/reasoning parser 以及 SGLang routed_experts 的修复；构建会通过安装后断言进行校验：`assert 'deepseek_v4' in get_tool_parser_names()`。

runtime 镜像的构建细节与可选 tag 见 [`<repo_root>/container/README.md`](../../../container/README.md)。

### 第 2 步 —— 构建 dsv4 叠加层

仍在**仓库根目录**下执行：

```bash
docker build \
  -f recipes/deepseek-v4/container/sglang/Dockerfile.dsv4.sglang.b200 \
  -t <your-registry>/sglang-dsv4:<tag> \
  .
```

该 Dockerfile 不依赖任何构建上下文（所有内容都来自 `FROM` / `COPY --from=`），因此任意上下文目录均可。

### 构建参数

| 参数 | 默认值 | 用途 |
|-----|---------|---------|
| `DYNAMO_SRC_IMAGE` | `dynamo:latest-sglang-runtime` | 提供 nats / etcd / UCX / NIXL 以及支持 V4 的 Dynamo wheel。默认值与第 1 步一致；如需可复现构建且无需本地重建，可覆盖为已发布的 Dynamo SGLang runtime tag。 |
| `DSV4_BASE_IMAGE`  | `lmsysorg/sglang:deepseek-v4-blackwell@sha256:da2acdc8...` | DeepSeek-V4 SGLang 基础镜像。按 digest 锁定以保证字节稳定的重建。 |

### 接入 recipe

推送：

```bash
docker push <your-registry>/sglang-dsv4:<tag>
```

在 recipe 的 SGLang manifest 中设置 `image:` 字段（Frontend 与 decode worker），然后参照 recipe 的 Quick Start：

- Flash → [`../deepseek-v4-flash/sglang/agg/deploy.yaml`](../deepseek-v4-flash/sglang/agg/deploy.yaml) —— 见 [Quick Start](../deepseek-v4-flash/README.md#quick-start)。
- Pro → [`../deepseek-v4-pro/sglang/agg/deploy.yaml`](../deepseek-v4-pro/sglang/agg/deploy.yaml) —— 见 [Quick Start](../deepseek-v4-pro/README.md#quick-start)。
