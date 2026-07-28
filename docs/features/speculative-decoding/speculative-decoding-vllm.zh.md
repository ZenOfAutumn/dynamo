---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Speculative Decoding with vLLM
---

在 vLLM 后端（backend）中使用推测解码（speculative decoding）。

> **另见**：[推测解码概览](./README.md)（跨后端文档）。

## 前置条件

- 支持 Eagle3 的 vLLM 容器
- 至少有 16GB 显存的 GPU
- Hugging Face 访问 token（用于受限模型）

## 快速开始：Meta-Llama-3.1-8B-Instruct + Eagle3

本指南演示如何在单节点上部署 **Meta-Llama-3.1-8B-Instruct**，并启用 **Eagle3** 推测解码。

### 第 1 步：搭建 Docker 环境

首先，使用 vLLM 后端初始化一个 Docker 容器。详见 [vLLM 快速上手指南](../../backends/vllm/README.md#vllm-quick-start)。

```bash
# Launch infrastructure services
docker compose -f deploy/docker-compose.yml up -d

# Build the container
./container/build.sh --framework VLLM

# Run the container
./container/run.sh -it --framework VLLM --mount-workspace
```

### 第 2 步：获取对 Llama-3 模型的访问权限

**Meta-Llama-3.1-8B-Instruct** 是受限模型。请在 Hugging Face 上申请访问：
[Meta-Llama-3.1-8B-Instruct 仓库](https://huggingface.co/meta-llama/Llama-3.1-8B-Instruct)

审核时间取决于 Hugging Face 当前的处理量。

获批后，在容器内设置访问 token：

```bash
export HUGGING_FACE_HUB_TOKEN="insert_your_token_here"
export HF_TOKEN=$HUGGING_FACE_HUB_TOKEN
```

### 第 3 步：运行聚合（aggregated）模式的推测解码

```bash
# Requires only one GPU
cd examples/backends/vllm
bash launch/agg_spec_decoding.sh
```

权重下载完成后，服务即可接收推理（inference）请求。

### 第 4 步：测试部署

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

### 输出示例

```json
{
  "id": "cmpl-3e87ea5c-010e-4dd2-bcc4-3298ebd845a8",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "In cherry blossom's gentle breeze ... A delicate balance of life and death, as petals fade, and new life breathes."
      },
      "index": 0,
      "finish_reason": "stop"
    }
  ],
  "model": "meta-llama/Meta-Llama-3.1-8B-Instruct",
  "usage": {
    "prompt_tokens": 16,
    "completion_tokens": 250,
    "total_tokens": 266
  }
}
```

## 配置

vLLM 中的推测解码使用 Eagle3 作为草稿模型（draft model）。启动脚本配置了：

- 目标模型：`meta-llama/Meta-Llama-3.1-8B-Instruct`
- 草稿模型：Eagle3 变体
- 聚合（aggregated）服务模式

完整配置参见 `examples/backends/vllm/launch/agg_spec_decoding.sh`。

## 限制

- 目前仅支持以 Eagle3 作为草稿模型
- 要求目标模型与草稿模型的架构相互兼容

## 另见

| 文档 | 路径 |
|----------|------|
| 推测解码概览 | [README.md](./README.md) |
| vLLM 后端指南 | [vLLM README](../../backends/vllm/README.md) |
| Meta-Llama-3.1-8B-Instruct | [Hugging Face](https://huggingface.co/meta-llama/Llama-3.1-8B-Instruct) |
