# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在本仓库中工作时提供指导。

## 构建命令

```bash
# Build (from repo root or this directory)
cargo build -p kvbm-logical

# Run all tests (263 tests)
cargo test -p kvbm-logical --lib

# Run a single test
cargo test -p kvbm-logical --lib test_name

# Run tests in a specific module
cargo test -p kvbm-logical --lib registry::tests

# Lint
cargo clippy -p kvbm-logical --lib

# Build with test utilities exposed (for downstream crates)
cargo build -p kvbm-logical --features testing
```

## 架构

**kvbm-logical** 是 KVBM（KV Block Manager）的核心逻辑块（block）生命周期管理器。它通过类型安全的状态机、注册表（registry）和池（pool）系统，为 LLM 推理（inference）管理 KV 缓存（KV cache）块。

### Block 生命周期（类型状态模式）

Block 使用编译期类型状态来强制保证合法的状态迁移：

```
MutableBlock<T>  →  CompleteBlock<T>  →  ImmutableBlock<T>  →  WeakBlock<T>
   (Reset)             (Staged)            (Registered)        (Non-owning)
```

- **MutableBlock**：从 `ResetPool` 中分配，可写。Drop 时归还到 reset pool。
- **CompleteBlock**：通过 `stage()`/`complete()` 进入暂存（staged）状态。Drop 时归还到 reset pool。
- **ImmutableBlock**：在 registry 中注册。强 Arc 引用可防止被驱逐（eviction）。Drop 时移入 inactive pool。
- **WeakBlock**：非占有引用，不阻止驱逐。两阶段升级（先快速 `Weak::upgrade`，失败后回退到慢速 pool 搜索）。

类型参数 `T: BlockMetadata` 是存储层级的标记（marker）：G1=GPU，G2=CPU，G3=Disk，G4=External。

### 模块结构

- **`manager/`** — `BlockManager<T>`：顶层编排器（orchestrator）。是分配、注册、匹配以及驱逐 block 的入口。使用 builder 模式（`BlockManagerConfigBuilder`）。
- **`blocks/`** — 每个生命周期状态对应的 RAII 守卫类型。所有守卫在 drop 时自动把 block 归还到正确的 pool。
- **`registry/`** — `BlockRegistry`：在 `PositionalRadixTree` 中按 `SequenceHash` 跟踪已注册的 block。支持类型化挂载（attachment）、存在性标记（presence marker）和 touch 回调。可选的 TinyLFU 频率跟踪。
- **`pools/`** — 三层池系统：
  - `ResetPool<T>`：空闲 block（FIFO）
  - `ActivePool<T>`：使用中的已注册 block
  - `InactivePool<T>`：可驱逐的缓存 block，后端可插拔
- **`pools/inactive/backends/`** — 实现 `InactivePoolBackend<T>` 的驱逐策略：`HashMapBackend`、`LruBackend`、`MultiLruBackend`（4 层频率感知）、`LineageBackend`（父链感知）。
- **`events/`** — Block 事件管线（pipeline），具有可配置的发射策略、批处理和广播通道。
- **`metrics/`** — Prometheus 指标，使用原子计数器（`BlockPoolMetrics`），可选的周期采样（`StatsCollector`）。
- **`tinylfu.rs`** — Count-Min Sketch 频率跟踪器（4 个哈希函数，可配置衰减）。
- **`testing/`** — 测试工具（在 `testing` feature flag 后）：`TestBlockBuilder`、`BlockSequenceBuilder`、`create_test_manager()`。

### 关键设计决策

- **同步核心 + 内部可变性**：池操作使用 `parking_lot` 锁，不使用 async 通道。RAII 归还以同步内联方式执行。
- **Registry 使用弱引用**：`PositionalRadixTree<Weak<BlockRegistrationHandleInner>>`。当所有强引用都被 drop 后，条目会自动清理。
- **挂载系统**：通过 `attach_unique<T>()`/`attach<T>()` 在 `BlockRegistrationHandle` 上扩展类型化的元数据，无需修改结构体。
- **`docs/advancements.md`** 包含详细的设计文档，对比 v1 与 kvbm-logical 架构。

## 测试

测试使用 `rstest` 作为 fixture 框架，使用 `proptest` 进行基于属性的测试（`pools/block_proptest.rs`）。`src/testing/` 中的测试工具受 `testing` feature flag 控制，对下游 crate 同样可用。
