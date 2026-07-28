---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Router 测试
subtitle: 路由器变更对应的测试层
---

## 概述

路由器有三个有用的测试层。当你新增一个非平凡或可能引入破坏性变更的特性时，默认不要止步于最小的本地测试。可考虑在下面相关的层级上扩展，以便变更在可能出错的同等层级上得到覆盖。

## 1. Rust 单元与集成测试

在 `lib/kv-router` 与 `lib/llm` 中使用 Rust 测试进行本地正确性验证：

- 成本模型计算
- indexer 行为
- 事件应用
- 恢复与持久化逻辑
- 序列跟踪不变量
- 远程 indexer 查询行为

这些测试应是范围较窄的逻辑的第一道防线。它们最适合用来固定靠近实现的精确边界情况和回归。

示例：

- `lib/kv-router/src/indexer/tests.rs`
- `lib/kv-router/src/sequences/*`
- `lib/llm/src/kv_router/indexer/worker_query.rs`

典型命令：

```bash
cargo test -p dynamo-kv-router
cargo test -p dynamo-llm --no-default-features
```

## 2. 基于 Bench 的端到端不变量测试

当你希望走真实的 replay 路径但又不想启动整个路由器栈时，使用 `lib/bench/tests` 中基于 fixture 的测试。这些测试与 Mooncake 和 active-sequences 基准共享同一套 replay 机器，但运行在 Rust 测试 profile 中，断言不变量而不是报告基准数字。

当前覆盖使用了仓库内 1000 行的 Mooncake trace fixture：

- [active_sequences_trace.rs](../../../lib/bench/tests/active_sequences_trace.rs)
- [mooncake_trace.rs](../../../lib/bench/tests/mooncake_trace.rs)
- [mooncake_trace_1000.jsonl](../../../lib/bench/testdata/mooncake_trace_1000.jsonl)

这些测试可用于捕获以下回归：

- replay 结束时状态未排空
- 热路径上出现意外的 `WARN` 或 `ERROR` 日志
- 重复存储或类似的告警指标
- 不同 indexer 实现之间的 replay 行为差异

典型命令：

```bash
cargo test --package dynamo-bench --all-targets
```

当一个特性会随时间改变路由器行为、依赖真实事件顺序，或应在多个 indexer 实现之间保持时，使用此层。

## 3. 完整路由器端到端进程测试

当需要请求平面与事件平面同时发挥作用时，使用 `tests/router` 中的 Python 测试。这些测试启动路由器与 mocker 或后端进程，演练 bench-backed replay 无法覆盖的跨进程行为。

当前的入口包括：

- [test_router_e2e_with_mockers.py](../../../tests/router/test_router_e2e_with_mockers.py)
- [test_router_e2e_with_vllm.py](../../../tests/router/test_router_e2e_with_vllm.py)
- [test_router_e2e_with_trtllm.py](../../../tests/router/test_router_e2e_with_trtllm.py)
- [test_router_e2e_with_sglang.py](../../../tests/router/test_router_e2e_with_sglang.py)

在以下变更涉及时使用此层：

- 进程边界
- 通过 Dynamo 前端或路由器服务的请求路由
- worker 注册与发现
- 事件平面的传输与投递
- 后端集成行为
- 启动、恢复或生命周期流程

典型命令：

```bash
.venv/bin/python -m pytest tests/router/test_router_e2e_with_mockers.py
```

## 推荐用法

当一项路由器变更非平凡或可能破坏行为时，建议默认按以下顺序考虑：

- 为本地逻辑添加或更新 Rust 单元测试。
- 如果变更影响 replay 顺序、indexer 行为、缓存事件处理或状态排空假设，添加或更新基于 bench 的不变量测试。
- 如果变更依赖真实进程、传输、注册或后端交互，添加或更新完整的 `tests/router` E2E 测试。

并非每项变更都需要全部三个层级。但如果一项变更可能在单个模块边界外破坏行为，通常值得超越单元测试。
