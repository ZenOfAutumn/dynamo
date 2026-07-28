---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: DP Rank 路由（注意力数据并行）
---

有关 TensorRT-LLM 的通用特性与配置，请参阅[参考指南](trtllm-reference-guide.md)。

---

TensorRT-LLM 为 DeepSeek 等模型支持[注意力数据并行](https://lmsys.org/blog/2024-12-04-sglang-v0-4/#data-parallelism-attention-for-deepseek-models)（attention DP）。启用后，多个注意力 DP rank 会运行在同一个 worker 内，每个 rank 拥有自己的 KV 缓存。Dynamo 可以根据 KV 缓存状态将请求路由到特定的 DP rank。

### Dynamo 路由 vs TRT-LLM 内部路由

- **Dynamo DP Rank 路由**：路由器根据 KV 缓存重叠选择最优的 DP rank，并指示 TRT-LLM 使用严格路由（`attention_dp_relax=False`）使用该 rank。可与 `--router-mode kv` 配合使用，以实现缓存感知的路由。
- **TRT-LLM 内部路由**：TRT-LLM 的调度器在内部分配 DP rank。当不需要 KV 感知路由时，可与 `--router-mode round-robin` 或 `random` 配合使用。

### 启用 DP Rank 路由

```bash
# Worker with attention DP
# (TP=2 acts as the "world size", in effect creating 2 attention DP ranks)
CUDA_VISIBLE_DEVICES=0,1 python3 -m dynamo.trtllm \
  --model-path <MODEL_PATH> \
  --tensor-parallel-size 2 \
  --enable-attention-dp \
  --publish-events-and-metrics

# Frontend with KV routing
python3 -m dynamo.frontend --router-mode kv
```

`--enable-attention-dp` 标志将 `attention_dp_size` 设为 `tensor_parallel_size`，并配置 Dynamo 按 DP rank 发布 KV 事件。路由器会自动为每个 `(worker_id, dp_rank)` 组合创建路由目标。

<Note>
注意力 DP 需要 TRT-LLM 的 PyTorch 后端。AutoDeploy 不支持注意力 DP。
</Note>
