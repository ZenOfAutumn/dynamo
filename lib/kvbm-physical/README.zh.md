# kvbm-physical

KV 缓存（KV cache）块存储的物理布局与传输管理。

`kvbm-physical` 提供了底层构建块，用于将 KV cache 块映射到内存、通过 NIXL 注册以支持 RDMA 传输，并在异构存储层（GPU、主机、磁盘、远端）之间执行传输。

## 模块

### `layout` — 块到内存的映射

KV cache 块在内存中如何组织的抽象。

- **`Layout` trait** — 核心抽象，将 `(block_id, layer_id, outer_id)` 映射到 `MemoryRegion`。实现包括完全连续（单一分配）和按层分离（每层一次分配）等变体。
- **`KvBlockLayout`** — 描述块内的维度顺序。包含五种命名格式（`UniversalTP`、`UniversalPP`、`OperationalHND`、`OperationalNHD`、`Custom`）以及 `Unknown`。提供 `requires_transform()`、`is_operational()` 和 `is_universal()` 用于内核选择。
- **`PhysicalLayout`** — 将 `Layout` 与其物理存储位置（`StorageKind`）和 NIXL 注册元数据（`NixlMetadata`）封装在一起。通过类型状态构建器构造：Config &rarr; Layout type &rarr; Memory allocation &rarr; `build()`。
- **`LayoutConfig`** — 块的维度信息：`num_blocks`、`num_layers`、`outer_dim`、`page_size`、`inner_dim`、`dtype_width_bytes`，以及可选的 `num_heads`。
- **`KvBlocks`** — 将块 ID 与共享的 `PhysicalLayout` 进行分组，并可选地通过 `KvBlockLayout` 覆盖以实现跨格式的传输。

### `manager` — 布局注册与传输编排

- **`TransferManager`** — 主要 API。注册布局、在 worker 之间导出/导入 RDMA 元数据，并按句柄执行传输。
- **`LayoutHandle`** — 紧凑的 `u128` 编码，包含 `(worker_id, layout_id)`。在特定 worker 中标识已注册的布局；在不同 worker 之间不对称。
- **`LogicalLayoutDescriptor`** — 将 `LayoutHandle` 桥接到 `LogicalLayoutHandle`（G1/G2/G3/G4 层）。允许调用方表达"从 G1 复制到 G2"，由 `TransferManager` 解析为各 worker 特定的物理句柄。
- **`SerializedLayout`** — 用于 RDMA 元数据交换的线格式。将 worker 地址、NIXL 元数据和布局描述符打包为 bincode 二进制块。
- **`WorkerAddress`** — `(worker_id, nixl_agent_name)` 二元组，标识网络上的一个 worker。

### `transfer` — 传输配置与执行

- **`TransferConfig` / builder** — 在构建 `TransferManager` 之前配置事件系统、NIXL 后端、CUDA 设备、能力和内存池。
- **`TransferOptions`** — 单次传输配置：`layer_range`、`nixl_write_notification`、`bounce_buffer`、调用方提供的 `cuda_stream`，以及源/目的的 `kv_layout` 覆盖。
- **`TransferPreferences`** — 通过 `NativeVsNixlPolicy`（PreferNative / PreferNixl / Automatic）提供策略提示。
- **`TransferCompleteNotification`** — `Either<Ready, EventAwaiter>`，实现了 `IntoFuture`。同步完成时零开销。`aggregate()` 可组合多个通知。`could_yield()` 用于检查 await 是否会挂起。
- **`BounceBuffer`** — 两跳传输的中转区（例如 Device &rarr; Host &rarr; Remote）。
- **校验和工具** — 使用 BLAKE3 计算块/层校验和以验证传输。
- **填充工具** — 用于测试和初始化的常量/顺序填充模式。

## 快速上手

```rust,ignore
use kvbm_physical::{TransferManager, TransferOptions};
use kvbm_physical::layout::{LayoutConfig, PhysicalLayout};

// 1. Build the TransferManager (creates NIXL agent, CUDA streams, event system)
let manager = TransferManager::builder()
    .nixl_backend("ucx")
    .cuda_device_id(0)
    .build()?;

// 2. Configure a layout
let config = LayoutConfig::builder()
    .num_blocks(64)
    .num_layers(32)
    .outer_dim(2)
    .page_size(16)
    .inner_dim(128)
    .dtype_width_bytes(2)
    .build()?;

// 3. Build a physical layout (type-state builder: config -> layout type -> memory -> build)
let gpu_layout = PhysicalLayout::builder(manager.nixl_agent().clone())
    .with_config(config.clone())
    .fully_contiguous()
    .allocate_device(0)
    .build()?;

let host_layout = PhysicalLayout::builder(manager.nixl_agent().clone())
    .with_config(config)
    .fully_contiguous()
    .allocate_pinned(Some(0))
    .build()?;

// 4. Register layouts to get handles
let gpu_handle = manager.register_layout(gpu_layout)?;
let host_handle = manager.register_layout(host_layout)?;

// 5. Execute a transfer and await completion
let notification = manager.execute_transfer(
    gpu_handle,
    &[0, 1, 2, 3],        // source block IDs
    host_handle,
    &[0, 1, 2, 3],        // destination block IDs
    TransferOptions::new(),
)?;
notification.await?;
```

## 测试

`kvbm-physical` 中所有功能性测试都需要真实的 NIXL 安装和一块 CUDA GPU。它们通过两个 feature flag 进行门控：

- **`testing-kvbm`** — 启用需要 NIXL 与 CUDA 的测试（创建 NixlAgent 实例、分配设备内存 / 启动 kernel）

### 运行测试

```bash
# Without GPU/NIXL — only the sentinel test runs (confirms skipping)
cargo test -p kvbm-physical

# With GPU + NIXL available
cargo test -p kvbm-physical --features testing-kvbm
```

未启用任何 feature 时，将只运行一个**哨兵测试**并打印提示信息。这能确保 `cargo test` 不会在零测试的情况下静默通过。

### 哨兵测试的样子

```
running 1 test
test sentinel::all_functional_tests_skipped___enable_testing_nixl_and_testing_cuda ... ok
```

`layout::tests` 中的 `test_version_check_on_deserialization` 是唯一无需 feature flag 即可运行的功能性测试，因为它不需要 NIXL 或 CUDA。

## 文档

- [v1 Migration Guide](docs/v1_migration.md) — 从 `dynamo-llm::block_manager` 迁移到 `kvbm-physical`
