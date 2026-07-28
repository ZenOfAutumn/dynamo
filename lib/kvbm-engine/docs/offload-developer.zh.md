# Offload 模块开发者指南

本文档面向参与 offload 流水线（pipeline）实现的开发者。关于高层概念与策略说明，请参见 [offload.md](offload.md)。

## 基于容器（Container）的架构

### OffloadContainer

container 是流水线中流动的基本单元：

```rust,ignore
struct OffloadContainer<T: BlockMetadata> {
    /// Source blocks to transfer
    blocks: Vec<SourceBlock<T>>,
    /// Precondition event - Some before PreconditionAwaiter, None after
    precondition: Option<EventHandle>,
    /// Cancellation token (cloned from TransferHandle)
    cancel_token: CancellationToken,
}

impl<T: BlockMetadata> OffloadContainer<T> {
    /// Check if this container has been cancelled
    fn is_cancelled(&self) -> bool {
        self.cancel_token.is_requested()
    }

    /// Upgrade all blocks from Weak → Strong
    /// Returns None if any block was evicted
    fn upgrade(self) -> Option<UpgradedContainer<T>> {
        // Implementation upgrades each SourceBlock
    }
}
```

### OffloadBatch

batch 把多个 container 组合在一起以提升传输效率：

```rust,ignore
struct OffloadBatch<T: BlockMetadata> {
    containers: Vec<OffloadContainer<T>>,
}

impl<T: BlockMetadata> OffloadBatch<T> {
    /// Total blocks across all containers
    fn total_blocks(&self) -> usize {
        self.containers.iter().map(|c| c.blocks.len()).sum()
    }

    /// Remove cancelled containers, return count removed
    fn sweep_cancelled(&mut self) -> usize {
        let before = self.containers.len();
        self.containers.retain(|c| !c.is_cancelled());
        before - self.containers.len()
    }

    /// Check if batch is empty
    fn is_empty(&self) -> bool {
        self.containers.is_empty()
    }
}
```

### 各阶段的数据变换

| 阶段 | 输入 | 输出 | 变换 |
|-------|-------|--------|-----------|
| Enqueue | `Vec<SourceBlock<T>>` | `OffloadContainer<T>` | 用 token + precondition 包裹 |
| PolicyEvaluator | `OffloadContainer<T>` | `OffloadContainer<T>` | 过滤 `blocks` 向量 |
| PreconditionAwaiter | `OffloadContainer<T>` | `OffloadContainer<T>` | 等待事件，置 `precondition = None` |
| Batcher | `OffloadContainer<T>` | `OffloadBatch<T>` | 按总 block 数分组 |
| TransferExecutor | `OffloadBatch<T>` | `Vec<ImmutableBlock<T>>` | Sweep → Upgrade → Flat map |

---

## 基于 Token 的取消（Cancellation）

### Token 生命周期

1. **创建**：在 enqueue 时创建一对 `CancellationToken`
2. **分发**：handle 拿到该 token，container 拿到一个克隆
3. **传播**：token 跟随 container 一起穿过流水线
4. **终止**：token 在 upgrade（提交点）处被消费

```rust,ignore
// At enqueue
let (cancel_token, cancel_updater) = CancellationToken::new();

// Give to handle
let handle = TransferHandle { cancel_token: cancel_token.clone(), ... };

// Give to container
let container = OffloadContainer {
    blocks,
    precondition: Some(event),
    cancel_token: cancel_token.clone(),
};
```

### CancellationToken API

```rust,ignore
impl CancellationToken {
    /// Request cancellation (called by handle)
    fn request(&self);

    /// Check if cancellation requested
    fn is_requested(&self) -> bool;

    /// Await cancellation request (for select!)
    async fn wait_requested(&self);

    /// Await confirmation that all blocks released
    fn wait_confirmed(&self) -> CancelConfirmation;
}
```

### PreconditionAwaiter 的 select 模式

awaiter 使用 `select!` 同时处理事件完成与取消：

```rust,ignore
async fn process(&self, mut container: OffloadContainer<T>) {
    // Fast path: event already satisfied
    if let Some(ref event) = container.precondition {
        if event.is_done() {
            container.precondition = None;
            self.output_queue.push(container);
            return;
        }
    }

    // Slow path: select on event OR cancellation
    if let Some(event) = container.precondition.take() {
        tokio::select! {
            _ = event.wait() => {
                // Event satisfied, propagate
                self.output_queue.push(container);
            }
            _ = container.cancel_token.wait_requested() => {
                // Cancelled while waiting - drop container
                tracing::debug!("Container cancelled during precondition wait");
                // container dropped here
            }
        }
    } else {
        // No precondition, pass through
        self.output_queue.push(container);
    }
}
```

### CancellableQueue 的 sweep 机制

队列通过 sweep 支持主动取消：

```rust,ignore
impl<T: HasCancellationToken> CancellableQueue<T> {
    /// Push item, reject if already cancelled
    fn push(&self, item: T) -> bool {
        if item.cancel_token().is_requested() {
            return false;
        }
        self.inner.push(item);
        true
    }

    /// Pop, skipping cancelled items
    fn pop_valid(&self) -> Option<T> {
        loop {
            match self.inner.pop() {
                Some(item) if item.cancel_token().is_requested() => continue,
                other => return other,
            }
        }
    }

    /// Remove all cancelled items
    fn sweep(&self) -> usize {
        let mut removed = 0;
        let mut kept = Vec::new();

        while let Some(item) = self.inner.pop() {
            if item.cancel_token().is_requested() {
                removed += 1;
            } else {
                kept.push(item);
            }
        }

        for item in kept {
            self.inner.push(item);
        }
        removed
    }
}
```

### Batch 级别的 sweep

对 `CancellableQueue<OffloadBatch<T>>`，sweep 会移除 batch 内被取消的 container：

```rust,ignore
fn sweep(&self) -> usize {
    let mut removed_containers = 0;
    let mut kept_batches = Vec::new();

    while let Some(mut batch) = self.inner.pop() {
        // Remove cancelled containers from this batch
        removed_containers += batch.sweep_cancelled();

        // Keep batch if it still has containers
        if !batch.is_empty() {
            kept_batches.push(batch);
        }
    }

    for batch in kept_batches {
        self.inner.push(batch);
    }
    removed_containers
}
```

### 各阶段的取消

| 阶段 | 机制 | 行为 |
|-------|-----------|----------|
| PolicyEvaluator | Token 检查 | 在 block 评估之间检查 `is_cancelled()` |
| PreconditionAwaiter | `select!` | 等待期间被取消则立即丢弃 |
| Batcher Queue | CancellableQueue | sweep 移除被取消的 container |
| Executor Queue | CancellableQueue | sweep 移除 batch 内被取消的 container |
| TransferExecutor | 最终 sweep | 在 upgrade 前执行 `batch.sweep_cancelled()` |

### Upgrade 处的取消边界

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                        CANCELLABLE ZONE                                 │
│                                                                         │
│  Enqueue → PolicyEval → PrecondAwaiter → Batcher → ExecutorQueue        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                                            │
                                                            ▼
                                                 ┌───────────────────┐
                                                 │  sweep_cancelled  │
                                                 │  (last check)     │
                                                 └───────────────────┘
                                                            │
                                                            ▼
═══════════════════════════════════════════════════════════════════════════
                              UPGRADE BOUNDARY
═══════════════════════════════════════════════════════════════════════════
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        COMMITTED ZONE                                   │
│                                                                         │
│  Upgrade → Flat Map → Transfer                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## TransferExecutor 设计

### Sweep → Upgrade → Flat Map → Transfer

```rust,ignore
impl<T: BlockMetadata, D: TransferDestination> TransferExecutor<T, D> {
    async fn run(self) {
        while let Some(mut batch) = self.input_queue.pop() {
            // 1. SWEEP: Last cancellation check
            batch.sweep_cancelled();

            if batch.is_empty() {
                continue;
            }

            // 2. UPGRADE: Weak → Strong (commitment point)
            let upgraded: Vec<UpgradedContainer<T>> = batch
                .containers
                .into_iter()
                .filter_map(|c| c.upgrade())
                .collect();

            if upgraded.is_empty() {
                continue;
            }

            // 3. FLAT MAP: Consolidate into single vec
            let all_blocks: Vec<ImmutableBlock<T>> = upgraded
                .into_iter()
                .flat_map(|c| c.blocks)
                .collect();

            // 4. TRANSFER: Execute via destination
            self.destination.execute_transfer(all_blocks).await;
        }
    }
}
```

### 通用的 TransferDestination Trait

```rust,ignore
trait TransferDestination {
    type Output;

    async fn execute_transfer(
        &self,
        blocks: Vec<ImmutableBlock<T>>,
        src_layout: LogicalLayoutHandle,
    ) -> Result<Self::Output>;
}
```

### Block 目标（G2、G3）

向另一个 `BlockManager` 传输：

```rust,ignore
struct BlockDestination<Dst: BlockMetadata> {
    leader: Arc<InstanceLeader>,
    dst_manager: Arc<BlockManager<Dst>>,
    src_layout: LogicalLayoutHandle,
    dst_layout: LogicalLayoutHandle,
}

impl<Dst: BlockMetadata> TransferDestination for BlockDestination<Dst> {
    type Output = Vec<ImmutableBlock<Dst>>;

    async fn execute_transfer(&self, blocks: Vec<ImmutableBlock<_>>) -> Result<Self::Output> {
        // 1. Allocate destination blocks
        let dst_blocks = self.dst_manager.allocate_blocks(blocks.len())?;

        // 2. Execute transfer via leader
        let notification = self.leader.execute_local_transfer(
            self.src_layout,
            self.dst_layout,
            src_block_ids,
            dst_block_ids,
        )?;
        notification.await?;

        // 3. Register destination blocks
        let registered = dst_blocks.into_iter()
            .zip(sequence_hashes)
            .map(|(block, hash)| self.dst_manager.register_with_hash(block, hash))
            .collect();

        Ok(registered)
    }
}
```

### Object 目标（G4）

向对象存储传输：

```rust,ignore
struct ObjectDestination {
    object_ops: Arc<dyn ObjectBlockOps>,
    src_layout: LogicalLayoutHandle,
    lock_manager: Option<Arc<dyn ObjectLockManager>>,
}

impl TransferDestination for ObjectDestination {
    type Output = Vec<SequenceHash>;

    async fn execute_transfer(&self, blocks: Vec<ImmutableBlock<_>>) -> Result<Self::Output> {
        // 1. Extract keys and block IDs
        let keys: Vec<SequenceHash> = blocks.iter().map(|b| b.sequence_hash()).collect();
        let block_ids: Vec<BlockId> = blocks.iter().map(|b| b.block_id()).collect();

        // 2. Execute object put
        let results = self.object_ops.put_blocks(keys.clone(), self.src_layout, block_ids).await;

        // 3. Handle lock management
        if let Some(lock_manager) = &self.lock_manager {
            for hash in &successful_hashes {
                lock_manager.create_meta(*hash).await?;
                lock_manager.release_lock(*hash).await?;
            }
        }

        Ok(successful_hashes)
    }
}
```

---

## Batcher 设计

### 对 container 进行分组

batcher 累积 container，并在以下任一情况触发 flush：
- 总 block 数达到 `max_batch_size`
- flush interval 到期且满足 `min_batch_size`
- 一次传输的全部 block 已被处理（哨兵 flush）

```rust,ignore
struct Batcher<T: BlockMetadata> {
    config: BatchConfig,
    input_queue: Arc<CancellableQueue<OffloadContainer<T>>>,
    output_queue: Arc<CancellableQueue<OffloadBatch<T>>>,
    current_batch: OffloadBatch<T>,
}

impl<T: BlockMetadata> Batcher<T> {
    async fn run(mut self) {
        let mut flush_timer = tokio::time::interval(self.config.flush_interval);

        loop {
            tokio::select! {
                _ = flush_timer.tick() => {
                    self.try_flush().await;
                }
                Some(container) = self.input_queue.pop_valid() => {
                    self.current_batch.containers.push(container);

                    if self.current_batch.total_blocks() >= self.config.max_batch_size {
                        self.flush().await;
                    }
                }
            }
        }
    }

    async fn try_flush(&mut self) {
        if self.current_batch.total_blocks() >= self.config.min_batch_size {
            self.flush().await;
        }
    }

    async fn flush(&mut self) {
        if self.current_batch.is_empty() {
            return;
        }

        let batch = std::mem::replace(
            &mut self.current_batch,
            OffloadBatch { containers: Vec::new() },
        );

        self.output_queue.push(batch);
    }
}
```

### 保留每个 container 的可取消性

每个 container 都保留自己的 `cancel_token`。当 batch 进入 executor 队列后：

1. **队列级别 sweep**：从 batch 中移除被取消的 container
2. **executor 级别 sweep**：在 upgrade 前做最终检查
3. **部分取消**：某些 container 可能被取消，而其他 container 继续推进

---

## 扩展规则

### 添加新的策略（policy）

1. 实现 `OffloadPolicy` trait
2. 加入流水线配置
3. 策略必须足够快或与 async 兼容

```rust,ignore
trait OffloadPolicy<T: BlockMetadata>: Send + Sync {
    fn name(&self) -> &str;
    fn evaluate(&self, ctx: &EvalContext<T>) -> impl Future<Output = Result<bool>>;
}
```

### 添加新的目标（destination）类型

1. 实现 `TransferDestination` trait
2. 创建一个新的流水线变体，或使用通用 executor
3. 处理目标特有的注册/清理逻辑

### 维持取消不变量

修改流水线时：

1. **绝不可越过 upgrade 边界** —— 这是提交点
2. **upgrade 前必须 sweep** —— 最后一次取消机会
3. **token 必须随 container 一起流动** —— 不要过早剥离它
4. **batch 应保留 container 的身份** —— 直到 flat map 之前

---

## 测试指引

### 单元测试

- 对每个阶段进行独立测试
- 用 mock 的 `CancellationToken` 验证取消场景
- 验证 sweep 移除的是正确的项

### 集成测试

- 在每个阶段触发取消，对完整流水线进行测试
- 取消后不应有遗留的 block
- 测试 batch 的部分取消

### 性能测试

- 测量取消检查的额外开销
- 大规模下对 sweep 操作做基准测试
- 对 upgrade → flat map → transfer 路径做 profile
