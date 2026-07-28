---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: "Flash Indexer: A Story of Inter-Galactic KV Routing"
subtitle: "Rudy Pei, [John Thomson](https://developer.nvidia.com/blog/author/jwillthomson/), Janelle Cai, Alec Flowers, Ryan Olson, Dan Gil — February 2026"
description: "Dynamo's Flash Indexer tracks every cached KV block across all inference workers at 170M ops/s. Six iterations of data structure design got it there."
keywords: KV cache, prefix caching, LLM inference, radix tree, distributed inference, disaggregated serving, Dynamo, concurrent data structures, block routing
last-updated: February 23, 2026
---

**Flash Indexer** 是一个并发的全局索引，它跟踪所有推理（inference） worker 上每一个已缓存的 KV 块（block），可持续支撑 **每秒一亿次以上的操作**。它经历了六次迭代演化 —— 从一个 Python 字典开始，到一个面向跳跃搜索（jump search）优化的位置索引 —— 直到网络延迟、tokenize 与哈希计算才成为瓶颈。我们正把它作为 Dynamo v1.0.0 的默认 indexer 发布。

数量级直觉：在 100M+ 次/秒索引操作量下，系统大约可以支撑 $$N \approx 10^8 / r$$ 个并发负载，其中 $$r$$ 是该负载在真实流量下持续的索引操作率（插入 + 查询，包括 prefill 突发），远超当前任何"行星级"推理需求。

本文按这些迭代逐步展开 —— 解释每一次重构如何带来一个数量级的提升，以及背后具体的数据结构或并发突破。

---

## 1. 背景

### 1.1 KV 块身份

每个被缓存的 block 携带三个标识：

- **本地块哈希（local block hash，`u64`）**：单个 block 内 token 的内容哈希。位置无关 —— 两个 token 内容相同的 block 产生相同的哈希。frontend 与发布器使用相同的算法。

- **序列块哈希（sequence block hash，`u64`）**：从开头到该 block 的整个前缀的滚动哈希。位置相关 —— 相同 token 出现在不同位置时哈希不同。

```text
seq_hash[0] = local_hash[0]
seq_hash[i] = hash(seq_hash[i-1] || local_hash[i])
```

- **Worker ID**：哪一个 worker 拥有这个 block。

local hash 被刻意设计为*分块哈希*（不带前缀上下文），这样 frontend 可以并行廉价地哈希查询块。代价：分块哈希无法区分位置。*"Predict the next token | Learn from the error | Predict the next token."* 在 block 0 与 block 2 上得到相同的哈希。这一冲突问题驱动了下文每一处数据结构选择。

### 1.2 事件与请求

indexer 处理两种流量：

**KV 事件**（写）：每个引擎旁配有一个发布器，当一个 block 被缓存时发出 `Store(worker_id, local_hash, seq_hash)`，被驱逐时发出 `Remove(worker_id, seq_hash)`。我们必须依赖显式事件，因为引擎会把 block 缓存超出请求生命周期，而它们的驱逐策略（LRU 扫描、内存压力、抢占）对外不透明 —— 仅凭请求-响应循环无法推断缓存状态。事件流是突发的：prefill 一次产生几十个 store；驱逐扫描一次产生大量 remove。

<Frame caption="Figure 1 — KV Event Density">
  <img src="./images/fig-1-kv-event-density.svg" alt="KV Event Density" />
</Frame>

<Info icon="hand-point-up" className="fig-caption">
Per-worker and aggregate KV cache event density heatmap derived from 5% of the Mooncake FAST'25 trace, replayed across 16 Mocker workers with 2,048 GPU blocks each. Green cells indicate Store-dominant time bins (prefill bursts); amber cells indicate Remove-dominant bins (eviction sweeps). The diverging colorscale is clamped at ±10 events per worker and ±100 events aggregate, highlighting the bursty, temporally correlated nature of KV cache traffic that the Flash Indexer must sustain at line rate.
</Info>

**请求**（读）：每次推理请求时，frontend 发送一串分块哈希 `[local_hash_0, ..., local_hash_D]`。indexer 返回一组 `(worker_id, match_depth)` 评分，让 router 能选出"已缓存前缀最深"的 worker。

<Frame caption="Figure 2 — KV Event Flow">
  <img src="./images/fig-2-kv-event-flow.svg" alt="KV Event Flow" style={{width: "100%"}} />
</Frame>

<Info icon="hand-point-up" className="fig-caption">
Each engine is paired with a publisher that enriches raw KV events with worker identity and block hashes, then broadcasts them via pub/sub. The router requests store lookups from the indexer, which computes prefix overlap scores used for routing decisions.
</Info>

两条路径都是热点。事件慢意味着路由决策陈旧；查询慢意味着用户可见的延迟。设计目标：让两条路径都快，且彼此不相互争用。

---

## 2. 嵌套字典 → Rust Actor

### 2.1 Python 字典

最简单的索引就是嵌套字典。对每个 worker，存放从 local block hash 到对应外部 sequence hash 集合的映射。由于 local hash 是分块哈希，相同 token 在不同序列里可能出现在不同位置，因此同一个 worker 上一个 local hash 可以对应多个 sequence hash。要找匹配，就遍历每个 worker，沿着查询序列逐位比对。

```py
class KvIndex:
    # worker_id -> { local_hash -> set of seq_hashes }
    index: dict[int, dict[int, set[int]]] = {}

    def store(self, worker_id: int, blocks: list[tuple[int, int]]):
        if worker_id not in self.index:
            self.index[worker_id] = {}
        for local_hash, seq_hash in blocks:
            if local_hash not in self.index[worker_id]:
                self.index[worker_id][local_hash] = set()
            self.index[worker_id][local_hash].add(seq_hash)

    def remove(self, worker_id: int, seq_hashes: list[int]):
        if worker_id not in self.index:
            return
        for seq_hash in seq_hashes:
            for local_hash, hashes in self.index[worker_id].items():
                hashes.discard(seq_hash)

    def find_matches(self, query: list[int]) -> dict[int, int]:
        scores = {}
        for worker_id, blocks in self.index.items():
            depth = 0
            for local_hash in query:
                if local_hash in blocks and blocks[local_hash]:
                    depth += 1
                else:
                    break
            if depth > 0:
                scores[worker_id] = depth
        return scores
```

这种做法存在正确性问题。`local_hash in blocks` 只能告诉我们该 worker 拥有*某个* token 内容相同的 block，但不知道是*哪一个* —— 共享同一分块哈希的不同序列被混为一谈。这一冲突问题塑造了后续每一处数据结构选择。每次查询的复杂度是 `O(W × D)`（W 个 worker，查询深度 D）。

在数百个 worker、序列长达数千 block 的场景下，这条路根本走不通。

### 2.2 Rust Actor

把它移植到 Rust（`HashMap<WorkerId, HashMap<LocalHash, HashSet<ExternalHash>>>`）后，解释器开销被消除。一个**单线程 actor** 独占该索引，并通过 channel 通信 —— 正确且无锁，但所有读都必须排在所有写后面。这条单线程就成了吞吐天花板。

---

## 3. 倒排索引

`worker -> { hash -> ... }` 强制 `find_matches` 必须遍历每个 worker。但真正要回答的问题是 *"哪些 worker 拥有这个 block？"* —— 应当以 block 为键，而不是 worker。我们改为构造以 LocalHash 为键、映射到 sequence hash 与其 worker 集合的正向索引。

```rust
// local_hash -> (seq_hash -> set of workers)
index: HashMap<LocalHash, HashMap<ExternalHash, HashSet<WorkerId>>>
```

现在 `find_matches` 只需在查询上做一次遍历。每个位置取若干 worker 集合的并集。worker 只会随着深入而*被淘汰* —— 每个最多被淘汰一次 —— 因此复杂度从 O(W × D) 降到 **O(D + W)**。

倒排索引在读侧是巨大胜利，但每一处数据结构选择都是查询性能与更新成本的双向权衡。

读侧来看，第 2.1 节的冲突问题以另一种形式出现。当我们对某个 local hash 下不同 sequence hash 的 worker 集合做并集时，会把缓存了不同序列、却共享同一分块哈希的 worker 混在一起。索引里其实有 seq hash 数据，但 `find_matches` 若想用上它，就要先计算查询自己的 seq hash —— 这又把滚动哈希计算带回了读路径，正是分块哈希设计要避免的。

写侧来看，remove 同样昂贵：没有按 worker 的反向查找，按 seq hash 删除一个 block 需要扫描整张索引。我们可以再加一张反向查找表，但每次 store 的记账成本就会增加。

radix tree 同时解决了两端。

---

## 4. 基数树（Radix Tree）

每个节点有一张以 `LocalHash` 为键的小型 children map，再加上一份 worker 集合。父子关系把冲突风险限定在一个范围内：两个分块哈希相同的 block 只有在共享同一个父节点（也就是同一前缀）时才可能冲突。前缀不同则父节点不同。这要求 KV 事件中新增一个字段：**parent hash**，以便事件到达时可以把子节点链接到父节点。

<Frame caption="Figure 3 — Prefix Tree Structure">
  <img src="./images/fig-3-prefix-tree.svg" alt="Prefix Tree Structure" style={{maxWidth: "420px", width: "100%"}} />
</Frame>

<Info icon="hand-point-up" className="fig-caption">
Prefix-aware radix tree indexes cached blocks by local hash. Shared prefixes branch where sequences diverge; each node records which workers hold that block.
</Info>

```rust
type SharedRadixBlock = Rc<RefCell<RadixBlock>>;

struct RadixBlock {
    children: HashMap<LocalHash, SharedRadixBlock>,
    workers: HashSet<WorkerWithDpRank>,
    block_hash: Option<ExternalHash>,
}

struct RadixTree {
    root: SharedRadixBlock,
    lookup: HashMap<WorkerWithDpRank, HashMap<ExternalHash, SharedRadixBlock>>,
}
```

每个节点同时携带一个 sequence hash。一张按 worker 维度的**查找表**（`worker -> { seq_hash -> node }`）为事件处理提供 O(1) 访问：store 通过父节点的 seq hash 把孩子挂上去；remove 直接定位到节点。两种 key 服务两种访问模式 —— local hash 用于遍历，sequence hash 用于事件。

树与查找表都通过 `Rc<RefCell<T>>` 指向同一批节点（共享所有权 + 内部可变性，单线程）。每个节点的 children map 都很小 —— 受分支因子限制，与 block 总数无关。

这一方案仍藏在 actor 之后单线程运行，读写串行。

---

## 5. 并发基数树

读之间彼此不冲突。我们把 `Rc<RefCell<T>>` 替换为 `Arc<RwLock<T>>`（原子引用计数 + 读写锁）。如此一来，`find_matches` 只需获取读锁，并*直接在调用方线程内执行* —— 没有 channel、没有 actor、没有队列。

写则采用**粘性路由（sticky routing）**：`ThreadPoolIndexer` 把每个 `WorkerId` 确定地分配到某一条线程。同一 worker 的事件总是落在同一线程，因此该 worker 的子树上不会有写-写竞争。

<Frame caption="Figure 4 — Concurrency Model">
  <img src="./images/fig-4-concurrency-model.svg" alt="Concurrency Model" style={{width: "100%"}} />
</Frame>

<Info icon="hand-point-up" className="fig-caption">
Write events are sticky-routed by worker ID to a thread pool, ensuring sequential ordering. A concurrent radix tree with `Arc<RwLock>` allows `find_matches()` reads in parallel, enabling concurrent traversals.
</Info>

```rust
type SharedBlock = Arc<RwLock<Block>>;

struct ConcurrentRadixTree {
    root: SharedBlock,
    lookup: DashMap<WorkerWithDpRank, RwLock<HashMap<ExternalHash, SharedBlock>>>,
}
```

`DashMap` 把外层 map 分片，对不同 worker 的读写不会触碰同一把锁。`parking_lot::RwLock` 在无竞争路径上避免了 OS 系统调用（约比 `std::sync::RwLock` 快 2–3x）。`FxHashMap` 用 multiply-xor 一步替代 SipHash —— 在这里安全，因为 key 是 `u64` 哈希、不是用户输入。

`parking_lot::RwLock` 默认是 task-fair 的：按到达顺序处理等待者，而不会无条件偏向读者或写者。再叠加粘性路由对每个 worker 写线程化的保证，写竞争极小，读写都不会饥饿。

读路径上 actor 已经消失。多个 `find_matches` 与对不同 worker 的写可以并行进行。

---

## 6. 带跳跃搜索的位置索引器

基数树是逐节点遍历的，沿着指针从父走到子 —— 缓存不友好，且本质上是顺序的。你无法不访问 0..127 就直接看位置 128。

把树替换成一个按位置索引的 `Vec<DashMap<LocalHash, SeqEntry>>`。`index[position]` 是一张并发 map，从 local hash 映射到 sequence entry。任意位置都是 O(1) 访问 —— 无需遍历。

```rust
enum SeqEntry {
    Single(ExternalHash, HashSet<WorkerWithDpRank>),
    Multi(HashMap<ExternalHash, HashSet<WorkerWithDpRank>>),
}

struct PositionalIndexer {
    // index[position] -> { local_hash -> SeqEntry }
    index: Vec<DashMap<LocalHash, SeqEntry>>,
    worker_blocks: DashMap<WorkerWithDpRank, LevelIndex>,
    jump_size: usize,
}
```

`SeqEntry` 这一枚举用于处理冲突：常见情况下一个 `(position, local_hash)` 槽位只对应一个 sequence hash，可以内联存放、无需分配 `HashMap`。只有当多个前缀在同一位置产生相同分块哈希时才升级为 `Multi`。

`Single`/`Multi` 的拆分还使懒哈希计算成为可能：当查找命中 `Single` 时，无需计算查询自己的 sequence hash 即可消除歧义。昂贵的滚动哈希只在罕见的 `Multi` 条目里、当分块哈希冲突需要消歧时才被用到。

但位置索引器最大的优势不是数据布局 —— 而是**随机访问带来的可能性**。

随机访问解锁了**跳跃搜索（jump search）**：

1. 用位置 0 处的活跃 worker 集合初始化。
2. 向前跳 `jump_size` 个位置（如 64）到下一个 checkpoint。
3. 在 checkpoint 上，统计仍然匹配的活跃 worker 数（基数比对 —— 不需要克隆集合）。
4. 如果全部匹配：被跳过的整段都得到确认。继续跳。
5. 如果匹配数变少：在跳过段中有 worker 退出。回扫位置 `[previous_checkpoint + 1 .. current_checkpoint]`，找到每个掉队 worker 的精确退出点。
6. 从当前 checkpoint 继续跳。

<Frame caption="Figure 5 — Positional Jump Search">
  <img src="./images/fig-5-jump-search.svg" alt="Positional Jump Search" style={{width: "100%"}} />
</Frame>

<Info icon="hand-point-up" className="fig-caption">
With position as a first-class key, the indexer jumps ahead by a fixed stride. On a partial match, a lookback from the previous checkpoint identifies exact drain points, then resumes jumping from the current checkpoint.
</Info>

最佳情况：`D / J` 次查找而非 `D` 次。

最坏情况（每次跳跃都掉 worker）：退化为线性扫描并附加跳跃开销。

位置索引器在长序列且前缀共享高的负载上获胜；基数树在短序列或高度发散的序列上获胜。

`Vec` 布局也提升了缓存局部性：靠前的位置（共享的 system prompt、常见前导）正是热点路径，集中在数组前部，长期保持在 cache 中。

跳跃步长 *J*（即 `jump_size`，默认 64）下，摊销代价降为 **O(D/J + W)**。由于 *J* 是可调常量，复杂度依旧关于 *D* 线性；实际收益是当前缀共享高时跳过绝大多数位置。

---

## 7. 基准测试

所有基准在一台 24 核 Arrow Lake（285K）桌面机上跑，把公开的 [Mooncake 生产 trace](https://github.com/kvcache-ai/Mooncake/tree/main/FAST25-release/arxiv-trace) 通过一个开启了前缀缓存（prefix caching）、配置 16,384 个 GPU block 的 mock 引擎回放。测试框架使用 24 个并发的事件处理线程，分别测试全部 5 种后端。

**操作吞吐**是 KV 事件与 `find_matches` 请求每秒合计的速率。我们通过把同一 trace 压缩到更短时长来扫描 offered load，并对比 achieved 与 offered。**threshold throughput** 是 achieved 不再跟得上 offered 的拐点 —— 即 indexer 的饱和点。

<Frame caption="Figure 6 — Indexer Performance">
  <img src="./images/fig-6-indexer-throughput.svg" alt="Indexer Performance" />
</Frame>

<Info icon="hand-point-up" className="fig-caption">
Achieved vs. offered block throughput across five indexer backends, measured with `mooncake_bench` on real trace data. The Flash Indexer sustains 170M ops/s — 42x faster than the Radix Tree shipped in Dynamo v0.1.0 (4M ops/s) and 440x faster than the naive implementations (385K ops/s).
</Info>

---

## 8. 后续方向

Flash Indexer 已随 Dynamo v1.0.0 发布，下一轮优化将攻克剩下的常数因子：

- **跳跃内的二分查找。** 把跳跃失败后的线性回扫替换成二分查找：每次失败跳跃从 `O(J)` 降为 `O(log J)`。
- **分层路由。** 顶层是稀疏的、跨部署组的粗粒度前缀覆盖索引，叶子上挂完整 indexer。
- **内联 bitset 表示 worker 集合。** 用每节点内联的定宽 bitset 替代 `HashSet`，把成员检测变成单个位运算并消除指针追逐。

---

## 9. 结语

从 Python 字典到 Flash Indexer，共经历六次迭代，每一次都是被前一版具体瓶颈推动的：

1. **Naive 嵌套 Dict** —— 简单但每次查询 O(W × D)。
2. **Rust + Actor 模式** —— 语言更快、并发正确，但单线程瓶颈。
3. **倒排索引** —— 翻转 key 结构后每次查询 O(D + W)；用 `seq_hash` 第二层应对分块哈希冲突。
4. **Radix Tree** —— 用树替代巨型扁平 map；每节点 children map 保持小；双 key 设计（local hash 用于遍历，seq hash 用于事件处理）；`Rc<RefCell<>>` 提供单线程共享所有权。
5. **并发 Radix Tree** —— 用 `Arc<parking_lot::RwLock<>>` 替换 `Rc<RefCell<>>`；查找表使用 `DashMap` + 每 worker 内部 `RwLock`（罕见变更走分片级锁，热路径上的共享读廉价）；读绕过 actor；粘性路由让每个 worker 的写在单线程上序列化、零竞争。
6. **基于跳跃搜索的并发位置索引器（Flash Indexer）** —— 长序列负载下的 radix tree 替代品；按位置索引的 `Vec<DashMap<>>` 用 O(1) 随机访问替代指针追逐，使跳跃搜索得以跳过深度上的大部分位置；反向查找用 `DashMap` + 每 worker 内部 `RwLock`；热门前缀位置聚在 `Vec` 前部并长期驻留 cache。

最终结果：持续支撑 **每秒 1.7 亿次操作**（事件与请求合计），achieved throughput 一路紧跟 offered，直到极限。
