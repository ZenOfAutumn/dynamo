---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 推测解码
---

推测解码是一种优化技术，使用一个较小的"draft"模型预测多个 token，然后由主模型并行验证。这可以显著降低自回归生成的延迟。

## 后端支持

| 后端 | 状态 | 备注 |
|---------|--------|-------|
| vLLM | ✅ | 支持 Eagle3 draft 模型 |
| SGLang | 🚧 | 文档尚未提供 |
| TensorRT-LLM | 🚧 | 文档尚未提供 |

## 概述

推测解码的工作方式：

1. **Draft 阶段**：一个更小、更快的模型生成候选 token
2. **Verify 阶段**：主模型在一次前向传播中验证这些候选
3. **接受 / 拒绝**：如果候选与主模型本应生成的一致，则被接受

这种方法以额外的算力换取更低的延迟，因为每次主模型前向传播可以生成多个 token。

## 快速开始（vLLM + Eagle3）

本指南介绍如何在至少 16GB 显存的单 GPU 上，使用 **Eagle3** 推测解码部署 **Meta-Llama-3.1-8B-Instruct**。

### 前置条件

1. 启动基础设施服务：

```bash
docker compose -f deploy/docker-compose.yml up -d
```

2. 构建并运行 vLLM 容器：

```bash
./container/build.sh --framework VLLM
./container/run.sh -it --framework VLLM --mount-workspace
```

3. 设置 Hugging Face 访问（Meta-Llama-3.1-8B-Instruct 是受限模型）：

```bash
export HUGGING_FACE_HUB_TOKEN="your_token_here"
export HF_TOKEN=$HUGGING_FACE_HUB_TOKEN
```

### 运行推测解码

```bash
cd examples/backends/vllm
bash launch/agg_spec_decoding.sh
```

### 测试部署

```bash
curl http://localhost:8000/v1/chat/completions \
   -H "Content-Type: application/json" \
   -d '{
     "model": "meta-llama/Meta-Llama-3.1-8B-Instruct",
     "messages": [
       {"role": "user", "content": "Write a poem about why Sakura trees are beautiful."}
     ],
     "max_tokens": 250
   }'
```

## 后端专项指南

| 后端 | 指南 |
|---------|-------|
| vLLM | [speculative_decoding_vllm.md](./speculative-decoding-vllm.md) |

## 另请参阅

- [vLLM 后端](../../backends/vllm/README.md) - 完整 vLLM 部署指南
- [分离式服务](../../design-docs/disagg-serving.md) - 另一种优化方式
- [Hugging Face 上的 Meta-Llama-3.1-8B-Instruct](https://huggingface.co/meta-llama/Llama-3.1-8B-Instruct)
