# Offload 模块

offload 模块负责管理 KV 缓存（KV cache）块在不同存储层级之间的异步传输。它提供了基于管线（pipeline）的架构，用于评估、批处理和执行块传输，并完整支持取消（cancellation）。

## 概述

Offload 将块从源层级（如 GPU 内存）移动到目标层级（如主机内存、远程存储或对象存储）。该管线确保：

- **基于策略的过滤**：仅传输满足条件的块
- **批量执行**：将块分组以提高传输效率
- **取消支持**：传输在提交（commitment）前的任何时刻都可以取消
- **前置条件同步**：传输会等待前向计算（forward pass）完成

## 管线架构

```text
┌─────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐     ┌──────────────────┐
│ PolicyEvaluator │────►│ PreconditionAwaiter │────►│       Batcher       │────►│ TransferExecutor │
└─────────────────┘     └─────────────────────┘     └─────────────────────┘     └──────────────────┘
                                                             ▲                          ▲
                                                             │                          │
                                                    CancellableQueue          CancellableQueue
                                                             │                          │
                                                             └──────── CancelSweeper ───┘
```

### 各阶段

| 阶段 | 用途 |
|-------|---------|
| **PolicyEvaluator** | 根据已配置的策略（频率、是否存在等）过滤块 |
| **PreconditionAwaiter** | 等待前向计算完成后再继续 |
| **Batcher** | 根据总块数将容器分组成批次 |
| **TransferExecutor** | 升级（upgrade）块并执行真正的传输 |

## 容器数据模型

在管线中流动的基本单元是 **OffloadContainer**：

```rust,ignore
struct OffloadContainer<T: BlockMetadata> {
    /// The blocks to offload
    blocks: Vec<SourceBlock<T>>,
    /// Precondition event (forward pass completion)
    precondition: Option<EventHandle>,
    /// Cancellation token
    cancel_token: CancellationToken,
}
```

容器被分组到批次中以便高效传输：

```rust,ignore
struct OffloadBatch<T: BlockMetadata> {
    /// Multiple containers, each independently cancellable
    containers: Vec<OffloadContainer<T>>,
}
```


### P1：容器是取消的最小单位

容器内的单个块不可独立取消。当容器被取消时，其中所有块一并被取消。

### P2：Token 随容器一起流动

每个容器都带有自己的 `CancellationToken`，在入队时从 `TransferHandle` 克隆得到。该 token 会随容器穿过管线的各个阶段，直到 upgrade。

### P3：Upgrade 是提交边界

upgrade 步骤（Weak → Strong）是不可逆的提交点：

- **upgrade 之前**：可以通过 sweep 或 token 检查取消容器
- **upgrade 之后**：我们已经持有这些块，取消不再适用

### P4：Upgrade 前最后一次 Sweep

最后一次取消检查在 upgrade 前立刻执行。`TransferExecutor` 会在提交前调用 `batch.sweep_cancelled()` 移除已取消的容器。

### P5：Upgrade 后扁平化

upgrade 后，所有容器中的所有块都会被合并为单个 `Vec<ImmutableBlock<T>>`，以便高效批量传输。此时单个容器的标识信息丢失。

### P6：PreconditionAwaiter 使用 Select

precondition awaiter 通过 `select!` 同时等待前置条件事件和取消 token。如果在等待时被取消，容器会被立刻丢弃。

## 配置

管线行为由 `PipelineConfig` 控制：

| 选项 | 默认值 | 说明 |
|--------|---------|-------------|
| `batch_config.max_batch_size` | 64 | 每个批次最大块数 |
| `batch_config.min_batch_size` | 8 | 在刷新（flush）前所需的最小块数 |
| `batch_config.flush_interval` | 10ms | 刷新部分批次前等待的时间 |
| `policy_timeout` | 100ms | 策略评估的超时 |
| `sweep_interval` | 10ms | cancel sweeper 的扫描间隔 |
| `max_concurrent_transfers` | 1 | 并发传输批次数 |

## 用法

### 块入队

```rust,ignore
let handle = pipeline.enqueue(source_blocks, precondition_event);

// Track progress
println!("Status: {:?}", handle.status());

// Wait for completion
let result = handle.wait().await?;
```

### 取消传输

```rust,ignore
// Request cancellation and wait for confirmation
handle.cancel().await;
// All blocks are now released
```

## 相关文档

- [offload-developer.md](offload-developer.md) - 实现细节和扩展规则
