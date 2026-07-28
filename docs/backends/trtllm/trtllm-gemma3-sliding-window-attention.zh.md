---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Gemma3 滑动窗口
---

有关 TensorRT-LLM 的通用特性与配置，请参阅[参考指南](trtllm-reference-guide.md)。

---

本指南演示如何使用 Dynamo 部署带有可变滑动窗口注意力（VSWA）的 google/gemma-3-1b-it。由于 google/gemma-3-1b-it 是一个较小的模型，每个聚合（aggregated）、解码（decode）或预填充（prefill）的 worker 仅需要一块 H100 GPU 或一块 GB200 GPU。
VSWA 是一种让模型各层在多种滑动窗口大小之间交替的机制。Gemma 3 即是一个例子，它同时包含全局注意力层和滑动窗口层。

> [!Note]
> - 启动前，请确保 `nats` 和 `etcd` 等所需服务已运行。
> - 在 Hugging Face 上申请 `google/gemma-3-1b-it` 的访问权限，并设置 `HF_TOKEN` 环境变量用于身份验证。

## 聚合式服务
```bash
cd $DYNAMO_HOME/examples/backends/trtllm
export MODEL_PATH=google/gemma-3-1b-it
export SERVED_MODEL_NAME=$MODEL_PATH
export AGG_ENGINE_ARGS=$DYNAMO_HOME/examples/backends/trtllm/engine_configs/gemma3/vswa_agg.yaml
./launch/agg.sh
```

## 带 KV 路由的聚合式服务
```bash
cd $DYNAMO_HOME/examples/backends/trtllm
export MODEL_PATH=google/gemma-3-1b-it
export SERVED_MODEL_NAME=$MODEL_PATH
export AGG_ENGINE_ARGS=$DYNAMO_HOME/examples/backends/trtllm/engine_configs/gemma3/vswa_agg.yaml
./launch/agg_router.sh
```

## 分离式服务
```bash
cd $DYNAMO_HOME/examples/backends/trtllm
export MODEL_PATH=google/gemma-3-1b-it
export SERVED_MODEL_NAME=$MODEL_PATH
export PREFILL_ENGINE_ARGS=$DYNAMO_HOME/examples/backends/trtllm/engine_configs/gemma3/vswa_prefill.yaml
export DECODE_ENGINE_ARGS=$DYNAMO_HOME/examples/backends/trtllm/engine_configs/gemma3/vswa_decode.yaml
./launch/disagg.sh
```

## 带 KV 路由的分离式服务
```bash
cd $DYNAMO_HOME/examples/backends/trtllm
export MODEL_PATH=google/gemma-3-1b-it
export SERVED_MODEL_NAME=$MODEL_PATH
export PREFILL_ENGINE_ARGS=$DYNAMO_HOME/examples/backends/trtllm/engine_configs/gemma3/vswa_prefill.yaml
export DECODE_ENGINE_ARGS=$DYNAMO_HOME/examples/backends/trtllm/engine_configs/gemma3/vswa_decode.yaml
./launch/disagg_router.sh
```
