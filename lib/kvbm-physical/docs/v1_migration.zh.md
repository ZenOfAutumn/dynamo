# 迁移指南：从 block_manager 到 kvbm-physical

从 `dynamo-llm::block_manager`（v1）迁移到 `kvbm-physical` 的指南。

## 概览

`kvbm-physical` 是从零开始重写的物理传输层，对应原先的 `lib/llm/src/block_manager/`。核心数据流与原实现一致（注册 layout、交换元数据、执行传输），但 `kvbm-physical` 增加了块格式（block format）感知、更丰富的传输选项，以及逻辑层级（logical tier）与物理句柄之间更清晰的分离。

两者都使用相同的 `vectorized_copy` CUDA 内核（kernel）。原实现将其嵌入到 `.fatbin`（`lib/llm/src/block_manager/block/transfer/kernels/vectorized_copy.fatbin`）中通过 `cuModuleLoadData` 加载。`kvbm-physical` 通过 `kvbm-kernels` crate 用显式 Rust FFI 包装相同内核，以提升透明度与可测性。

## 类型对应表

| 原实现（block_manager） | kvbm-physical | 说明 |
|--------------------------|---------------|-------|
| `TransportManager` | `TransferManager` | 角色相同，API 更丰富 |
| `LayoutHandle` | `LayoutHandle` | 概念相同；编码已更改 — 详见 LayoutHandle 文档 |
| `PhysicalLayout` + builder | `PhysicalLayout` + builder | 模式相同；新增 `with_external_device_regions()` |
| `LayoutConfig` | `LayoutConfig` | 字段相同 + 可选 `num_heads` |
| `TransferOptions` | `TransferOptions` | 新增 `cuda_stream`、`src_kv_layout`、`dst_kv_layout` |
| `TransferCapabilities` | `TransferCapabilities` | 相同 |
| `TransferPreferences` | `TransferPreferences` | 相同 |
| `SerializedLayout` | `SerializedLayout` | wire 格式概念相同 |
| `WorkerAddress` | `WorkerAddress` | 相同 |
| `TransferCompleteNotification`（oneshot） | `TransferCompleteNotification`（`Either`/`EventAwaiter`） | 零成本同步路径 |
| `BounceBufferSpec`（trait object） | `BounceBuffer`（具体结构体） | 更简单，无堆分配 |
| N/A | `LogicalLayoutDescriptor` | **新增** — 跨层级桥接 |
| N/A | `KvBlockLayout` | **新增** — 块格式感知 |
| N/A | `KvBlocks` | **新增** — 带 layout 覆盖的分组块 |
| `CudaBlockingH2D` / `CudaBlockingD2H` | 已移除 | 仅异步；如需同步直接 `.await` |
| `OperationalCopyBackend` | 已移除 | 由 `kvbm_kernels` 直接 FFI 替代 |

## kvbm-physical 新增内容

### LogicalLayoutDescriptor

将 `LayoutHandle`（物理）桥接到 `LogicalLayoutHandle`（G1/G2/G3/G4 层级）。这是跨多 worker 协调的关键新抽象：调用方只需声明"从 G1 拷贝到 G2"，由 `TransferManager` 解析具体 worker 的句柄。

```rust,ignore
// Build descriptor for RDMA exchange
let descriptor = manager.build_logical_descriptor(gpu_handle, LogicalLayoutHandle::G1)?;
```

### KvBlockLayout

包含 5 种命名块格式以及 `Custom` 与 `Unknown`。可基于类型驱动地为不同维度顺序之间的传输选择 kernel。

```rust,ignore
let needs_permute = src_layout.requires_transform(&dst_layout);
```

### kvbm-kernels FFI

`kvbm_kernels` crate 提供 `memcpy_batch`，使用 CUDA 12.9+ 的 batch API，并在不可用时自动回退到逐次拷贝。这取代了原本基于 fatbin 加载的方式，改为直接 Rust FFI。

### 流（stream）池化

4 个 H2D + 4 个 D2H 流，按轮询选择，取代原本 1+1 的流配对。降低并发传输的争用。

### 调用方提供的 CUDA 流

`TransferOptions::cuda_stream` 允许调用方传入 stream。执行器会跳过事件记录；同步由调用方管理。这对 layer-wise 传输（要求所有 layer 使用同一 stream）非常有用。

```rust,ignore
let stream = manager.context().acquire_h2d_stream();
let options = TransferOptions::builder()
    .cuda_stream(stream.clone())
    .build()?;
```

### CudaMemPool

用于内核临时分配（permute buffer 等）的设备内存池。通过 `TransferConfig` 配置：

```rust,ignore
TransferManager::builder()
    .cuda_pool_reserve_size(64 * 1024 * 1024)         // 64 MiB pre-allocated
    .cuda_pool_release_threshold(Some(64 * 1024 * 1024)) // free above this
    .build()?;
```

### TransferCompleteNotification::aggregate()

将多个传输完成通知合并为一个，仅当全部完成时才完成。当所有输入都已完成时会优化掉聚合开销。

```rust,ignore
let combined = TransferCompleteNotification::aggregate(
    vec![n1, n2, n3],
    manager.context().event_system(),
    &tokio::runtime::Handle::current(),
)?;
combined.await?;
```

### src/dst kv_layout 覆盖

`TransferOptions` 现支持覆盖源端与目的端的块 layout 解读，从而无需修改已注册的 layout 即可实现跨格式传输。

```rust,ignore
let options = TransferOptions::builder()
    .src_kv_layout(KvBlockLayout::OperationalNHD)
    .dst_kv_layout(KvBlockLayout::UniversalTP)
    .build()?;
```

## 有意移除的内容

### 阻塞式 CUDA 策略

`CudaBlockingH2D` 与 `CudaBlockingD2H` 已被移除。所有传输均为异步。如需同步行为，立即 `.await` 即可：

```rust,ignore
// v1 (blocking)
let result = blocking_h2d_transfer(...);

// kvbm-physical (async, but can be used synchronously)
let notification = manager.execute_transfer(...)?;
notification.await?;
```

### OperationalCopyBackend 枚举

用于在不同 kernel 加载策略之间选择的 `OperationalCopyBackend` 枚举已被移除。`kvbm-physical` 全面采用 `kvbm_kernels` 直接 FFI，使 kernel 调度变得透明。

### Trait object 形式的 bounce buffer

`BounceBufferSpec`（需要堆分配的 trait object）已被 `BounceBuffer` 取代 — 后者是包装 `LayoutHandle` + 块 ID 的具体结构体：

```rust,ignore
// v1
struct MyBounce { layout: PhysicalLayout, blocks: Vec<BlockId> }
impl BounceBufferSpec for MyBounce { ... }

// kvbm-physical
let bounce = BounceBuffer::from_handle(host_handle, vec![0, 1, 2, 3]);
```

## 迁移步骤

### 1. 用 TransferManager 替换 TransportManager

builder 模式相同。`TransferManager::builder()` 返回同类的链式 builder。

```rust,ignore
// v1
let manager = TransportManager::builder()
    .worker_id(0)
    .nixl_backend("ucx")
    .cuda_device_id(0)
    .build()?;

// kvbm-physical
let manager = TransferManager::builder()
    .nixl_backend("ucx")
    .cuda_device_id(0)
    .build()?;
// worker_id is now derived from the event system
```

### 2. 替换 TransferOptions

按需添加新字段。原有的 `layer_range` 与 `nixl_write_notification` 用法保持不变。

```rust,ignore
// v1
let options = TransferOptions::builder()
    .layer_range(0..16)
    .build()?;

// kvbm-physical (same, with optional new fields)
let options = TransferOptions::builder()
    .layer_range(0..16)
    .cuda_stream(stream)        // new: caller-managed stream
    .src_kv_layout(layout)      // new: format override
    .build()?;
```

### 3. 用 BounceBuffer 替换 BounceBufferSpec

```rust,ignore
// v1 — trait object
let spec: Box<dyn BounceBufferSpec> = Box::new(MyBounce::new(layout, blocks));
options.bounce_buffer(spec);

// kvbm-physical — concrete type
let bounce = BounceBuffer::from_handle(host_handle, block_ids);
let options = TransferOptions::builder()
    .bounce_buffer(bounce)
    .build()?;
```

### 4. 替换 TransferCompleteNotification 的 await 模式

通知现在直接实现 `IntoFuture`，无需再包装 oneshot 通道。

```rust,ignore
// v1
let notification = manager.execute_transfer(...)?;
notification.recv().await??;

// kvbm-physical
let notification = manager.execute_transfer(...)?;
notification.await?;
```

### 5. 增加 LogicalLayoutDescriptor 以支持多 worker 层级解析

如果你按层级名称（G1、G2 等）跨多个 worker 协调传输，请使用 `LogicalLayoutDescriptor`：

```rust,ignore
// Build descriptors that include tier information
let g1_desc = manager.build_logical_descriptor(gpu_handle, LogicalLayoutHandle::G1)?;
let g2_desc = manager.build_logical_descriptor(host_handle, LogicalLayoutHandle::G2)?;

// Remote workers can now resolve "copy G1 to G2" to the correct physical handles
```

### 6. 为跨格式传输考虑使用 KvBlockLayout 注解

如果你的传输涉及以不同维度顺序存储的块（例如来自引擎的 operational NHD 与用于存储的 universal TP），可以用 `KvBlockLayout` 注解：

```rust,ignore
let options = TransferOptions::builder()
    .src_kv_layout(KvBlockLayout::OperationalNHD)
    .dst_kv_layout(KvBlockLayout::UniversalTP)
    .build()?;
```

这会告知执行器选择 permute kernel 而非直接拷贝。
