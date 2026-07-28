<!-- SPDX-FileCopyrightText: Copyright (c) 2024-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# Concurrent Radix Tree Compressed

`ConcurrentRadixTreeCompressed` 是用于 KV 缓存（KV cache）路由的压缩 trie。它在逻辑形态上与 radix tree（基数树）保持一致，但每个非根节点持有一条压缩边：一组由 `(LocalBlockHash, ExternalSequenceBlockHash)` 对构成的向量。这降低了节点数量，并使 decode（解码）能够在父哈希仍然是当前 tail 的情况下，直接向已有叶子节点追加。

该树支持分裂但不支持合并。清理（cleanup）可以移除过期叶子，但分裂之后的边不会再被重新压缩。

## 动机

原有的 KV-router 索引器在不同权衡之间做了不同选择：

- `RadixTree` 简单且精确，但每个 trie 节点只存一个 block。共享的长 prompt 与 decode tail 会产生大量节点，且每次追加都必须以 block 粒度遍历或分配。
- `RadixTreeIndex` 通过将 worker 分片改善了并发事件吸收，但每个分片仍是一棵普通的 radix tree。跨分片边界的共享前缀只能被复制，而非以共享结构表达。
- `BranchShardedIndexer` 将发散分支路由到不同分片，但单个分片之内的结构仍要廉价地处理高 fanout、decode 扩展以及过期父节点查找。

`ConcurrentRadixTreeCompressed` 正是为这种工作模式量身打造的单分片结构。它的主要不同点：

- **Radix 压缩**：每个非根节点保存一条压缩边而不是单个 block。一条 prefill 链可以由一个节点表示，且在节点尚无子节点时，decode 可以直接向叶子追加。
- **每个 worker 的截断点（cutoff）**：worker 的覆盖范围以"截断点"的方式记录在压缩边内。移除可以缩短某个 worker 的覆盖范围，而无需在每次驱逐时都对物理树进行分裂。
- **粘性内部节点（sticky internal node）**：一旦节点曾经拥有过子节点，即便清理后将这些子节点全部移除，它在逻辑上仍保持为内部节点。这避免了在竞态或清理之后，老的 fanout 点重新被打开为 decode 扩展点。
- **延迟查找修复（lazy lookup repair）**：worker 本地的反向查找仅在观察到陈旧条目时才修复。跨线程的分裂不需要同步性地为每个其他线程修补查找表。
- **半无锁结构性读**：子节点表使用 `DashMap`，边状态使用单独的保护机制。读热路径无需获取 shape gate，而对形态敏感的写入则借助版本验证在计划过期时进行重试。
- **带版本的 shape gate**：节点的 `shape_gate` 与 `shape_version` 将一个小的临界区与显式的过期计划检测结合起来。共享操作（例如在稳定的内部节点下添加子节点）可以通过共享 gate 推进；结构性变更（例如分裂与叶子扩展）则走独占路径。

预期的结果并不是一棵完全无锁的树，而是一棵压缩树：常见的 prefill fanout 与 decode 扩展可以避免不必要的独占串行化，而少见的结构性竞态则通过重试或延迟修复查找状态来解决。

## 节点状态

每个节点包含：

- `edge`：本地与外部 block 哈希组成的压缩序列。
- `edge_index`：从 `ExternalSequenceBlockHash` 到边内位置的反向查找表。一旦确定了节点，移除可借此 O(1) 找到被驱逐的 block。
- `full_edge_workers`：覆盖完整压缩边的 worker。
- `worker_cutoffs`：仅覆盖边某个前缀的 worker。截断点 `k` 表示该 worker 已缓存 `edge[0..k]`，其中 `0 < k < edge.len()`。
- `children`：以子节点边的首个 `LocalBlockHash` 作为键的子节点表。
- `shape_gate` 与 `shape_version`：每节点的形态守卫，用于跨锁间隔地校验计划。
- `internal`：粘性标记，节点曾拥有过子节点后即变为 true。即便清理之后所有物理子节点都被移除，它仍保持为 true，从而该节点不会再被开放给叶子扩展。

worker 查找表并不保存在节点上。每个事件 worker 自己拥有一份 `WorkerLookup`，将外部 block 哈希映射到应当包含该哈希的节点。

## 存储路径

该索引器针对两种常见的 KV 缓存存储模式做了优化。

### Prefill Fanout

许多 worker 可能共享一段前缀，然后在同一个父节点下保存不同的 prompt 后缀：

```text
shared prefix parent
many different workers
many different first child hashes
insert new compressed child nodes
```

期望的行为是：当父节点的边稳定时，独立的子节点插入可以并发推进。子节点插入使用共享 shape gate 加形状版本校验，因此它无需独占地拥有边形态。

### Decode 扩展

decode 期间，一个 worker 通常会在自己压缩边的尾部追加小批量数据：

```text
one worker
one leaf compressed node
parent hash is current tail
node has never been internal
append more blocks to the edge
```

该路径会尝试直接进行叶子扩展。仅当节点不是 `internal`、父哈希仍是边的 tail，且该 worker 覆盖了该 tail 时才允许。如果节点曾经有过子节点，decode 会回退到子节点插入或分裂，而不是扩展压缩边。

### 后缀复用与分裂

当一次 store 的父哈希指向已有压缩边的内部位置时，节点可以复用一段已有的匹配后缀。如果新的 store 与该后缀发生分歧，则会在父位置处对节点进行分裂：后缀变成一个子节点并继承原有的子节点，因此分裂之后已有的后代仍然可达。

## 移除（Removal）

移除会更新 worker 覆盖范围，但不会在结构上分裂边。

当 worker `w` 在边位置 `i` 收到一个 remove 事件时：

- 如果 `w` 在 `full_edge_workers` 中，则 `current_cutoff` 为 `edge.len()`；否则为 `worker_cutoffs[w]`。
- 若 `i >= current_cutoff`，则该 remove 是 no-op，因为该 block 已超出该 worker 的覆盖。
- 若 `i < current_cutoff`，新的截断点为 `i`。
- 如果新截断点为 `0`，该 worker 会从节点中被移除。
- 否则该 worker 被移到 `worker_cutoffs[w] = new_cutoff`。
- 新失去覆盖的后缀对应的 worker 查找条目会被立刻清除。

在覆盖更新之后，仅当不再有 full-edge worker 时，移除才可能清空子节点。由于 `internal` 是粘性的，清空子节点不会让该节点重新具备未来叶子扩展的资格。

## 查找修复

跨线程分裂可能让某个 worker 查找条目变得陈旧：查找仍指向旧的前缀节点，而所请求的外部哈希实际已经迁移到新的后缀子节点。`resolve_lookup` 以延迟方式处理：

```text
worker lookup says hash -> old node
old node no longer contains hash
scan descendants for a node containing hash
rewrite the useful covered range in the resolved node
```

store 路径的修复会从所请求的父哈希向 worker 覆盖的 tail 方向重写。remove 路径的修复则从头向被移除的哈希方向重写，并排除即将被 remove 清除的后缀。这对于 tail-to-head 的移除尤为重要：一旦一个被搬迁压缩边内的某个哈希修复了有用的头部范围，同一条边内后续的哈希应直接命中已解析的节点，而不必再付出一次子树扫描代价。

如果 store 路径中扫描失败，则该 store 会以 `ParentBlockNotFound` 被拒绝并记录为 warning。如果 remove 路径扫描失败，remove 会将该 block 视为已经消失或陈旧并跳过。

## 并发模型

`ThreadPoolIndexer` 按 worker id 进行 sticky 路由，因此同一个 worker 的 KV 事件会在同一个事件线程上串行化。不同 worker 仍可以并发地修改共享的 CRTC 节点。

节点内部对边状态与子节点表使用了独立的保护：

- `NodeState` 由 `parking_lot::RwLock` 保护。
- `children` 是 `DashMap`。
- `shape_gate` 与 `shape_version` 协调依赖于"边与子节点表关系"的计划。

轻量的形状更新（如在稳定父节点下挂上一个缺失的子节点）使用共享 shape gate 加版本校验。重量级的边形态更新（如分裂、扩展叶子、把子节点搬迁到后缀节点）使用独占的 shape gate。

`find_matches` 在并发形状变化期间是 best-effort 的：在热点步骤上读取节点状态与子节点指针时不会取 `shape_gate`，因此在分裂中可能观察到相邻的树形态并低估匹配数。但它绝不能 panic，也不能返回越过有效可达前缀的匹配。

## 与 `RadixTree` 相比的限制

- 不支持 `expiration_duration` 或频率跟踪。
- 不提供 `new_with_frequency()`。
- `find_matches` 不会填充 `OverlapScores.frequencies`。
