# Worker Group 模块

worker group 模块提供了一组抽象，用于从单一 leader 并行驱动多个 worker。

## ParallelWorkers Trait

`ParallelWorkers` 在一组 worker 的层面上扩展了 `WorkerTransfers + ObjectBlockOps`，新增以下方法：

- `export_metadata()` → `Vec<SerializedLayoutResponse>`（每个 rank 一个）
- `import_metadata(Vec<SerializedLayout>)` → `Vec<ImportMetadataResponse>`
- `worker_count()` → worker 数量
- `workers()` → 底层 `Arc<dyn Worker>` 切片

## SpmdParallelWorkers

`SpmdParallelWorkers` 实现了 SPMD（Single Program, Multiple Data，单程序多数据）执行模型：将同一操作并行广播到每个 worker，并聚合结果。

### Fan-out 执行

每个 `WorkerTransfers` 方法（本地传输、远程 onboard、远程 offload）都会遍历所有 worker 并在每个 worker 上调用相同的操作。各 worker 并行执行——每个 worker 都会把共享的逻辑布局句柄解析为自己的物理布局。

### Rank 感知路由

对于 `connect_remote`，每个 worker 会收到针对其 rank 的元数据切片。远程句柄映射存储为 `(InstanceId, worker_idx, LogicalLayoutHandle) → LayoutHandle`，因此 `execute_remote_onboard_for_instance` 可以按 rank 为每个 worker 查找正确的远程句柄。

### 事件聚合

各个 worker 发出的传输完成通知，会通过事件系统聚合为单个 `TransferCompleteNotification`。聚合后的通知只在所有 worker 都完成后才会触发。

### ObjectBlockOps 聚合

- `has_blocks`：查询所有 worker，返回 worker 0 的结果（在 SPMD 语义下所有 worker 应该一致）。
- `put_blocks` / `get_blocks`：在所有 worker 上并行执行。某个 key 只有在**所有** worker 都对该 key 成功时才视为成功。

### 构造方式

```rust,ignore
let parallel = SpmdParallelWorkers::new(
    workers,        // Vec<Arc<dyn Worker>>, one per rank
    event_manager,  // Arc<EventManager> for aggregation
    runtime_handle, // tokio::runtime::Handle for spawning
);
```
