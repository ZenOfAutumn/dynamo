---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
subtitle: "Ishan Dhanani and Matej Kosec — March 2026"
description: "How Dynamo optimizes for agentic workloads at three layers: frontend API, router, and KV cache management."
keywords: agentic inference, KV cache, prefix caching, agent hints, disaggregated serving, Dynamo
last-updated: March 10, 2026
---

# 用 Dynamo 全栈优化 Agentic 推理

编码 Agent（Coding agent）开始大规模地写生产代码。[Stripe 的 Agent 每周生成 1300+ 个 PR](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents)。[Ramp 把 30% 已合并 PR 归功于 Agent](https://www.infoq.com/news/2026/01/ramp-coding-agent-platform/)。[Spotify 报告每月 650+ 个由 Agent 生成的 PR](https://engineering.atspotify.com/2025/11/spotifys-background-coding-agent-part-1)。Claude Code 和 Codex 这类工具，每个编码会话都会发起数百次 API 调用，每次都带着完整对话历史。在每一条这样的工作流背后，都是一套承受着巨大 KV 缓存（KV cache）压力的推理（inference）栈。

![Cumulative cache reads vs writes across a 42-call Claude Code session. Cache reads (891K tokens) grow steeply while writes (76K) and uncached input stay flat -- an 11.7x read/write ratio.](./cumulative-reads-writes.png)

以 Claude Code 为例。第一次 API 调用把对话前缀写入 KV 缓存后，后续每次发往同一 worker 的调用都能命中 85–97% 的缓存。Agent 团队（或 Swarm）能进一步把这一数字推到 4 个 Opus 协作成员之间 97.2% 的总体命中率。11.7 倍的读/写比意味着系统每写一个 token，几乎从缓存中读取了 12 次。这是一种"一次写入、多次读取（write-once-read-many，WORM）"的访问模式：系统提示和不断增长的对话前缀只计算一次，之后每次调用都从缓存中提供。让所有 worker 间的缓存复用率最大化、并保持 KV 块温热可路由，是 Agentic 推理的核心优化目标。

这些数字来自托管 API 基础设施，由服务提供商控制前缀匹配、缓存放置与驱逐。对于在自有 GPU 上运行开源模型的团队而言，这些都不是开箱即用的。我们一直在打造 Dynamo 来弥合这一差距。本文将介绍我们如何在三个层面把 Dynamo 打造成 Agent 原生：前端 API、路由（router）、以及 KV 缓存管理。

文中我们一致使用三个术语：
- **Harness**：驱动工作流的 Agent 框架（Claude Code、Codex、OpenClaw、OpenCode 等）
- **Orchestrator**：Dynamo 的路由、调度与缓存管理层
- **Runtime**：执行模型并掌管 KV 缓存管理器的推理引擎（SGLang、vLLM、TRT-LLM）

## 第 1 层：前端

### 多协议支持

Agent harness 越来越多地从 `v1/chat/completions` 转向 `v1/responses` 和 `v1/messages`，以更干净地处理交错思考（interleaved thinking）和工具调用等新模式。这些 API 的关键差异是结构性的。在 `v1/chat/completions` 中，消息内容是扁平字符串，工具调用作为单独字段附加。例如，注意 [GLM](https://docs.z.ai/guides/capabilities/thinking-mode#example-usage) 和 [MiniMax](https://platform.minimax.io/docs/guides/text-m2-function-call#important-note) 在通过 `v1/chat/completions` 端点托管模型时，对交错思考的处理方式各不相同。`v1/responses` 和 `v1/messages` 使用类型化的内容块（content block），所以一个 assistant 轮次中可以将思考、工具调用和文本作为独立对象同时存在。这对推理很重要：orchestrator 能看到块的边界，进而做 prompt 优化，并按块类型应用不同的缓存与调度策略。Dynamo 通过统一的内部表示同时服务这三个端点，因此一次部署就能作为任意 harness 的推理后端。我们团队内部一直在用 Dynamo 部署 GLM-5 和 MiniMax2.5 来支撑我们的 Codex 和 Claude Code harness。这让我们可以将自家后端实现与闭源推理对标，目标是在缓存复用性能上达到对等。我们将在接下来几周分享完整的 writeup 和这两个模型的优化部署 recipe。

<table>
<tr>
<td width="50%">

**用 Dynamo 服务 Claude Code**

<video controls width="100%">
  <source src="https://github.com/user-attachments/assets/5fa8a224-44e8-4acb-943b-9b8af84815e6" type="video/mp4" />
</video>

</td>
<td width="50%">

**用 Dynamo 服务 Codex**

<video controls width="100%">
  <source src="https://github.com/user-attachments/assets/8694b544-3fd1-4931-9fd0-7d9a3a2fb78f" type="video/mp4" />
</video>

</td>
</tr>
</table>

我们也在为多种开源模型投入"上线即用（day-0）"的工具调用与推理（reasoning）解析支持。如果你发现某个模型不被支持，请[提交 issue](https://github.com/ai-dynamo/dynamo/issues)，或使用 [tool-call-parser-generator](https://github.com/ai-dynamo/dynamo/blob/main/.agents/skills/tool-parser-generator/SKILL.md) skill，用你选用的 harness 来生成它。

### Agent Hints：harness 与 orchestrator 之间的接口

如今推理服务器看到的，是匿名化的 token 化请求。但 Agent harness 拥有基础设施完全看不到的全局上下文：哪些 Agent 正在等待工具调用、哪些刚刚被启动、当前会话还剩多少轮、当前调用是一次快速查询还是一次长时间的合成。在使用编码 Agent 时，用户等待的是最终结果而非单个 token 流，所以 orchestrator 可以跨 Agent 重排和提升请求优先级而不影响最终用户体验。一次会话可能持续几分钟、[甚至好几天](https://factory.ai/news/missions)，期间穿插长时间的工具调用暂停。这些都足以以传统服务无法实现的方式优化推理调度。

![Where nvext fits in the agentic protocol stack alongside MCP and A2A](./protocol-stack.svg)

Dynamo 新引入的 agent hints 扩展正是为弥合这一差距而设计。它允许任意 harness 在请求上附加结构化提示信息，跨三个 API 端点传递，使 router 与 runtime 能拥有做出 Agent 感知的调度与缓存决策所需的上下文。这是一个 v1 API，我们正与社区一起共同设计，非常希望听到正在构建 Agent harness 的团队反馈：哪些信号最有用？欢迎随时告诉我们！

```json
{
  "model": "MiniMaxAI/MiniMax-M2.5",
  "messages": [...],
  "tools": [...],
  "nvext": {
    "agent_hints": {
      "osl": 256,
      "speculative_prefill": true,
      "priority": 10
    },
    "cache_control": {
      "type": "ephemeral",
      "ttl": "1h"
    }
  }
}
```

`agent_hints` 字段：

- **`priority`** 同时控制 router 和引擎的调度。值越大代表在 Dynamo API 层面"越重要"；Dynamo 会把它翻译为 router 队列顺序与具体后端引擎的优先级。
- **`osl`**（output sequence length）是 harness 对该请求会生成多少 token 的估计。router 用它来评估某个 worker 会被占用多久，从而改善负载均衡。Harness 可以通过追踪每种工具调用的平均输出长度，逐步学到这个值。
- **`speculative_prefill`** 提示 orchestrator 在完整请求就绪之前，先在很可能被选中的 worker 上为该请求的前缀进行缓存。这在 harness 知道某次工具调用即将返回、想提前预热缓存时很有用。

`cache_control` 字段对用过 Anthropic prompt caching API 的人会很熟悉。它告诉 orchestrator：把已经计算好的前缀在 worker 上钉住（pin）指定的 TTL 时长，避免在工具调用间隙被驱逐。目前仅支持 `ephemeral`（与 Anthropic 的 API 对齐）。下文的缓存保留章节会讨论它如何工作。完整的 agent hints 文档见 [此处](../../components/frontend/nvext.md#cache-control)。

## 第 2 层：路由（Router）

编码 Agent 遵循一种顺序模式：长预填充（prefill）、工具调用、扩展前缀、再循环。多 Agent harness 则把工作 fanout 到多个并行 sub-agent，每个 sub-agent 都有较短、独立的上下文。默认的轮询路由对这两种模式都视而不见——它无法考虑缓存局部性、请求优先级或会话结构。Dynamo 的 router 通过三种机制来弥合这一差距：KV 感知放置、优先级调度和可扩展的路由策略。

### KV 感知放置

如果没有缓存感知路由，第二轮对话有大约 1/N 的概率落到第一轮所在的 worker。每一次未命中都意味着完整前缀重算，这是显著的性能瓶颈，对终端用户也极其昂贵。Dynamo 的 router 维护着一份全局索引，记录哪些 KV 缓存块存在于哪些 worker 上。[Flash Indexer 文章](../flash-indexer/flash-indexer.md) 介绍了将这个索引器迭代到 1.7 亿次操作/秒（**行星级** KV 路由）的六次迭代历程。每个请求到来时，router 都会向该索引查询每个 worker 的重叠分数，并选择能最小化"缓存未命中代价 + 当前解码（decode）负载"的 worker。该代价函数可调，下文我们会展示团队如何在它之上构建自定义的 Agent 感知路由策略。

### 优先级调度

`priority` 是面向用户唯一的调度旋钮。值越大表示在 Dynamo API 层面"越重要"。Dynamo 在两个层面都使用这一提示：

- 在 **router** 层，启用 `--router-queue-threshold` 时高优先级请求会在队列中提前。
- 在 **引擎** 层，Dynamo 会归一化各后端的优先级极性，并把请求转发用于队列排序、抢占与 KV 缓存驱逐。

在 router 处，进入的请求按"有效到达时间"进入一个 `BinaryHeap<QueueEntry>`。更高的 `priority` 让请求看起来到达更早，从而排在低优先级请求之前。请求只在所有 worker 的负载都超过可配置阈值时才会进入队列；在阈值之下则完全绕过队列直接选择 worker。当容量释放（预填充完成或某请求结束）时，队列优先排空高优先级条目。

请求一经派发，SGLang、vLLM 和 TRT-LLM 对引擎层优先级的解释可能不同，所以 Dynamo 会按后端归一化最终面向引擎的取值。像 SGLang 这类引擎也可以基于优先级做 radix cache 驱逐——内存压力大时优先驱逐低优先级的块。

![How priority flows from harness through router dispatch to engine treatment](./two-gates.svg)

### Agentic 工作负载路由策略

带有 200K 上下文窗口的研究 Agent，需要找到剩余 KV 容量足以容纳完整状态的 worker。Router 默认的代价函数（重叠分数 + 解码负载）能覆盖常见场景，但拥有领域特定负载的团队可以使用 router 的 Python 绑定实现自定义路由策略。核心 `KvRouter` 类提供 `best_worker()`（查询路由决策）、`get_potential_loads()`（查看每个 worker 的负载）以及 `generate()`（路由 + 派发一步到位）。自定义 router 注册到与默认组件相同的服务网格上，并可按请求覆盖 router 配置：

```python
# Query per-worker load and overlap for custom routing logic
loads = await router.get_potential_loads(token_ids)

# Override routing config based on request properties
# Long contexts benefit from heavier overlap weighting
config = {"overlap_score_weight": 2.0} if len(token_ids) > 8192 else {}
worker_id, dp_rank, overlap = await router.best_worker(
    token_ids,
    request_id="req-123",
    update_indexer=True,
    router_config_override=config
)

# Or bypass the default selector entirely when the harness
# has its own worker selection logic (e.g., session affinity)
stream = await router.generate(
    token_ids, model=model, worker_id=chosen_worker
)
```

[NeMo Agent Toolkit (NAT)](https://github.com/NVIDIA/NeMo-Agent-Toolkit/tree/develop/examples/dynamo_integration) 团队使用这些 API 构建了一个自定义的在线学习 agentic router。他们的 router 从 `nvext` 注解中提取会话元数据，并喂给一个 [Thompson Sampling](https://en.wikipedia.org/wiki/Thompson_sampling) bandit 风格的代价函数，让其学习在负载下"哪类 worker 对哪类前缀模式表现最好"。相比 Dynamo 默认路由，他们测得 p50 TTFT 降低 4 倍，p50 tokens/s 提升 1.5 倍。对延迟敏感请求做优先级标记，在中等内存压力下可达到最多 63% 的 p50 TTFT 降低。实现细节见 [NAT Dynamo 集成示例](https://github.com/NVIDIA/NeMo-Agent-Toolkit/tree/develop/examples/dynamo_integration)。我们很快会把这一功能作为路由策略加入 Dynamo。


## 第 3 层：KV 缓存管理

Agentic 工作负载产生的块在复用价值上差异巨大——系统提示每轮复用、推理 token 几乎不再被复用——但默认的 LRU 驱逐对所有块一视同仁。一次 2–30 秒的工具调用暂停就可能让某个 Agent 整个前缀过期，恢复时必须重算。缓存需要理解块的价值、支持跨 worker 共享、并尊重 Agent 生命周期边界。

### 统一驱逐的弊端

| 块类型 | 复用模式 | 价值 |
|------------|---------------|-------|
| 系统提示 + 工具定义 | 每一轮 | 最高 |
| 对话历史 | 后续轮次中单调增长 | 高 |
| 思考/推理 token | 推理循环关闭后通常零复用（占输出的相当部分） | 接近零 |
| Subagent 的 KV | 几轮后 Agent 终止，无需保留 | 接近零 |

LRU 仅看最近性。在高流量环境中，Agent 等待外部 API 工具调用返回的 2–30 秒，可能就足以让其块过期；当 Agent 恢复时，整个前缀必须重算。要解决这个问题，我们需要为 orchestrator 提供 API，控制哪些块应该被保留、放在哪里、保留多久。

### 把 KV 缓存当作共享资源

如今，KV 缓存被视为每个 worker 上本地、临时的资源。某个 Agent 的约 32K token 系统提示和工具定义，会在每个为其请求服务的 worker 上独立计算。当一个主 Agent 派生 4 个共享工具定义的 sub-agent 时，如果这些 sub-agent 落到不同 worker 上，那部分共享前缀会被重算 4 次。在我们对 Claude Code 团队会话的分析中直接测得：协作成员（teammate）平均 79.4% 的命中率，而主 Agent 的探索 sub-agent 是 91.3%（读/写比 5.0x vs. 11.7x），差距几乎完全由每个 teammate 首次调用的冷启动写入造成。目标是让高价值 KV 缓存块对集群中所有 worker 可见——在冷启动时一次写入，之后任何 worker 在任何时候都能读取。

像 SGLang 的 HiCache 与 Dynamo 的 KV Block Manager（KVBM）这类方案，正在朝着 4 层内存层级（memory hierarchy）演进：

![KV cache memory hierarchy: GPU (HBM) at ~ns, CPU (pinned DRAM) at ~us, Local NVMe at ~ms, Remote Storage (NIXL) at ~ms via RDMA](./kv-memory-hierarchy.svg)

块走 write-through 路径：当 worker 为某前缀计算了 KV 后，块自动从 GPU 流向 CPU、再流向磁盘。每个块在全局注册表中按 sequence hash 去重。一旦注册，该块不可变（immutable），并可被任何能访问对应存储层级的 worker 寻址。

这正面解决了 sub-agent 冷启动问题。当主 Agent 计算工具定义和系统提示时，那些块会 write-through 写入共享存储。当 sub-agent 1 落到不同 worker 时，router 查询 Flash Indexer，发现块已在共享存储中，于是该 worker 通过 NIXL（RDMA 读）加载，而非从头重算。Sub-agent 2 同理。原本 4 次冗余预填充计算变为 1 次计算 + 3 次加载。同样的机制也解决了解耦预填充-解码（disaggregated prefill-decode）服务下的缓存一致性问题。在解耦模式下，prefill worker 计算 KV 后通过 NIXL 传给 decode worker；decode worker 生成 token，产生新的 KV 状态。下一轮，prefill worker 同时需要原始前缀以及第 1 轮生成的 token，但它们只在 decode worker 上。有了共享存储，decode worker 把新块写入公共层级，任意 prefill worker 在下一轮都能拉取。

多层存储解决了共享和持久化问题，但块仍只在请求到达 worker 之后才会出现在 GPU 上。Agentic 系统缺失的一环是预取（prefetch）：harness 可以利用历史时序数据预测 Agent 的工具调用何时返回，因此能预知哪些块会被需要、何时被需要。我们正在构建预取钩子（prefetch hook），让 harness 能发出"在下一次请求之前把这些块从存储拉到 GPU"的信号。结合下面的保留 API，harness 就拥有了完整的生命周期控制：钉住块以阻止驱逐、设置优先级以控制驱逐顺序、在需要前主动预取块。

![During tool calls, KV blocks offload to host memory and storage, then prefetch back to GPU before the next LLM call.](./tool-call-offload-prefetch.svg)

### 选择性缓存保留

让块全局可见解决了共享问题，但没有解决驱逐问题。SGLang 和 vLLM 都通过优先级堆支持基于优先级的驱逐——harness 给每个请求指定一个数值优先级，低优先级的块先被驱逐。TensorRT-LLM 把这一思想又推进一步：`TokenRangeRetentionConfig`（由 Dynamo 团队成员 [@jthomson04](https://github.com/jthomson04) 设计实现），可以在单个请求内做按区间的细粒度控制。

请求携带零或多个指令。没有指令的块走默认 LRU 路径，零额外开销。驱逐器变成两结构系统：未标记块的 LRU 空闲列表（O(1)，与原先一致）和带注解块的优先队列。Harness 可以表达"系统提示块最后被驱逐（priority: 100）；对话上下文要熬过 30 秒的工具调用（duration: 45s）；解码 token 最先被丢弃（priority: 1）"，引擎无需理解为什么。

Anthropic 的 prompt caching 让你可以把前缀标记为可缓存。Dynamo 的 `cache_control` API 把同样的语义带到了自托管推理中。当请求带 `cache_control: { type: "ephemeral", ttl: "1h" }` 时，router 会在 worker 的 radix tree 中钉住对应前缀节点指定时长，避免它们在 worker 的 L2 存储中被驱逐。

下一步是把保留与分布式缓存连起来。今天，保留指令仅作用于单个 worker 的本地缓存。如果某块在 worker A 上被钉住而下一个请求被路由到 worker B，钉住关系不会跟过去。把保留语义扩展到 HiCache/KVBM 的共享存储层，意味着 harness 可以钉一次块，让它跨 worker 存活：优先级与 TTL 元数据随块在 write-through 路径上一起流动，任何从共享存储加载该块的 worker 都继承该保留策略。结合上文的预取钩子，harness 就拥有了贯穿整个内存层级的端到端生命周期控制。

### Agent 生命周期感知

考虑一个典型的 Claude Code 会话：主 Agent 运行 20+ 轮，对话前缀不断累积；途中它派生若干个探索 sub-agent，每个跑 1–3 轮就终止；它可能派出由 4 个专家组成的并行小队，处理不同子任务后终止；中途它撞上上下文上限，对历史做汇总，把约 175K token 压缩到约 40K。这些事件每一次都会产生短期 KV：那些块再也不会被引用。Sub-agent 终止、上下文汇总、关闭的推理循环都会产生短期 KV，却与系统提示等高价值块占用相同的内存。推理模型把这一现象放大：`<think>...</think>` 块约占生成 token 的 40%，但推理循环一关闭就成了短期块。如果缺乏生命周期感知，缓存只能一视同仁。

![Lead agent conversation flow branches to a sub-agent. The sub-agent's ephemeral KV is evicted on session end.](./subagent-lifecycle-vertical.svg)

上文提到的保留原语（优先级、TTL、token 区间）提供了基础构件。缺的是一种把它们与会话关联起来的能力。如果 harness 可以把 sub-agent 的请求标记为属于某个会话，并把该会话的 KV 标记为短期（ephemeral），驱逐器就能优先针对这些块下手，甚至跳过把它们写入共享存储。当 sub-agent 终止时，其会话的块最先回收。同样的机制也适用于思考 token：引擎可以在生成期间检测 `<think>` 边界，并在插入时就把这些块标为短期，让它们跳过 L2 回写并比普通块更早被驱逐，无需任何外部信号。这里的设计空间很广：harness 驱动的会话标签、引擎原生的语义检测、二者结合的混合方案。我们正在多个方向积极探索，最佳答案预计会因工作负载与框架而异。

## 弥合差距

Agentic 推理中最大的优化空间，在于"harness 知道的"和"基础设施能看到的"之间的鸿沟。哪些 Agent 在等待、哪些即将恢复、哪些 KV 值得保留、哪些可以丢弃——这些上下文都存在于 harness 层，但从未跨越 API 边界。`nvext.agent_hints` 是我们朝着弥合这一差距迈出的第一步：一组小而结构化的信号，让 orchestrator 可以做出有依据的路由、调度与缓存管理决策，而不是把每个请求当作匿名 token。这是一个 v1 API，我们仍在持续演进。如果你正在构建 Agent harness、为 Agentic 工作负载运行开源模型、或在思考缓存感知推理，我们很想听听对你的用例最重要的是哪些信号。欢迎在 [GitHub](https://github.com/ai-dynamo/dynamo) 上联系我们，或在 X 上 @ 我们：[@0xishand](https://x.com/0xishand)、[@KranenKyle](https://x.com/KranenKyle)、[@flowpow123](https://x.com/flowpow123)。
