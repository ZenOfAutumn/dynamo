---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Llama4 + Eagle
---

本指南演示如何在 GB200x4 节点上部署带 Eagle 推测解码的 Llama 4 Maverick Instruct。我们将参照[多节点部署说明](./multinode/trtllm-multinode-examples.md)来为以下场景搭建环境：

- **聚合式服务：**
  在单个 GB200x4 节点上部署整个 Llama 4 模型，进行端到端服务。

- **分离式服务：**
  将工作负载分布在两个 GB200x4 节点上：
    - 一个节点运行解码 worker。
    - 另一个节点运行预填充 worker。

## 注意
* 请确保在 `examples/backends/trtllm/engine_configs/llama4/eagle` 目录下的 LLM API 配置中设置了 (`eagle3_one_model: true`)。

## 准备

假设你已经通过 `salloc` 分配了节点，并且已在其中一个分配节点的交互式 shell 中，请基于实际情况设置以下环境变量：

```bash
cd $DYNAMO_HOME/examples/backends/trtllm

export IMAGE="<dynamo_trtllm_image>"
# export MOUNTS="${PWD}/:/mnt,/lustre:/lustre"
export MOUNTS="${PWD}/:/mnt"
export MODEL_PATH="nvidia/Llama-4-Maverick-17B-128E-Instruct-FP8"
export SERVED_MODEL_NAME="nvidia/Llama-4-Maverick-17B-128E-Instruct-FP8"
```

更多关于以上选项的说明，请参阅[多节点准备说明](./multinode/trtllm-multinode-examples.md#setup)。


## 聚合式服务
```bash
export NUM_NODES=1
export ENGINE_CONFIG="/mnt/examples/backends/trtllm/engine_configs/llama4/eagle/eagle_agg.yml"
./multinode/srun_aggregated.sh
```

## 分离式服务

```bash
export NUM_PREFILL_NODES=1
export PREFILL_ENGINE_CONFIG="/mnt/examples/backends/trtllm/engine_configs/llama4/eagle/eagle_prefill.yml"
export NUM_DECODE_NODES=1
export DECODE_ENGINE_CONFIG="/mnt/examples/backends/trtllm/engine_configs/llama4/eagle/eagle_decode.yml"
./multinode/srun_disaggregated.sh
```

## 示例请求

请参阅[示例请求章节](./multinode/trtllm-multinode-examples.md#example-request)，了解如何向部署发送请求。

```
curl localhost:8000/v1/chat/completions -H "Content-Type: application/json" -d '{
        "model": "nvidia/Llama-4-Maverick-17B-128E-Instruct-FP8",
        "messages": [{"role": "user", "content": "Explain why Roger Federer is considered one of the greatest tennis players of all time"}],
        "max_tokens": 1024
    }' -w "\n"


# output:
{"id":"cmpl-3e87ea5c-010e-4dd2-bcc4-3298ebd845a8","choices":[{"text":"NVIDIA is considered a great company for several reasons:\n\n1. **Technological Innovation**: NVIDIA is a leader in the field of graphics processing units (GPUs) and has been at the forefront of technological innovation.
...
and the broader tech industry.\n\nThese factors combined have contributed to NVIDIA's status as a great company in the technology sector.","index":0,"logprobs":null,"finish_reason":"stop"}],"created":1753329671,"model":"nvidia/Llama-4-Maverick-17B-128E-Instruct-FP8","system_fingerprint":null,"object":"text_completion","usage":{"prompt_tokens":16,"completion_tokens":562,"total_tokens":578,"prompt_tokens_details":null,"completion_tokens_details":null}}
```
