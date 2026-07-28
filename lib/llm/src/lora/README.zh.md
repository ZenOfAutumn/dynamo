# LoRA 适配器下载、缓存与路由（lora）

> 提供 LoRA 适配器从多种来源（本地、S3）下载并缓存的基础设施，以及在集群多个 worker 间分配 LoRA 的路由/分配算法。

## 这个模块解决什么问题

LoRA（Low-Rank Adaptation）是一种轻量微调产物：在基座模型之上加一小组适配器权重。在分布式推理里要解决两件事：（1）按需把某个 LoRA 适配器从来源（`file://` 本地、`s3://` 对象存储）下载到本地并缓存，供引擎加载；（2）当集群有多个 worker 时，决定每个 LoRA 应该放在哪些 worker 上（副本分配），并保证同名 LoRA 的请求被路由到持有它的 worker。`lora` 模块同时覆盖"下载/缓存"和"分配/路由"两条线，外加一个用于辅助分配决策的负载估计器。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `lora.rs`（父模块，目录外） | 模块入口，`pub use` 重导出对外 API（`LoRACache`、`LoRADownloader`、`LoadEstimator`、`LoraAllocator` 等） |
| `cache.rs` | `LoRACache`：本地缓存目录管理，URI→cache key 转换、缓存命中与文件校验 |
| `downloader.rs` | `LoRADownloader`：编排"先查缓存、否则按来源顺序下载并校验"的流程 |
| `source.rs` | `LoRASource` trait 及两个实现 `LocalLoRASource`、`S3LoRASource`（含重试与原子落盘） |
| `load_estimator.rs` | `LoadEstimator`：按时间序列跟踪每个 LoRA 的活跃请求数，支持轮询模式与事件订阅模式 |
| `routing/mod.rs` | `LoraAllocator` trait、`AllocationAlgorithmType` 枚举、工厂函数 `create_lora_allocator`，以及 `Random` 分配实现 |
| `routing/hrw.rs` | `RendezvousHasher`：基于 blake3 的 HRW（最高随机权重）一致性哈希分配算法 |
| `routing/table.rs` | `LoraRoutingTable` / `LoraReplicaConfig`：线程安全的"LoRA→副本集"分配表 |

## 核心概念（Java 视角）

- **`LoRASource`（trait）**：等价于 Java 的 `interface`，定义 `async fn download(...)` 和 `async fn exists(...)`。用 `#[async_trait]` 让 trait 支持异步方法（`.await` ≈ CompletableFuture）。`LocalLoRASource`、`S3LoRASource` 是两个实现类。`LoRADownloader` 持有 `Vec<Arc<dyn LoRASource>>`，相当于 `List<LoRASource>` 的多态来源列表，按顺序尝试。
- **`LoRACache`**：纯本地缓存管理类。`uri_to_cache_key` 是静态方法（关联函数），把 `s3://bucket/x` 这类 URI 规范化成文件名安全的 key，保证 Rust 与 Python 侧生成一致。`from_env()` 读取 `DYN_LORA_PATH` 环境变量。
- **`LoraAllocator`（trait）+ `RendezvousHasher`**：分配算法接口。HRW 哈希对每个 `(lora_name, worker)` 用 blake3 算一个分数，取分数最高的前 N 个 worker 作为副本集。它的关键性质是**稳定性**：增删 worker 时，已有放置尽量不变（类似一致性哈希），避免大规模重新搬运适配器。`AllocationAlgorithmType` 是带变体的 `enum`（≈ sealed class），通过 `create_lora_allocator` 工厂返回 `Box<dyn LoraAllocator>`（≈ 返回接口类型的多态对象）。
- **`LoraRoutingTable`**：`Arc<DashMap<String, LoraReplicaConfig>>`，即线程安全的并发 Map（≈ Java 的 `ConcurrentHashMap` 包在共享指针里），存"哪个 LoRA 当前分配到了哪些 worker"。`Clone` 共享同一份底层数据。
- **`LoadEstimator`**：用 `DashMap<String, LoraLoadData>` 记录每个 LoRA 的活跃计数与历史采样。两种驱动方式：`start_polling`（单 router，定时 `interval.tick()` 从 `KvScheduler` 拉计数）和 `start_event_subscription`（多 router，订阅 `ActiveSequenceEvent` 事件增减计数）。两者都用 `tokio::select!` + `CancellationToken` 优雅退出（≈ 可中断的后台线程）。

## 与其他模块的关系

- `load_estimator.rs` 依赖 `crate::kv_router`（`ACTIVE_SEQUENCES_SUBJECT`、`KvScheduler`）以及 `dynamo_kv_router::protocols` 的 `ActiveSequenceEvent` / `ActiveSequenceEventData`，并通过 `dynamo_runtime` 的 `Component`、`EventSubscriber` 订阅事件平面。
- `routing` 全程依赖 `dynamo_kv_router::protocols::WorkerWithDpRank`（worker + data-parallel rank 的标识）。
- `source.rs` 依赖 `object_store`（AWS S3）、`url`、`tokio` 异步 IO；`cache.rs` 依赖 `dynamo_runtime::config::environment_names::llm` 读取 `DYN_LORA_PATH`。
- 对外通过 `lora.rs` 的 `pub use` 暴露；LoRA 在集群中的注册由 `local_model` 的 `attach`（传入 `LoraInfo`）配合完成。

## 阅读建议

1. 先读 `lora.rs` 看清对外导出了哪些类型，建立整体地图。
2. 下载线：`downloader.rs` 的 `LoRADownloader::download_if_needed` →（命中走 `cache.rs`，未命中走 `source.rs` 的 `LoRASource::download`）。
3. 路由线：`routing/mod.rs` 的 `LoraAllocator` trait → `routing/hrw.rs` 的 `RendezvousHasher::compute_replica_set` → 结果存入 `routing/table.rs` 的 `LoraRoutingTable`。
4. 最后看 `load_estimator.rs` 的 `LoadEstimator`，理解负载数据如何为分配决策提供输入。
