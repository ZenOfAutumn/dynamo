---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 构建自定义 TensorRT-LLM 容器
---

如需使用预构建容器，请参阅 [TensorRT-LLM 快速开始](README.md#quick-start)。

## 构建自定义容器

如果你需要从源码构建容器（例如，用于自定义修改或不同的 CUDA 版本）：

```bash
# TensorRT-LLM uses git-lfs, which needs to be installed in advance.
apt-get update && apt-get -y install git git-lfs

# On an x86 machine:
python container/render.py --framework=trtllm --target=runtime --output-short-filename --cuda-version=13.1
docker build -t dynamo:trtllm-latest -f container/rendered.Dockerfile .

# On an ARM machine:
python container/render.py --framework=trtllm --target=runtime --platform=arm64 --output-short-filename --cuda-version=13.1
docker build -t dynamo:trtllm-latest -f container/rendered.Dockerfile .
```

运行自定义容器：

```bash
./container/run.sh --framework trtllm -it
```
