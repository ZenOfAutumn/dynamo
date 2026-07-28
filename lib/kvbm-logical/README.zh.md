# kvbm-logical

KVBM（KV Block Manager，KV 块管理器）的逻辑块生命周期管理。通过类型安全的状态机、注册表与池系统，管理用于 LLM 推理（inference）的 KV 缓存（KV cache）块。

## 块生命周期

块通过 type-state 模式在编译期强制约束状态机：

```text
MutableBlock<T> → CompleteBlock<T> → ImmutableBlock<T> ⇄ WeakBlock<T>
   (Reset)           (Staged)          (Registered)       (Non-owning)
```

- **MutableBlock** —— 从 reset 池分配，可写。Drop 时归还到 reset 池。
- **CompleteBlock** —— 已使用 `SequenceHash` 暂存（staged）但尚未注册。Drop 时归还到 reset 池。
- **ImmutableBlock** —— 已在块注册表中注册。强引用阻止其被驱逐（eviction）。Drop 时进入 inactive 池用于缓存。
- **WeakBlock** —— 不持有所有权的引用，不阻止驱逐。可通过两阶段查找升级回 `ImmutableBlock`。

类型参数 `T: BlockMetadata` 用于标识存储分级（例如 GPU、CPU、磁盘）。

## 用法

```rust,no_run
use kvbm_logical::{
    BlockManager, BlockRegistry, MutableBlock, CompleteBlock, ImmutableBlock, WeakBlock,
    SequenceHash,
    manager::FrequencyTrackingCapacity,
};

# fn main() {
// 任何 Clone + Send + Sync + 'static 的类型都满足 BlockMetadata。
#[derive(Clone)]
struct G2; // CPU 分级标记

// 构建带 TinyLFU 频率跟踪的注册表。
let tracker = FrequencyTrackingCapacity::Medium.create_tracker();
let registry = BlockRegistry::builder()
    .frequency_tracker(tracker)
    .build();

// 使用 LRU 驱逐后端构建块管理器。
let manager = BlockManager::<G2>::builder()
    .block_count(1024)
    .block_size(16)
    .registry(registry)
    .with_lru_backend()
    .build()
    .expect("failed to build block manager");

// 从 reset 池分配可变块。
let mut blocks: Vec<MutableBlock<G2>> = manager
    .allocate_blocks(2)
    .expect("not enough blocks available");

// 使用预先计算好的 sequence hash 暂存（stage）一个块，得到 CompleteBlock。
// SequenceHash 包装了从 token 数据计算得到的位置感知（positional）lineage 哈希。
let seq_hash_0 = SequenceHash::new(42, None, 0);
let complete: CompleteBlock<G2> = blocks
    .remove(0)
    .stage(seq_hash_0, manager.block_size())
    .expect("block size should match");

// 注册暂存的块，得到 ImmutableBlock。
let immutable: ImmutableBlock<G2> = manager.register_block(complete);

// 通过 sequence hash 进行前缀匹配。
let matched: Vec<ImmutableBlock<G2>> = manager.match_blocks(&[seq_hash_0]);
assert_eq!(matched.len(), 1);

// 降级为 WeakBlock（不阻止驱逐）。
let weak: WeakBlock<G2> = immutable.downgrade();

// 若该块尚未被驱逐，可升级回 ImmutableBlock。
if let Some(restored) = weak.upgrade() {
    assert_eq!(restored.sequence_hash(), seq_hash_0);
}

// RAII：丢弃 ImmutableBlock 时会将其移入 inactive 池以供缓存。
{
    let temporary = manager.match_blocks(&[seq_hash_0]);
    // `temporary` 在此处被丢弃 → 块归还到 inactive 池
}

// 检查池状态。
let available = manager.available_blocks();
let total = manager.total_blocks();
# }
```

## Prometheus 指标（Metrics）

所有指标都带有 `pool` 标签以标识存储分级。

### 计数器（Counters）

| 名称 | 描述 |
|------|-------------|
| `kvbm_allocations_total` | 从池分配的块总数 |
| `kvbm_allocations_from_reset_total` | 从 reset 池分配的块总数 |
| `kvbm_evictions_total` | 从 inactive 池驱逐的块总数 |
| `kvbm_registrations_total` | 已注册的块总数（CompleteBlock → ImmutableBlock）|
| `kvbm_duplicate_blocks_total` | 创建的重复块总数（Allow 策略）|
| `kvbm_registration_dedup_total` | 被去重的块注册总数（Reject 策略）|
| `kvbm_stagings_total` | MutableBlock → CompleteBlock 转换总数 |
| `kvbm_match_hashes_requested_total` | match_blocks 调用中请求的哈希总数 |
| `kvbm_match_blocks_returned_total` | match_blocks 调用返回的块总数 |
| `kvbm_scan_hashes_requested_total` | scan_matches 调用中请求的哈希总数 |
| `kvbm_scan_blocks_returned_total` | scan_matches 调用返回的块总数 |

### 仪表（Gauges）

| 名称 | 描述 |
|------|-------------|
| `kvbm_inflight_mutable` | 当前位于池外的 MutableBlock 数量 |
| `kvbm_inflight_immutable` | 当前位于池外的 ImmutableBlock 数量 |
| `kvbm_reset_pool_size` | 当前 reset 池大小 |
| `kvbm_inactive_pool_size` | 当前 inactive 池大小 |
