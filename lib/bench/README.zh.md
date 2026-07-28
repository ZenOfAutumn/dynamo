<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Bench 入口

`multiturn_bench` 模拟对一个 OpenAI 兼容聊天端点发起并发的多轮会话，并报告每轮的 TTFT 和总时延统计。它可选地启用 **speculative prefill（推测性预填充）**——一种在每次助手响应后用预测的下一轮前缀预热 KV 缓存的技术，可降低后续轮次的 TTFT。

`offline_replay_bench` 直接运行 Rust 原生的 replay 循环，用于剖析与吞吐量测量，绕过 Python 包装层。它使用 mocker 内部的多项式性能模型，因此结果聚焦在 replay 开销上，而不受外部计时后端影响。

## 快速开始

```bash
# 冒烟测试（1 个用户、1 轮、约 50 tokens）
cargo bench --package dynamo-bench --bench multiturn_bench -- --ping
```

## Speculative prefill 演示

speculative prefill 在多轮会话工作负载下效果最佳，对话会逐步增长（例如 agentic 循环中的推理模型）。在每个助手轮次结束后，前端构造下一轮 prompt 前缀，并发送一个 `max_tokens=1` 请求来预热 KV 缓存，使真实的后续请求命中预热的缓存，从而显著降低 TTFT。

### 1. 启动后端与前端

```bash
# 终端 1 — 后端（vLLM 示例，任意受支持后端均可）
python -m dynamo.vllm \
  --model deepseek-ai/DeepSeek-R1-Distill-Llama-8B

# 终端 2 — 带 KV router 的前端
python -m dynamo.frontend \
  --router-mode kv \
  --http-port 8000
```

### 2. 运行基线（无 speculative prefill）

```bash
cargo bench --package dynamo-bench --bench multiturn_bench -- \
  --url http://localhost:8000 \
  --num-users 10 \
  --num-turns 5 \
  --num-user-tokens 128 \
  --max-completion-tokens 256 \
  --mean-delay-ms 5000 \
  --output baseline.json \
  --verbose
```

### 3. 启用 speculative prefill 运行

```bash
cargo bench --package dynamo-bench --bench multiturn_bench -- \
  --url http://localhost:8000 \
  --num-users 10 \
  --num-turns 5 \
  --num-user-tokens 128 \
  --max-completion-tokens 256 \
  --mean-delay-ms 5000 \
  --speculative-prefill \
  --output specprefill.json \
  --verbose
```

对比每轮 TTFT 列：第 2 轮及之后的 TTFT 应有显著下降（最多约 3 倍），因为真实请求到达时 KV 缓存已经预热。

## CLI 参考

| Flag | 默认值 | 描述 |
|------|---------|-------------|
| `--url` | `http://localhost:8000` | 前端 HTTP 端点 |
| `--model` | 自动检测 | 模型名（省略时查询 `/v1/models`） |
| `--num-users` | `10` | 并发模拟用户数 |
| `--num-turns` | `5` | 每个用户的会话轮数 |
| `--num-user-tokens` | `128` | 每轮用户 prompt 的近似 token 数 |
| `--max-completion-tokens` | `1000` | 输出序列长度上限 |
| `--ignore-eos` | `true` | 强制生成到 max tokens |
| `--mean-delay-ms` | `5000` | 轮次间平均延迟（指数分布） |
| `--speculative-prefill` | `false` | 通过 `nvext.agent_hints` 启用 speculative prefill |
| `--output <path>` | 无 | 将结果写入 JSON 文件 |
| `--verbose` / `-v` | `false` | 打印每轮日志 |
| `--seed` | `42` | 随机种子 |
| `--ping` | `false` | 冒烟测试模式（1 用户、1 轮、约 50 tokens、无延迟） |

## Speculative prefill 工作原理

1. 客户端在每个请求中发送 `{"nvext": {"agent_hints": {"speculative_prefill": true}}}`。
2. 助手响应流式返回时，前端累积完整的响应文本。
3. 一旦设置了 `finish_reason`，一个后台任务构造下一轮 prompt（会话历史 + 助手响应，去除 thinking 内容），并通过管道发送一个 `max_tokens=1` 的仅预填充请求。
4. KV 路由将该 speculative 请求路由到同一 worker，预热其缓存。
5. 当真实的下一轮请求到来时，KV 路由发现该 worker 上有较高缓存重叠，路由到那里，从而获得显著更低的 TTFT。

另见：[Agent Hints 文档](../../docs/components/frontend/nvext.md#agent-hints)

## 离线 replay

```bash
cargo bench --package dynamo-bench --bench offline_replay_bench -- \
  /path/to/mooncake_trace.jsonl \
  --num-workers 4 \
  --router-mode kv-router \
  --arrival-speedup-ratio 4 \
  --trace-block-size 512 \
  --block-size 64
```

如果你希望在保留同一内部多项式模型的同时使用一个简单的缩放旋钮，可使用 `--speedup-ratio` 与 `--decode-speedup-ratio`。

## KV router / sharded indexer 基准

参见 [kv_router/INDEXER_BENCH.md](kv_router/INDEXER_BENCH.md) 了解 trace 获取、基准命令以及 `mooncake_bench` 套件的结果：
`concurrent-radix-tree-compressed`、`branch-sharded-crtc`、`anchor-aware-branch-sharded-crtc`。
