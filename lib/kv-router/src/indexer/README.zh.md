# ⚡ FlashIndexer — KV Router 索引数据结构

> 本文是 [`README.md`](./README.md) 的中文译本，原文档保留不动。

本文档介绍 KV cache 的索引实现：`RadixTree`（及其并发变体 `ConcurrentRadixTree`）和 `PositionalIndexer`（NestedMap）。

并发版索引器的组合吞吐可达 **每秒 1000 万 events + requests 以上**，**p99 延迟低于 10 微秒**。

## 模块地图

| 文件 | 作用 |
|------|------|
| `mod.rs` | 模块声明与 re-export |
| `traits.rs` | `KvIndexerInterface`（async trait）与 `SyncIndexer`（线程池后端使用的同步 trait） |
| `types.rs` | `KvRouterError`、`MatchRequest`、`WorkerTask`、通道消息类型 |
| `metrics.rs` | `KvIndexerMetrics` — Prometheus 计数器和直方图 |
| `kv_indexer.rs` | `KvIndexer` — 基于 tokio mpsc 通道、对 `RadixTree` 的单线程异步封装 |
| `radix_tree.rs` | `RadixTree` — 单线程树，节点为 `Rc<RefCell<RadixBlock>>`，追踪每个 block 的访问频率 |
| `concurrent_radix_tree.rs` | `ConcurrentRadixTree` — 线程安全变体，节点为 `Arc<RwLock<Block>>`，查找使用 `DashMap` |
| `positional.rs` | `PositionalIndexer` — 扁平 `DashMap<(pos, hash), SeqEntry>`，带 jump 优化 |
| `thread_pool.rs` | `ThreadPoolIndexer<T: SyncIndexer>` — N 个 OS 线程承担 sticky-routed 写入、读取走 inline；包裹 `ConcurrentRadixTree` 或 `PositionalIndexer` |
| `local.rs` | `LocalKvIndexer` — 在 `KvIndexer` 之上加一层环形事件缓冲区，供 worker 侧去中心化路由使用 |
| `pruning.rs` | `PruneManager` — 基于 TTL 的过期和基于 `BinaryHeap<BlockEntry>` 大小的剪枝 |
| `naive.rs` | 暴力 baseline 索引器（仅 bench 用，藏在 `bench` feature 后） |
| `tests.rs` | 所有索引器变体的集成测试 |

## 设计动机：四个 Block 标识

分布式 LLM 系统里每个被缓存的 KV block 都需要四类信息：

### 1. Local Block Hash（`LocalBlockHash`，u64）

**是什么**：一个 block 内部 token（例如 64 个 token）的哈希，可选地包含 LoRA adapter 名和多模态元信息。

**为什么**：标识 block 本身的内容，与上下文无关。token（以及 LoRA adapter）相同的两个 block 就有相同的 local hash。当指定 LoRA adapter 时，会把 adapter 名带长度前缀拼接到待哈希字节序列后再哈希，保证不同 adapter（或基座模型）下的 block 总能得到不同 hash。

```text
位置 5 的 Block：tokens [101, 102, 103, ...]
LocalBlockHash = hash(tokens)                                = 0xABCD1234  （基座模型）
LocalBlockHash = hash(tokens || len("my-lora") || "my-lora") = 0xDEAD5678  （LoRA adapter）
```

### 2. External Sequence Block Hash（`ExternalSequenceBlockHash`，u64）

**是什么**：从序列起点累加到当前 block 的累积哈希。

**为什么**：唯一标识一个 block 在**某个具体**序列历史中的位置。两个 block 即使本身内容相同，只要前缀不同，sequence hash 就不同。

```
序列 A: [block0, block1, block2]
序列 B: [block0', block1', block2]  // block2 内容相同但前缀不同

A 中的 block2: seq_hash = hash(hash(hash(block0)  || block1)  || block2) = 0x1111
B 中的 block2: seq_hash = hash(hash(hash(block0') || block1') || block2) = 0x2222
```

**计算公式**：`seq_hash[i] = hash(seq_hash[i-1] || local_hash[i])`，其中 `seq_hash[0] = local_hash[0]`。

> **重要：引擎提供的 hash**
>
> 实际部署中，`ExternalSequenceBlockHash` 可能直接来自推理引擎（如 TensorRT-LLM、vLLM），由我们不掌控的滚动 hash 算法计算得到。引擎在内部计算这些 hash，然后通过 KV cache event 上报。
>
> **LoRA 隔离**：引擎需要在 emit KV event **之前**把 LoRA adapter 标识纳入 `ExternalSequenceBlockHash`。Dynamo 不在 router 层添加 LoRA 信息。例如 vLLM 通过 `_gen_lora_extra_hash_keys` 在调用 `hash_block_tokens(..., extra_keys)` 时把 LoRA ID 作为额外 key 附加进去。任何接入 KV router 的引擎都必须遵守同样的约定，才能保证不同 LoRA adapter 之间的缓存隔离正确。
>
> **对索引实现的影响：**
>
> - **RadixTree**：可以处理引擎提供的 hash，因为它用 `LocalBlockHash` 来导航树结构，`ExternalSequenceBlockHash` 只用作不透明的查找 ID，不需要自己重算 hash。
>
> - **NestedMap**：依赖 `find_matches` 中的"懒哈希"优化能够**增量计算** `ExternalSequenceBlockHash`。要用 NestedMap，必须满足以下任一条件：
>   1. **强制使用已知哈希器**：把引擎配置成 router 能复刻的某种特定哈希算法；或
>   2. **在 relay 上重算**：在 publisher/relay 层用已知算法重算滚动 hash 后再把事件转发给 router。
>
> 否则 NestedMap 的 `find_matches` 在遇到 `SeqEntry::Multi`（同一 position+local_hash 上有多个 seq_hash）时会失败，因为无法消歧到底该用哪个 entry。

### 3. Worker ID（`WorkerWithDpRank`）

**是什么**：标识"哪个 worker（推理服务器）"缓存了这个 block。

**为什么**：router 需要根据各 worker 的缓存内容决定哪些 worker 能服务这个请求。

### 4. Position（`usize`）

**是什么**：block 在序列中的下标（0、1、2、……）。

**为什么**：支持高效的前缀匹配。Position 0 是第一个 block，position N-1 是最后一个。

---

## 核心操作

两种数据结构都支持三类操作：

| 操作 | 说明 |
|------|------|
| `store_blocks` | 为某个 worker 添加 block（后台） |
| `remove_blocks` | 为某个 worker 移除 block（后台） |
| `find_matches` | 找出前缀匹配的 worker（每请求一次） |

**读 vs 写的代价与频率**：当请求在 radix tree 中没有共享前缀（或共享很短）时，`find_matches` 只需要在 root 一次查找就能退出（第一个 block 即 miss）——此时读比写更轻量（写要遍历并更新多个节点）。反之，前缀重合很大的场景下，读会向下走得更深，且读频率可能远高于写。我们同时考虑两种极端，所设计的数据结构能兼顾。

---

## RadixTree：基于树的索引

### 结构

```
RadixTree
├── root: SharedRadixBlock (Rc<RefCell<RadixBlock>>)
└── lookup: HashMap<Worker, HashMap<SeqHash, SharedRadixBlock>>

RadixBlock
├── children: HashMap<LocalBlockHash, SharedRadixBlock>
├── workers: HashSet<Worker>
├── block_hash: Option<SeqHash>
└── recent_uses: VecDeque<Instant>
```

### 可视化示意

```
                    [root]
                   /      \
            local=0xA    local=0xB
               ↓            ↓
           [block0]     [block0']
           workers:     workers:
           {W0,W1}      {W2}
              |
         local=0xC
              ↓
          [block1]
          workers:
          {W0,W1}
              |
         local=0xD
              ↓
          [block2]
          workers:
          {W0}         ← W1 在此分叉
```

### 操作工作流程

**store_blocks(worker, parent_hash, blocks)**：
1. 通过 `lookup[worker][parent_hash]` 找到父节点
2. 对每个 block，用 `local_hash` 找/创建子节点
3. 把 worker 加入每个节点的 `workers` 集合
4. 更新 `lookup[worker][seq_hash] = node`

**remove_blocks(worker, block_hashes)**：
1. 对每个 hash，用 `lookup[worker][hash]` 找到节点
2. 把 worker 从该节点的 `workers` 集合中移除
3. 若 `workers` 为空，则清空子节点（级联清理）
4. 从 `lookup[worker]` 中移除

**find_matches(local_hashes, early_exit)**：
1. 从 root 出发，初始候选 = 全部 worker
2. 对每个位置，沿匹配 `local_hash` 的子节点向下
3. 把候选集合与节点的 `workers` 取交集
4. 记录每个 worker 在哪一深度被淘汰
5. 返回 `{worker -> depth}` 评分

### 复杂度

| 操作 | 时间 | 空间 |
|------|------|------|
| store_blocks (N blocks) | O(N) | O(N) 个节点 |
| remove_blocks (N blocks) | O(N) | - |
| find_matches (深度 D) | O(D × W) | O(W) |

其中 W = worker 数量。

---

## ConcurrentRadixTree：线程安全变体

`ConcurrentRadixTree` 是 `RadixTree` 的并发改造版。核心变化是把节点的 `Rc<RefCell<>>` 换成 `Arc<RwLock<>>`，并用 `DashMap` 做每个 worker 的查找表：

```
ConcurrentRadixTree
├── root: SharedBlock (Arc<RwLock<Block>>)
└── lookup: DashMap<Worker, RwLock<HashMap<SeqHash, SharedBlock>>>
```

`DashMap` 把锁竞争分散到多个 shard 上，每个 worker 的 block map 由自己的 `RwLock` 保护。这意味着 `find_matches` 全程只拿读锁——无论是树节点还是 lookup 表——因此多个读可以并行进行，互不阻塞。

写入（`store_blocks`、`remove_blocks`）会对涉及节点采用 hand-over-hand 锁定方式拿写锁（先父后子）。为避免写-写竞争，`ConcurrentRadixTree` 被设计成包在 `ThreadPoolIndexer` 内使用，后者按 worker 做 sticky 路由：每个 `WorkerId` 通过 `DashMap<WorkerId, usize>` 映射到固定的 OS 线程，事件经过 per-thread 的 `flume` channel 派发。由于某个 worker 的 KV event 永远落在同一线程，该 worker 子树上的写入就被天然串行化，无需跨线程加锁。

```
                   ┌──────────────────────────────────┐
find_matches() ──→ │   Arc<ConcurrentRadixTree>       │ ← 读直接 inline 执行
                   │                                  │
 KV events ──→ flume[0] ──→ thread 0 (W0, W3) ──→     │
           ──→ flume[1] ──→ thread 1 (W1, W4) ──→     │ ← 写通过 sticky
           ──→ flume[2] ──→ thread 2 (W2, W5) ──→     │   worker 分配
                   └──────────────────────────────────┘
```

同样的模式——读走调用线程 inline、写走线程池 sticky 路由——也用于 `PositionalIndexer`（见下）。两者都实现 `SyncIndexer` trait，并被包进 `ThreadPoolIndexer`。

权衡点：`ConcurrentRadixTree` 砍掉了 `RadixTree` 中的 `recent_uses` 频次追踪，从而让 `find_matches` 完全只读（读路径不会写任何可变状态）。

---

## PositionalIndexer（NestedMap）：以位置为先的 HashMap 索引

### 结构

```
PositionalIndexer
├── index: DashMap<(Position, LocalHash), SeqEntry>
├── worker_blocks: DashMap<Worker, RwLock<HashMap<SeqHash, (Position, LocalHash)>>>
└── jump_size: usize

SeqEntry（用枚举做内存优化）
├── Single(SeqHash, HashSet<Worker>)               // 常见：单个 seq_hash
└── Multi(HashMap<SeqHash, HashSet<Worker>>)       // 少见：多个前缀
```

`PositionalIndexer` 实现 `SyncIndexer`，通过 `DashMap`（分片并发 map）和 `RwLock` 实现线程安全。它被设计成包在 `ThreadPoolIndexer` 中：写事件路由到专属 OS 线程，读直接 inline 执行。

`index` 用 `(position, local_hash)` 作为复合键存进 `DashMap`，既能借 shard 分散锁竞争，又能给 jump 优化提供 O(1) 的随机位置访问。`worker_blocks` 反向查找的外层用 `DashMap`，每个 worker 的 block 集合是 `RwLock<HashMap>`——因为 `ThreadPoolIndexer` 的 sticky 路由已经把同一 worker 的写入串行化了。

### 可视化示意

```
index (DashMap，复合键)：
┌──────────────────────┬──────────────────────────────────────┐
│ (pos=0, local=0xA)   │ Single(seq=0x1111, {W0,W1})          │
│ (pos=0, local=0xB)   │ Single(seq=0x2222, {W2})             │
│ (pos=1, local=0xC)   │ Single(seq=0x3333, {W0,W1})          │
│ (pos=2, local=0xD)   │ Multi{                               │
│                      │    seq=0x4444 → {W0},                │
│                      │    seq=0x5555 → {W1}   ← 分叉        │
│                      │  }                                   │
└──────────────────────┴──────────────────────────────────────┘

worker_blocks (DashMap<Worker, RwLock<HashMap>>)：
┌─────────┬─────────────────────────────────────────────────┐
│ W0      │ seq=0x1111 → (pos=0, local=0xA)                 │
│         │ seq=0x3333 → (pos=1, local=0xC)                 │
│         │ seq=0x4444 → (pos=2, local=0xD)                 │
├─────────┼─────────────────────────────────────────────────┤
│ W1      │ seq=0x1111 → (pos=0, local=0xA)                 │
│         │ seq=0x3333 → (pos=1, local=0xC)                 │
│         │ seq=0x5555 → (pos=2, local=0xD)                 │
└─────────┴─────────────────────────────────────────────────┘
```

### 操作工作流程

**store_blocks(worker, parent_hash, blocks)**：
1. 确定起始位置：
   - 如果批次显式给了 `start_position`，就用它
   - 否则 `worker_blocks[worker][parent_hash].position + 1`
   - 再否则 `0`
2. 对位置 `i` 上的每个 block：
   - 插入 `index[(pos+i, local_hash)]` → 把 worker 加入 SeqEntry
   - 插入 `worker_blocks[worker][seq_hash] = (pos+i, local_hash)`

**remove_blocks(worker, block_hashes)**：
1. 对每个 hash，查 `(pos, local_hash) = worker_blocks[worker][hash]`
2. 把 worker 从 `index[(pos, local_hash)]` 中移除
3. 从 `worker_blocks[worker]` 中移除
4. 清理 DashMap 中变空的 SeqEntry

**find_matches(local_hashes, early_exit) 配合 Jump 优化**：
1. 从位置 0 开始，按第一个 block 初始化候选集合
2. **Jump**：按 `jump_size`（例如 32）向前跳
3. 每次跳到的位置上检查候选是否仍然匹配（只数数，不 clone）
4. 若有 worker 掉队，**回扫**精确定位被淘汰的位置
5. 直到序列耗尽或只剩一个 worker 为止

```
查询：[b0, b1, b2, ..., b63, b64, ..., b127, ...]
        ↑                   ↑                  ↑
       pos=0              pos=64             pos=128
        │                   │                  │
        └── jump ──────────→└── jump ─────────→│
                           全部命中？        有人掉队？
                              ↓                   ↓
                           继续            回扫 [64,128]
```

**懒哈希优化**：
- 大多数 (position, local_hash) 上只有一个 seq_hash（SeqEntry::Single）
- 这种情况下完全跳过 seq_hash 的计算
- 仅在需要消歧时（SeqEntry::Multi）才计算

**dump_events()**：
1. 遍历 `worker_blocks`，收集每个 worker 的全部 block
2. 把每个 worker 的 block 按位置排序
3. 给每个 block 单独 emit 一条 `RouterEvent::Stored`，其中
   `start_position = Some(position)`，`parent_hash = None`
4. 这些事件可以重放到新的 `PositionalIndexer` 中重建同样的索引状态

### 复杂度

| 操作 | 时间 | 空间 |
|------|------|------|
| store_blocks (N blocks) | O(N) | O(N) 条 entry |
| remove_blocks (N blocks) | O(N) | - |
| find_matches (深度 D) | O(D/J) | O(W) |

其中 J = jump_size，W = worker 数量。Jump 优化把 D 次顺序查找减少为 D/J 次跳跃，仅在跳跃点发现 worker 掉队时才需要在跳过的区段做一次线性回扫。

---

## 对比

| 维度 | RadixTree | PositionalIndexer |
|------|-----------|-------------------|
| **结构** | 节点 Rc<RefCell<>> 的树 | 复合键的 DashMap |
| **并发版** | ConcurrentRadixTree (Arc<RwLock<>> + DashMap) | 默认线程安全（DashMap + RwLock） |
| **find_matches** | O(D×W) 树遍历 | O(D/J)，带 jump 优化 |
| **store_blocks** | O(N) 建节点 | O(N) DashMap 插入 |
| **remove_blocks** | O(N) 带级联清理 | O(N) 带 entry 清理 |
| **dump_events** | BFS 遍历整棵树 | 每个 worker 内按位置排序 |
| **内存** | 较高（每节点 Rc/Arc 开销） | 较低（扁平 entry） |
| **缓存局部性** | 差（指针跳跃） | 好（位置优先） |

---

## 为什么"位置"对 PositionalIndexer 这么关键

DashMap 用 `(position, local_hash)` 复合键正是为了支持 jump 优化：

```text
// 没有"位置优先"：必须遍历整棵树
for pos in 0..depth {
    node = node.children[local_hashes[pos]];  // O(depth) 次遍历
}

// 有了"位置优先"：可以直接跳到任意位置
let workers_at_64  = index.get(&(64,  local_hashes[64]));   // O(1) 查找
let workers_at_128 = index.get(&(128, local_hashes[128]));  // O(1) 查找
// 位置 1-63、65-127 整段直接跳过！
```

---

## 索引器选型指南

线上有五种索引器变体。本节把部署场景映射到合适的选择。

### 变体汇总

| 变体 | 结构体 | 并发模型 | 路由方式 | 何时使用 |
|------|--------|---------|---------|----------|
| `RadixTree` | `RadixTree` | 单线程 | N/A（本地） | Worker 侧的 `LocalKvIndexer`、单元测试 |
| `ConcurrentRadixTree` (CRT) | `ConcurrentRadixTree` | 线程安全的读 | N/A（本地） | 单节点、中等流量 |
| `ThreadPoolIndexer<CRT>` (CRTC) | `ThreadPoolIndexer<ConcurrentRadixTree>` | N 个写线程 + inline 读 | 全连接、覆盖所有 worker | 大多数部署的默认选择 |
| `BranchShardedIndexer<CRTC>` (BSI) | `BranchShardedIndexer<ThreadPoolIndexer<CRT>>` | 分片写池 | FNV 前缀哈希 | 高 worker 数、需要近似剪枝 |
| `AnchorAwareBranchShardedIndexer<CRTC>` | `AnchorAwareBranchShardedIndexer<ThreadPoolIndexer<CRT>>` | 分片写池 | 前缀 TRIE 路由 | 高 worker 数、正确性优先（不做近似剪枝） |

### 各变体的使用场景

**`RadixTree`（非并发）**
- 永远不要用在 router/scheduler 侧。
- 在 `LocalKvIndexer` 中用是合适的——worker 侧、单 tokio task 访问、无需加锁。
- 在写单元测试时也很有用，因为它简单、确定性强。

**`ConcurrentRadixTree`（裸用，不放进 ThreadPoolIndexer）**
- 仅当你只有一个专门的写入者、大量并发读者，且更看重代码简洁而非极致吞吐时使用。
- 实践中 `ThreadPoolIndexer<CRT>` 全面更优：既保证写并发安全，又能拿到更高吞吐。

**`ThreadPoolIndexer<ConcurrentRadixTree>` (CRTC)**
- 默认选择，从这里入手。
- 单一全局索引覆盖所有 worker。读路径 `find_matches` 始终为 O(深度 × worker 数)。
- 在 ~1000 worker 以内表现良好。再往上，由于每次查询都要扫所有 worker，`find_matches` 的 p99 会上涨。
- 不分片：在极端规模下，单一热点分支可能成为写瓶颈。

**`BranchShardedIndexer<CRTC>` (BSI)**
- 通过固定深度的 FNV 前缀哈希把每个会话路由到一个 shard。读路径只扫被路由到的那个 shard 上的前缀键。
- 支持**近似剪枝**：每个 shard 可以独立淘汰自己最久未使用的 entry，无需与其他 shard 协调。
- 适用场景：worker 数 > ~1000，需要 per-shard 近似剪枝，且能容忍 flat-map router 在浅链和乱序方面的限制。
- **跨 shard 行为**：`branch_sharded.rs` 为浅根存储 `block_to_fnv_state`，并为短查询注册前缀别名，但 flat map 并不能提供更强的 anchor/replay 模型，无法保证任何浅延续都和它的父 shard 待在一起。当延续在父被知晓之前到达时，乱序问题仍是注意点。
- 如果你需要"任何会话即使在乱序事件下也保证留在同一 shard"的保证，请用 anchor-aware BSI。

**`AnchorAwareBranchShardedIndexer<CRTC>`（anchor-aware BSI）**
- 用一个路由 TRIE 代替 flat hash map。在派发延续之前，预先在目标 shard 上安装 *anchor* 节点，使 CRTC 在查询时已经有父链。
- 架构上**防止**了 shard 跨越：路由决策在事件派发之前作出，而不是之后。
- 取舍：**不**支持近似剪枝（anchor 节点是共享状态，剪掉某个 shard 会破坏 TRIE）。
- 适用场景：路由正确性优先，且可以放弃近似剪枝。

### 快速决策树

```text
开始
  │
  ├─ 单 worker 或 worker 侧本地索引？
  │      └─ RadixTree
  │
  ├─ < ~1000 worker，不需要分片？
  │      └─ CRTC (ThreadPoolIndexer<ConcurrentRadixTree>)
  │
  └─ ≥ ~1000 worker（或希望按 shard 剪枝）？
         │
         ├─ 需要近似剪枝 / 完整 BSI 能力？
         │      └─ BranchShardedIndexer<CRTC>  （flat-map 路由）
         │
         └─ 需要路由正确性保证、无需剪枝？
                └─ AnchorAwareBranchShardedIndexer<CRTC>

