# 设计文档：`block_manager`（v1）vs `kvbm-logical` —— 评估与迁移路径

## 背景

`lib/llm/src/block_manager/`（下文统称 **v1**）管理 LLM 推理（inference）的 KV 缓存（KV cache）block 生命周期。其逻辑层 —— block 状态机、registry、pool、events —— 与物理层关注点（存储后端（backend）、CUDA、NIXL、卸载）交织在一起。

`lib/kvbm-logical/` 是一个独立的 crate，提供独立的逻辑 block 生命周期层。本文档从 API 表面、registry 设计、pool 架构、可测性与可用性等多个维度，对两种实现进行客观对比 —— 并评估 `kvbm-logical` 在“按 sequence hash 操作 block”这一未来方向上的契合度。

---

## 1. Registry

两种实现差异最大的部分就是 registry。

### v1 Registry（`block_manager/block/registry.rs`）

- **数据结构**：每个 pool 一份 `HashMap<SequenceHash, Weak<BlockHandle>>`，加上一份在所有 pool 间共享的 `GlobalRegistry = Arc<Mutex<HashMap<SequenceHash, Weak<RegistrationHandle>>>>`。
- **注册句柄**：`RegistrationHandle` 是固定结构体，存储完整的 `TokenBlock` 克隆、`block_hash`、`sequence_hash`、`parent_sequence_hash` 以及一个 `Arc<dyn EventReleaseManager>`。每次注册都会克隆 token 数据。
- **查找**：HashMap 平均 O(1)。无前缀感知或谱系感知操作。
- **清理**：后台 tokio 任务监听 `mpsc::UnboundedChannel<SequenceHash>` 中的 `BlockHandle` drop 事件，再通过 `Weak::upgrade()` 检查并从 per-pool 与全局 map 中移除失效条目。
- **可扩展性**：要给已注册 block 附加新数据，必须修改 `RegistrationHandle` 结构体。字段在编译期固定。
- **频率追踪**：registry 层无任何追踪。访问频率不被记录。

### kvbm-logical Registry（`kvbm-logical/src/registry/`）

- **数据结构**：`PositionalRadixTree<Weak<BlockRegistrationHandleInner>>`。查找为 O(log n)，且具备前缀感知 —— 树结构与 sequence hash 谱系一致。
- **注册句柄**：`BlockRegistryHandle` 是轻量的。它不克隆 token，而是使用一个支持类型化数据关联的 `AttachmentStore`：
  - `attach_unique<T>(value)` —— 每种类型一个值
  - `attach<T>(value)` —— 每种类型多个值
  - `get<T>() -> TypedAttachments<T>` —— 提供 `with_unique()`、`with_multiple()`、`with_all()` 及其可变变体
  - 不必修改句柄结构体即可扩展。
- **查找**：`register_sequence_hash()`、`match_sequence_hash()`、`is_registered()`、`check_presence<T>()`、`check_presence_any()`。
- **清理**：基于 drop —— 由 `BlockRegistrationHandleInner::Drop` 完成。无需后台任务。
- **频率追踪**：可选的 `TinyLFU` 追踪器（Count-Min Sketch，4 个 hash 函数，4-bit 计数器，可配置 decay）。方法：`touch(seq_hash)`、`count(seq_hash)`、`frequency_tracker()`。已集成进 `register_sequence_hash()` —— 每次注册都会增加频率。
- **Touch 回调**：`on_touch(callback)` 注册在 hash 被访问时触发的回调。`touch()` 触发所有已注册回调。
- **存在性追踪**：每种 block 类型可显式 `mark_present<T>()` / `mark_absent<T>()`。`has_block<T>()` 与 `has_any_block(type_ids)` 用于查询某类型化 block 当前是否仍存活于句柄上。

### 对比说明

两个 registry 都按 `SequenceHash` 追踪 block 并使用弱引用进行生命周期管理。v1 的 HashMap 平均 O(1)，但每次查找会在 pool 内已注册的所有 block 上 hash 与探测整个键空间。kvbm-logical 的 `PositionalRadixTree` 先按位置偏移（token 流中的序列位置）缩小范围，让候选集在搜索前被压缩多个数量级。对于跨大量序列位置注册大量 block 的工作负载，这种分区方式很重要。

kvbm-logical 中的 attachment 系统是另一处关键架构差异 —— 它允许在不修改句柄结构体的情况下，把任意类型化数据关联到一个已注册 hash。

---

## 2. Block 守卫类型：MutableBlock、ImmutableBlock、WeakBlock

这些是两个代码库中使用最频繁的类型 —— 每次 block 分配、注册、匹配与归还都要经过它们。

### v1 MutableBlock（`block.rs:572-688`）

```rust
pub struct MutableBlock<S: Storage, L: LocalityProvider, M: BlockMetadata> {
    block: Option<Block<S, L, M>>,
    return_tx: tokio::sync::mpsc::UnboundedSender<Block<S, L, M>>,
    parent: Option<Arc<MutableBlock<S, L, M>>>,
}
```

- 每次使用都带 **3 个泛型参数**：`Storage`、`LocalityProvider`、`BlockMetadata`。
- **Option 包装**：内层 block 是 `Option<Block>`。通过 `Deref`/`DerefMut` 访问，会调用 `.expect("block was dropped")`（`block.rs:680,686`）。如果 block 已被 take（通过 `try_take_block`）后再走 Deref 访问，会 panic。
- **RAII 归还**：drop 时通过 `mpsc::UnboundedSender<Block>` 把 block 送给 pool 的后台任务（`block.rs:646-653`）。
- **父链**：`parent: Option<Arc<MutableBlock>>` 跟踪 block 谱系。Drop 实现使用显式迭代展开，避免深层父链导致栈溢出（`block.rs:657-673`）。
- **状态转移**：`MutableBlock` deref 到 `Block`，后者暴露 `init_sequence()`、`add_token()`、`commit()`、`apply_token_block()`、`reset()` —— 都返回 `Result<()>`。调用方需自行确保 block 在正确状态。

### kvbm-logical MutableBlock（`blocks/mutable.rs:14-99`）

```rust
pub struct MutableBlock<T: BlockMetadata> {
    block: Option<Block<T, Reset>>,
    return_fn: ResetReturnFn<T>,
}
```

- **1 个泛型参数**：`T: BlockMetadata` —— 但 `BlockMetadata` 是无方法的 blanket trait：
  ```rust
  pub trait BlockMetadata: Clone + Send + Sync + 'static {}
  impl<T: Clone + Send + Sync + 'static> BlockMetadata for T {}
  ```
  在实践中，`T` 是一个 **零大小标记类型**（存储层级标记），如 `struct G1;` 或 `struct G2;`。它不携带数据也不携带行为 —— 仅在类型层面区分 block 类型。`MutableBlock<G1>` 与 `MutableBlock<G2>` 在编译期不能混用。这与 v1 的 `BlockMetadata`（一个有 4 个必需方法 `on_acquired`、`on_returned`、`reset_metadata`、`offload_priority` 的 trait）有显著不同。
- **Option 包装**：同样的 `Option<Block>` 模式。访问时 `.expect("MutableBlock missing block")`（`mutable.rs:75,80`）。同样可能 panic，但 take-then-access 的窗口更窄 —— `take_block()` 仅在 `complete()` 与 `stage()` 中被调用，而它们消耗 `self`。
- **RAII 归还**：drop 时调用 `(self.return_fn)(block)` —— 是闭包，而不是 channel send（`mutable.rs:84-90`）。无 async 开销，无 channel 分配，无后台任务 recv。
- **无父链**。无需迭代式 drop 展开。
- **状态转移**：`MutableBlock` 有两种转移：`complete(token_block) -> CompleteBlock<T>` 与 `stage(seq_hash) -> CompleteBlock<T>`。两者都**消耗 self** 并返回新类型。你不能 `complete()` 两次，也不能在完成之后再调用 `MutableBlock` 的任何方法 —— 这是编译器层面的强制。
- **错误恢复**：`complete()` 返回 `Result<CompleteBlock<T>, BlockError<MutableBlock<T>>>`。失败（大小不匹配）时，`MutableBlock` 会被放回错误中返回。v1 的 `apply_token_block()` 返回 `Result<()>` —— block 仍维持原状态，但调用方拿不到类型化保证。

### v1 ImmutableBlock（`block.rs:772-957`）

```rust
pub struct ImmutableBlock<S: Storage, L: LocalityProvider, M: BlockMetadata> {
    block: Arc<MutableBlock<S, L, M>>,
    sequence_hash: SequenceHash,
    duplicate: Option<Arc<MutableBlock<S, L, M>>>,
}
```

- **包装 Arc\<MutableBlock\>**：所谓 immutable 引用其实是 `Arc<MutableBlock>`。不可变只是约定，并非类型层面强制 —— 内部仍是 `MutableBlock`，所有可变方法都能通过 `Deref` 访问。
- **Deref 链**：`ImmutableBlock -> (deref) -> Block -> (methods)`。要触达 block 数据需要两层间接。
- **duplicate 字段**：`duplicate: Option<Arc<MutableBlock>>` —— 作为可选字段加入。当允许重复且第二个 block 以同 hash 注册时，重复 MutableBlock 会挂在这里。`block_id()` 方法判定：如果 duplicate 存在，返回 duplicate 的 ID；否则返回主 block 的 ID（`block.rs:843-847`）。这意味着 `ImmutableBlock.block_id()` 在是不是重复时返回值不同。
- **Clone**：克隆 Arc + sequence_hash + 可选的 duplicate Arc。多个 ImmutableBlock 可指向同一底层数据。
- **不能降级**：无法创建 weak 引用。持有 ImmutableBlock 即持有强引用，会阻止 block 被驱逐。

### kvbm-logical ImmutableBlock（`blocks/immutable.rs:13-67`）

```rust
pub struct ImmutableBlock<T: BlockMetadata> {
    block: Arc<dyn RegisteredBlock<T>>,
    upgrade_fn: UpgradeFn<T>,
}
```

- **包装 Arc\<dyn RegisteredBlock\<T\>\>**：内部是 trait object，而非具体的 `MutableBlock`。`RegisteredBlock<T>` trait 仅暴露 `block_id()`、`sequence_hash()`、`registration_handle()` —— 没有可变方法，没有状态转移。不可变性是结构上强制的。
- **1 个类型参数**：仅 `T: BlockMetadata` —— 一个零大小存储层级标记（见上文 MutableBlock 一节）。
- **`downgrade() -> WeakBlock<T>`**（`immutable.rs:33-39`）：创建 weak 引用。Weak 引用不阻止驱逐。这对希望追踪 block 但不固定（pin）block 的缓存至关重要。
- **`use_count() -> usize`**（`immutable.rs:55-57`）：暴露 `Arc::strong_count()` 用于诊断。
- **`registration_handle()`**（`immutable.rs:51-53`）：直接访问 registry 句柄，并经此进入 attachment 系统。

### kvbm-logical WeakBlock（`blocks/immutable.rs:19-97`）

```rust
pub struct WeakBlock<T: BlockMetadata> {
    sequence_hash: SequenceHash,
    block: Weak<dyn RegisteredBlock<T>>,
    upgrade_fn: UpgradeFn<T>,
}
```

- **v1 没有等价物**。在 v1 中，对已注册 block 的引用都是强 Arc。
- **两阶段升级**（`immutable.rs:71-83`）：先尝试 `Weak::upgrade()`（快速路径 —— block 仍存活）。失败则调用 `upgrade_fn(sequence_hash)`，在活跃 + 非活跃 pool 中搜索（慢路径 —— block 可能已迁移 pool）。
- **Clone**：可克隆。多个 WeakBlock 可追踪同一 sequence hash 而不阻止驱逐。
- **用途**：调度器（scheduler）、缓存与状态追踪器在不阻止驱逐的前提下引用 block。

### 对比说明

v1 的 `ImmutableBlock` 包装 `Arc<MutableBlock>` 意味着“不可变”的 block 仍通过 Deref 暴露所有可变 API。命名具有误导性 —— 更准确的描述是“对可变 block 的共享访问”。kvbm-logical 的 `Arc<dyn RegisteredBlock<T>>` 在结构上把 API 限制为只读操作。

v1 没有 WeakBlock。每个引用都是强引用。这意味着持有 block 引用的调度器会阻止该 block 被驱逐，即使调度器只想知道 block 是否存在。kvbm-logical 的 WeakBlock 直接解决了这个问题。

归还路径不同：v1 经 async channel（`mpsc::UnboundedSender`）发送，而 kvbm-logical 调用同步闭包。对于频繁发生的 MutableBlock drop，v1 每次归还都付出 channel send 开销 + 后台任务 recv + 处理。kvbm-logical 的闭包内联执行。

---

## 3. Block 重复策略

两种实现都处理“新 block 以与已注册 block 相同的 sequence hash 注册”这一情形，但采用了显著不同的方法。

### v1 重复（`pool/managed/state.rs:200-257`、`block.rs:820-831`）

**配置**：`BlockRegistrationDuplicationSetting` —— `Allowed` 或 `Disabled` —— 在 pool 构造时设定。

**注册流程**（简化）：
1. block 进入注册。
2. 检查非活跃 pool：`inactive.match_sequence_hash(seq_hash)`。命中则把已有 block 视作“primary”，新 block 视作“duplicate”。
3. 若不在非活跃中，尝试 `block.register(&mut self.registry)`。若返回 `BlockAlreadyRegistered`，则调用 `wait_for_returned_block()` —— 一个 **async wait**，会阻塞注册路径直到已存在 block 转换回来。
4. 包成 `ImmutableBlock::new(mutable_block)`。
5. 按策略：
   - **Allowed**：`immutable.with_duplicate(duplicate_arc)` —— 把 duplicate 作为 ImmutableBlock 上的 `Option<Arc<MutableBlock>>` 字段。
   - **Disabled**：`duplicate.try_take_block()` —— 抽取裸 block，归还到非活跃 pool。

**duplicate 存储**：duplicate 存在 `ImmutableBlock` 的 `duplicate: Option<Arc<MutableBlock>>` 中。这意味着：
- duplicate 存在时 `block_id()` 返回 duplicate 的 ID，否则返回 primary 的 ID。
- `try_take_block()` 必须 unwrap 两个 Arc。
- primary 的生命周期与 duplicate 的 `ImmutableBlock` 绑定 —— 二者都存活直到 `ImmutableBlock` 被 drop。
- 存在 `is_duplicate()` 检查，但是 `pub(crate)`。

**竞态处理**：`wait_for_returned_block()` 是 async 函数，从 block 归还 channel 持续 recv 直至目标 hash 出现。这可能阻塞整个注册路径。

### kvbm-logical 重复（`registry/registration.rs:119-166`、`blocks/registered.rs:26-154`）

**配置**：`BlockDuplicationPolicy` —— `Allow` 或 `Reject` —— 在 BlockManager 上设置。

**注册流程**（简化）：
1. block 通过 `register_block_inner()` 进入注册。
2. 在 registry 句柄上获取 attachment 锁。
3. 调用 `try_find_existing_block()` —— 检查：
   - **存在性标记**：`attachments.presence_markers.contains_key(&TypeId::of::<T>())`。无标记则立刻返回 None（该类型不存在已存在 block）。
   - **Weak 引用**：`weak_block.primary_block.upgrade()` —— 尝试找到活跃 primary。
   - **非活跃 pool**：`inactive_pool.find_block_as_primary(hash, false)` —— 使用 `new_unattached()` 避免死锁（调用方持有 attachment 锁）。
   - **自旋重试**：若存在性标记存在但在两 pool 中均未找到（block 在 pool 间迁移中），最多自旋 100 次。
4. 找到已存在则按策略：
   - **Allow**：`DuplicateBlock::new(registered_block, existing_primary, reset_return_fn)` —— 创建专门的 `DuplicateBlock<T>` 类型。primary 作为 `_primary: Arc<PrimaryBlock<T>>` 持有。
   - **Reject**：`extracted.discard(&reset_return_fn)` —— 新 block 立即归还到 reset pool。返回已有 primary。
5. 若不存在已有，则通过 `PrimaryBlock::new_attached()` 注册为新的 `PrimaryBlock<T>`。

**类型分离**：`PrimaryBlock<T>` 与 `DuplicateBlock<T>` 是分开的 `pub(crate)` 类型。两者都实现 `RegisteredBlock<T>`，但 drop 行为不同：
- `PrimaryBlock::drop()` 通过 `registered_return_fn` 把 `Arc<Block<T, Registered>>` 归还到非活跃 pool。
- `DuplicateBlock::drop()` 调用 `block.reset()` 并通过 `reset_return_fn` 把 `Block<T, Reset>` 归还到 reset pool。重复 block 被回收，而非保留。

**Weak 引用存储**：`PrimaryBlock::store_weak_refs()` 在句柄的 attachment 系统中同时存储 `Weak<Block<T, Registered>>` 与 `Weak<PrimaryBlock<T>>`（`registered.rs:76-92`）。这样能复活：当 block 正被归还（PrimaryBlock 已 drop、Arc 正被搬到 pool）时，raw_block 的 weak 引用仍能抓到它。

**锁纪律**：`new_attached()` 与 `new_unattached()`（`registered.rs:42-68`）防止调用方已持锁时再次加锁。6 个创建点中 5 个使用 `new_attached`；只有 `find_block_as_primary` 使用 `new_unattached`。

### 哪一版逻辑更紧凑

kvbm-logical 的重复逻辑在多处更紧凑：

1. **类型化角色**：`PrimaryBlock` 与 `DuplicateBlock` 是不同类型，drop 行为不同。v1 用一个带 `Option<Arc<MutableBlock>>` 字段的 `ImmutableBlock`，靠检查 Option 是否为 Some 来回答“这是 duplicate 吗？”，类型系统并不区分两种角色。
2. **drop 路径分离**：在 kvbm-logical 中，duplicate block 在 drop 时 reset 并归还 reset pool；primary block 以 registered 状态归还非活跃 pool。这两条是不同类型中的不同代码路径。在 v1 中，`ImmutableBlock::try_take_block()` 必须通过检查 `self.duplicate` 是否存在来同时处理两种情况，并 unwrap 两个 Arc，再把结果收到 Vec 中（`block.rs:936-957`）。
3. **没有 async 等待**：v1 的 `wait_for_returned_block()` 阻塞注册路径，等待特定 hash 出现在 async channel 上。kvbm-logical 的 `try_find_existing_block()` 使用存在性标记 + 弱引用升级 + 自旋（最多 100 次）。自旋有界，且不涉及 channel 通信。
4. **存在性标记**：kvbm-logical 通过对 registry 句柄的显式 `mark_present<T>()` / `mark_absent<T>()` 调用来追踪某类型是否存在。这让 `try_find_existing_block` 在不存在该类型 block 时可立刻短路。v1 必须先查 registry 的 HashMap，再查非活跃 pool 的 HashMap，最后回退到 async 等待。
5. **Weak 引用复活**：kvbm-logical 在 attachment 存储中存了 `WeakBlockEntry { raw_block, primary_block }`。这能处理 PrimaryBlock::drop 把 Arc 从 Option 中 take 出来与 return_fn 完成 pool 插入之间的竞态窗口。v1 没有这个 —— 它依赖 async 归还 channel 与 `wait_for_returned_block()`。
6. **锁感知**：kvbm-logical 提供 `new_attached()` / `new_unattached()` 显式管理锁顺序。v1 没有这一层关注，因为它的 pool 操作走 async channel，而非直接获取锁。

权衡：v1 的 `wait_for_returned_block()` 保证能找到（在 channel 上无限期等待）。kvbm-logical 的自旋上限为 100 次，可能放弃 —— 即 `MAX_RETRIES` 限制下，转换非常缓慢的 block 可能漏掉，但在 194+ 测试套件中尚未观察到。

---

## 4. 非活跃 Pool 后端

### v1 非活跃 Pool（`pool/managed/inactive.rs`）

- **数据结构**：用于查找的 `HashMap<SequenceHash, Block<S, L, M>>`，加上用于驱逐排序的 `BTreeSet<PriorityKey<M>>`，以及用于未初始化 block 的 `VecDeque<Block>`。
- **驱逐排序**：`PriorityKey<M>` 委托给 `M: Ord`。对于 `BasicMetadata`，派生的 `Ord` 先比 `priority`（u32），再比 `returned_tick`（u64），最后比 `acquired_tick`（u64）。优先级数值更小者先驱逐。同优先级下 `returned` 时间更早者先驱逐（类 LRU）。
- **分配**：先从 `uninitialized_set` 弹（FIFO），再从 `priority_set.pop_first()`（最低优先级 key）弹。
- **hash 匹配**：`HashMap::remove()` —— 平均 O(1)。
- **单一策略**：驱逐策略完全由 `BlockMetadata` 实现的 `Ord` 决定。不更换 metadata 类型就无法切换策略。

### kvbm-logical 非活跃 Pool（`pools/inactive/`）

- **后端 trait**：`InactivePoolBackend<T: BlockMetadata>`，方法包括 `find_matches()`、`scan_matches()`、`allocate()`、`insert()`、`len()`、`has_block()`、`allocate_all()`。
- **内置后端**：
  1. **HashMap** —— 类似 v1。可插拔的 `ReusePolicy`（FIFO、LRU 等）决定驱逐哪个 block。
  2. **LRU** —— 简单的 LRU 驱逐。
  3. **MultiLRU** —— 4 个频次层级（Cold、Warm、Hot、Very Hot），可配置阈值（默认 [3, 8, 15]）。block 根据 TinyLFU 追踪器中的频次计数在层级间晋升。驱逐优先从最冷层级开始。
  4. **Lineage** —— 自定义的基于谱系的驱逐，考虑 sequence hash 父子关系。
- **后端选择**：构建期通过 `BlockManager::builder()` 配置：`.with_lru_backend()`、`.with_multi_lru_backend()`、`.with_multi_lru_backend_custom_thresholds(t1, t2, t3)`、`.with_hashmap_backend(reuse_policy)`、`.with_lineage_backend()`。

### 对比说明

v1 只有一种与 `BasicMetadata` 的 `Ord` 实现绑定的驱逐策略。这能用，但很僵硬 —— 改驱逐行为就要改 metadata 类型或其顺序。kvbm-logical 通过 `InactivePoolBackend` trait 把驱逐策略与 metadata 解耦。MultiLRU 后端尤其能借助 TinyLFU 频次追踪器做出按层的驱逐决策，这在 v1 中没有等价物。

v1 的 `BTreeSet<PriorityKey>` 方法在卸载场景（G2->G3）下确实能很好地处理优先级感知 —— 高 `priority` 值先被卸载（卸载队列里的 `OffloadRequestKey` 反转了比较）。该优先级机制存在于非活跃 pool 之外 —— 在卸载路径里。

---

## 5. Block Pool 架构

### v1 Pool（`pool.rs`、`pool/managed.rs`）

- **async 优先**：所有 pool 操作（`allocate_blocks`、`register_blocks`、`match_sequence_hashes`、`touch_blocks`、`try_return_block`）都是 `BlockPool` trait 上的 `async` 方法。每个还有 `_blocking` 变体。
- **通信**：使用 `priority_tx: PriorityChannelSender` 与 `ctrl_tx: mpsc::Sender`，配合 `oneshot` 响应通道。后台 tokio 任务从这些通道处理请求。
- **trait 表面**：`BlockPool<S, L, M>` 需要实现 `BlockPoolController` + `AsyncBlockPoolController` —— 共 3 个 trait，约 14 个 async 方法 + 14 个阻塞变体 = 每种 pool 类型约 28 个方法签名。
- **block 归还**：通过 `UnboundedSender<Block>` 归还。`MutableBlock::drop()` 把内部 block 经此 channel 发送。

### kvbm-logical Pool（`pools/`、`manager/`）

- **同步核心**：pool 操作使用 `parking_lot` 锁。无需 async runtime。无 channel、无后台任务、无 oneshot receiver。
- **pool 分离**：`ResetPool<T>`（可用 block）、`ActivePool<T>`（通过 registry 查找已注册 block）、`InactivePool<T>`（可驱逐的已注册 block）。每个 pool 都是直接方法调用的简单结构体。
- **编排器**：`BlockManager<T>` 组合三个 pool + `BlockRegistry`。公共方法：`allocate_blocks()`、`register_blocks()`、`match_blocks()`、`scan_matches()`、`reset_inactive_pool()`、`total_blocks()`、`available_blocks()`、`block_size()`。
- **block 归还**：`MutableBlock::drop()` 调用 `return_fn: Arc<dyn Fn(Block<T, Reset>)>` 闭包。`InactivePool` 在归还时通过 `Arc::try_unwrap()` 回收 block。
- **reset pool 分配器**：可插拔 `BlockAllocator<T>` trait。默认 `DequeBlockAllocator`（FIFO）。

### 对比说明

v1 即使是内存内操作也把每次 pool 操作包成 async 请求/响应循环。这一设计可能是为了串行化访问并支持跨任务通信，但意味着每次 `allocate_blocks()` 都涉及：sender.send() -> 后台任务 recv() -> 处理 -> oneshot.send() -> 调用方 await。对于本质上是内存查找与 HashMap 修改的操作，这引入了每次操作的额外开销。

kvbm-logical 用 `parking_lot` mutex 的同步方式更简单。代价是调用方在拿锁时阻塞，而非让出给 async runtime。对于典型工作负载（短临界区内做 HashMap 与 BTreeSet 操作），这是合适的 —— 持锁时间在微秒级。

---

## 6. Block 状态机

### v1（`block/state.rs`、`block.rs`）

- **运行时枚举**：`BlockState { Reset, Partial(PartialState), Complete(CompleteState), Registered(Arc<RegistrationHandle>, Arc<BlockHandle>) }`
- **转移**：`Block` 上返回 `Result<()>` 的修改方法。无效转移在运行时产生 `BlockStateInvalid` 错误。
- **状态**：Reset -> Partial（通过 `init_sequence`）-> Complete（通过 `commit`）-> Registered（通过 pool 注册）。也可 Reset -> Complete（通过 `apply_token_block`）。
- **block 类型**：`Block<S: Storage, L: LocalityProvider, M: BlockMetadata>` —— 3 个泛型参数。
- **Partial 状态**：支持增量构建 token：`add_token()`、`add_tokens()`、`pop_token()`、`pop_tokens()`、`commit()`。

### kvbm-logical（`blocks/`）

- **编译期类型态**：`Block<T: BlockMetadata, State>`，其中 `T` 是零大小存储层级标记（`struct G1;`、`struct G2;` 等），`State` 是 `Reset`、`Staged`、`Registered` 之一（同样是零大小标记）。`BlockMetadata` 是无方法的 blanket trait —— 任何 `Clone + Send + Sync + 'static` 类型都满足。`T` 参数在编译期阻止跨存储层级混用 block。
- **转移**：消耗式方法返回新类型。`MutableBlock.complete(token_block) -> CompleteBlock<T>`。`CompleteBlock.register() -> ImmutableBlock<T>`。无效转移在编译期报错。
- **状态**：Reset -> Staged（通过 `complete()`）-> Registered（通过注册）。无 Partial 状态 —— token 构建发生在 block 之外。
- **block 类型**：`Block<T, State>` —— 2 个泛型参数。State 通常被推断。
- **错误处理**：`BlockError<B>` 在失败时把 block 带回，避免资源泄漏。v1 的错误不会带回 block。

### 对比说明

v1 有 `Partial` 状态，可直接在 block 上增量构建 token。kvbm-logical 没有 —— 它要求传入已完成的 `TokenBlock`。也就是说在 kvbm-logical 中，逐 token 构建必须发生在外部。

kvbm-logical 的类型态模式在编译期阻止整类运行时错误。v1 的运行时枚举更灵活（可动态查询状态），但每个调用方都得处理无效状态转移。

---

## 7. 可测性

| 方面 | v1 | kvbm-logical |
|--------|-----|-------------|
| 运行时要求 | 因 async pool 操作，多数测试需要 `#[tokio::test]` | 同步。无需 async runtime |
| block 创建 | 需要 layout 分配、`BlockData::new(Arc<layout>, idx, set_idx, worker_id)`、`Block::new(data, metadata)` | `TestBlockBuilder::new(id).with_block_size(8).fill_iota(100).build_staged()` |
| 提供的测试工具 | `create_reference_block_manager_config()` —— 需要物理设置 | `testing` feature：`create_test_manager()`、`TestBlockBuilder`、`BlockSequenceBuilder`、`create_staged_block()`、`create_reset_blocks()` |
| 测试 metadata | `BasicMetadata`（生产类型） | `TestMeta(u64)`、`MetadataA`/`B`/`C`（专为测试设计） |
| 测试数量 | 约 15 个活跃测试，多个被注释掉 | 194+ 个测试 |
| 下游测试支持 | 无 test feature | `#[cfg(any(test, feature = "testing"))]` 把工具暴露给消费者 |

---

## 8. 未来策略：按 Sequence Hash 操作 Block

未来的关键方向是 **按 sequence hash 跨所有 pool 操作 block** —— 附加 metadata、固定 block、保留指定时长，并在 touch 等事件时更新驱逐顺序。

### 两种实现对该方向的支持

**v1**：`GlobalRegistry` 把 `SequenceHash -> Weak<RegistrationHandle>` 映射，`RegistrationHandle` 是固定结构体。要按 hash 固定 block 或附加 metadata，需要：
1. 给 `RegistrationHandle` 加字段（如 `pinned: AtomicBool`、`custom_metadata: HashMap<TypeId, Box<dyn Any>>`）
2. 修改 pool 驱逐以检查 pin 状态
3. 从零开始添加 touch/频率追踪
4. 把这些改动穿过 async channel 边界

**kvbm-logical**：`BlockRegistry` 把 `SequenceHash -> BlockRegistryHandle` 映射，`BlockRegistryHandle` 已带 attachment 系统。要按 hash 固定 block 或附加 metadata：
1. `handle.attach_unique::<PinState>(PinState::Pinned)` —— 不需要改结构体
2. `handle.on_touch(callback)` —— 注册驱逐顺序更新逻辑
3. 通过 TinyLFU 已集成频率追踪
4. 驱逐时通过 `handle.get::<PinState>().with_unique(|pin| ...)` 查询 pin 状态
5. pool 在做驱逐决策时可读取 attachment，并通过 `handle.has_block::<T>()` 检查存在性

kvbm-logical 中的 attachment 系统 + touch 回调 + 存在性追踪正是为这种工作流设计的。在 v1 中，每多一种“按 hash 操作”的能力都需要对 `RegistrationHandle` 做结构性修改并穿过 async 边界。

### kvbm-logical 启用的具体未来模式

- **按 hash 固定**：`registry.match_sequence_hash(hash)?.attach_unique::<Pinned>(Pinned(duration))`
- **按 hash 附加 metadata**：`handle.attach::<CustomMetadata>(data)` —— 同类型多次附加
- **touch 时跨 pool 更新驱逐**：`handle.on_touch(|h| update_eviction_order(h))` —— 任何 pool 触碰该 hash 时触发
- **频率感知驱逐**：MultiLRU 后端从 TinyLFU 读取频次，自动跨层晋升/降级 block
- **存在性查询**：`handle.has_block::<G1Block>()` / `handle.has_any_block(&[g1_id, g2_id])` —— 检查哪些存储层持有该 block

---

## 9. 总结

| 维度 | v1（`block_manager`） | `kvbm-logical` |
|-----------|---------------------|----------------|
| Registry 数据结构 | HashMap（搜索整个键空间） | PositionalRadixTree（先按位置缩小再桶内搜索） |
| Registry 可扩展性 | 固定结构体字段 | 类型化 attachment 系统 |
| 频率追踪 | 无（卸载路径基于时间戳的优先级） | TinyLFU Count-Min Sketch，与 registry 与 MultiLRU 集成 |
| 非活跃 pool 后端 | 单一（基于 metadata Ord 的 BTreeSet） | 4 个可插拔后端（HashMap、LRU、MultiLRU、Lineage） |
| Pool 通信 | async channel + 后台任务 | parking_lot 锁的同步方式 |
| Block 状态强制 | 运行时枚举 | 编译期类型态 |
| 类型参数 | 3 个（`Storage`、`LocalityProvider`、`BlockMetadata`） | 2 个（`T` 存储层级标记、`State`） |
| BlockMetadata 语义 | 带 4 个方法的 trait（`on_acquired`、`on_returned`、`reset_metadata`、`offload_priority`） | 无方法的 blanket trait —— `T` 是 `struct G1;` 这样的零大小标记 |
| 错误恢复 | 出错丢失 block | `BlockError<B>` 把 block 带回 |
| 测试数量 | 约 15 | 194+ |
| 测试运行时 | async（tokio） | 同步 |
| block 上 token 构建 | 是（Partial 状态） | 否（外部） |
| Touch 回调 | 无 | 有 |
| WeakBlock 支持 | 无 | 有 |
| MutableBlock 类型参数 | 3（`S`、`L`、`M`） | 1（`T`） |
| MutableBlock RAII 归还 | async channel 发送（`mpsc::UnboundedSender`） | 同步闭包调用 |
| MutableBlock 父链 | 有（迭代式 drop 防栈溢出） | 无 |
| ImmutableBlock 内层类型 | `Arc<MutableBlock>`（仍能 Deref 出可变 API） | `Arc<dyn RegisteredBlock<T>>`（只读 trait object） |
| WeakBlock | 不支持 | `WeakBlock<T>`，两阶段升级（先弱引用再 pool 搜索） |
| Block 重复类型 | ImmutableBlock 上的可选字段 | 专用 PrimaryBlock / DuplicateBlock 类型 |
| 重复 drop 行为 | 与 primary 合一（try_take_block 同时处理两者） | 分离：DuplicateBlock 重置并归还 reset pool；PrimaryBlock 归还非活跃 pool |
| 重复检测 | async `wait_for_returned_block()`（在 channel 上无限等待） | 存在性标记 + 弱引用升级 + 有界自旋（最多 100 次） |
| 注册时错误恢复 | 保留 block 但无类型化保证 | `BlockError<B>` 在错误类型中带回 block |

### kvbm-logical 替换了什么

- `block/state.rs` -> `kvbm-logical::blocks::state`
- `block/registry.rs` -> `kvbm-logical::registry/`
- `block.rs`（Block、MutableBlock、ImmutableBlock）-> `kvbm-logical::blocks/`
- `pool.rs` + `pool/managed.rs`（逻辑 pool 操作）-> `kvbm-logical::pools/` + `kvbm-logical::manager/`
- `events.rs` -> `kvbm-logical::events/`

---

## 10. 验证

1. `cd lib/kvbm-logical && cargo test` —— 194+ 个测试通过
2. 对比 registry：`kvbm-logical/src/registry/` vs `block_manager/block/registry.rs`
3. 对比非活跃 pool 后端：`kvbm-logical/src/pools/inactive/backends/` vs `block_manager/pool/managed/inactive.rs`
4. 对比 pool 通信：`kvbm-logical/src/manager/mod.rs` vs `block_manager/pool/managed.rs`
5. 对比 block 守卫类型：`kvbm-logical/src/blocks/` vs `block_manager/block.rs`
