---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: KV 事件回放 — Dynamo vs vLLM
subtitle: 两个系统如何处理 KV 缓存事件的间隙检测、回放与恢复
---

## 概述

Dynamo 与 vLLM 都通过 fire-and-forget 传输（ZMQ PUB/SUB）发布 KV 缓存事件（block stored、block removed 等）。由于 PUB/SUB 是有损的，两者都需要让消费者检测丢失的消息并恢复的机制。本文比较两种方法。

## 问题

KV 事件消费者（路由器、缓存协调器）订阅来自 worker 的实时 block 事件流。事件携带单调递增的序列号。当消费者检测到序列中的间隙（例如收到 seq 42 后是 seq 45），它需要恢复丢失的事件，否则将拥有该 worker KV 缓存状态的过时、错误视图。

## 架构对比

| | vLLM Replay Buffer | Dynamo Local Indexer |
|---|---|---|
| **核心 buffer** | `collections.deque[tuple[int, bytes]]`，带 `maxlen` | `VecDeque<RouterEvent>`，带 `max_buffer_size` |
| **Buffer 语义** | FIFO 环，旧条目静默丢弃 | FIFO 环，旧条目静默丢弃 |
| **事件顺序** | 单调序列号（8 字节 int） | 单调 `event_id`，带连续 ID 校验 |
| **查找** | 线性扫描（`for seq, buf in buffer`） | 二分查找（`binary_search_by_key`） |
| **序列化** | buffer 中存储预序列化的 msgpack 字节 | 存储结构化事件；按需序列化 |
| **buffer 太旧时的回退** | 消费者必须从外部重建 | 完整 RadixTree 状态的 tree dump |
| **初始同步** | 未内置——消费者从实时流开始 | Tree dump（请求 `start_event_id=None`） |
| **权威状态** | 仅 buffer | RadixTree（buffer 是优化层） |
| **压缩 / 去重** | 事件按原样存储（预序列化） | RadixTree 压缩跨序列共享的前缀 |
| **过期** | 通过 `maxlen` 驱逐隐式实现 | 通过 `PruneManager` 进行 TTL 过期 |
| **传输** | ZMQ PUB/SUB + ROUTER/REQ | Dynamo 服务 RPC（请求 / 响应） |
| **多 rank** | 每 DP rank 端口偏移 | 每 DP rank 单独的查询端点 |
| **线程模型** | 后台线程 + 队列 | 在专用 OS 线程上的单线程 tokio runtime |
| **投递保证** | 至少一次（消费者去重） | 至少一次（路由器通过事件 ID 跟踪去重） |
| **去重责任** | 消费者必须按 seq 号过滤 | 在 indexer 基础设施中处理 |

## 各系统如何工作

### vLLM：仅 Buffer 回放

vLLM 的 `ZmqEventPublisher`（位于 `vllm/distributed/kv_events.py`）在后台线程中运行两个 ZMQ socket：

1. **PUB socket**（默认 `tcp://*:5557`）：流式发送带单调序列号的 `KVEventBatch` 消息。
2. **ROUTER socket**（可选，例如 `tcp://*:5558`）：处理消费者的回放请求。

publisher 维护一个 `deque`，保留最近 `buffer_steps`（默认 10,000）个序列化批次。当消费者检测到间隙时，会将缺失的起始序列号发送到 ROUTER socket。publisher 线性扫描 buffer 并从该序列号开始回放所有批次，并以哨兵（`seq=-1, payload=empty`）结束。

**取舍：**
- 轻量——除 buffer 本身外没有额外状态；易于理解与部署。
- 如果间隙比 buffer 窗口更早，消费者必须通过其他方式（例如重启并重新发现）重建状态。
- 没有内置的初始状态同步——在事件已经发布后接入的消费者从空视图开始。
- 每次回放请求都要做线性扫描（buffer 内无索引）。
- 消费者通过检查 `replay_seq > last_seq` 实现去重。

### Dynamo：Buffer + Indexer，并以 tree dump 作为回退

Dynamo 的 `LocalKvIndexer`（位于 `lib/kv-router/src/indexer/local.rs`）将一个 `KvIndexer`（由 `RadixTree` 支撑）与一个循环事件缓冲区结合：

```text
LocalKvIndexer
├── indexer: KvIndexer          // Authoritative state (RadixTree)
├── event_buffer: VecDeque      // Circular buffer for fast replay
└── max_buffer_size: usize
```

当路由器通过 `get_events_in_id_range(start_id, end_id)` 向 worker 查询事件时，本地 indexer 返回三种响应之一：

| 响应 | 何时 | 发生什么 |
|----------|------|--------------|
| `Events` | 请求范围在 buffer 内 | 直接返回缓冲事件（用二分查找定位切片边界） |
| `TreeDump` | 范围太旧或初始同步（`start_id=None`） | 将整棵 RadixTree 作为合成事件 dump 出来——完整状态快照 |
| `TooNew` | 消费者超前于生产者 | 错误响应；没有间隙需要填补 |

tree dump 回退意味着，当 buffer 无法满足请求时，indexer 退而 dump 完整的树状态。这使 "buffer 太旧" 成为可恢复的情况，但代价是额外的复杂度和维护树所需的内存。

## 间隙检测

两个系统检测间隙的方式相同：消费者跟踪它处理的最后序列号 / 事件 ID，并与下一个收到的对比。

**vLLM**（来自 `examples/online_serving/kv_events_subscriber.py`）：
```python
if last_seq >= 0 and seq > last_seq + 1:
    missed = seq - last_seq - 1
    replay.send((last_seq + 1).to_bytes(8, "big"))
    # ... receive and process replayed events
```

**Dynamo**（来自 `lib/llm/src/kv_router/worker_query.rs`）：
路由器对每个 worker 跟踪 `last_recovered_event_id`，并在检测到间隙或初始发现时请求 `recover_from_worker(worker_id, dp_rank, start_event_id, end_event_id)`。本地 indexer 处理决定从 buffer 回放还是 dump 整棵树的复杂逻辑。

## 何时使用哪一个

**vLLM 内置回放**适合以下情况：
- 你在独立运行 vLLM，希望在不引入额外基础设施的前提下进行基本的间隙恢复。
- 你的消费者长生命周期，很少断开——主要担心的是瞬时间隙。
- 你正在构建自定义的外部路由器或缓存协调器，希望直接消费 vLLM 的 KV 事件，不再用其他框架包装。

**Dynamo 的 local indexer**适合以下情况：
- 你需要稳健的恢复，包括为新加入的路由器或离线较久的消费者进行初始状态同步。
- 你正在运行多个路由器副本，可能在不同时间启动，需要对缓存状态收敛到一致视图。
- 你希望去重与恢复由基础设施处理，而不是在每个消费者中实现。

两种方法共享同一个核心思想——使用 FIFO 环形缓冲区赶上小的瞬时间隙。Dynamo 在其下加了一棵 RadixTree 作为权威状态，使得在 buffer 不够时通过 tree dump 实现完整状态恢复，代价是额外内存与复杂度。vLLM 仅保留 buffer，已足以应对消费者稳定且间隙短暂的场景。

对于使用 Dynamo KV 感知路由的部署，会自动使用 local indexer。对于希望构建自己事件消费者的独立 vLLM 部署，vLLM 的 replay buffer 提供了一个轻量的起点。

## 另请参阅

- **[KV Router Index 数据结构](https://github.com/ai-dynamo/dynamo/blob/main/lib/kv-router/src/indexer/README.md)**：`RadixTree`、`ConcurrentRadixTree` 与 `PositionalIndexer` 内部细节
- **[Router 指南](router-guide.md)**：KV 感知路由的部署模式与快速上手
- **[配置与调优](router-configuration.md)**：路由器标志与调优细节
- **[Router 设计](../../design-docs/router-design.md)**：架构细节与事件传输模式
