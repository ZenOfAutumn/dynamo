# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在本仓库中处理代码时提供指导。

## 构建与测试

这是 dynamo 工作区中的一个 Rust crate（`kvbm-engine`）。Rust edition 2024，需要 rustc 1.93.1+。

```bash
# 构建
cargo build -p kvbm-engine
cargo build -p kvbm-engine --features s3,testing,nats

# 测试（大多数测试需要 `testing` feature）
cargo test -p kvbm-engine --features testing
cargo test -p kvbm-engine --features testing -- test_name  # 单个测试

# Lint
cargo clippy -p kvbm-engine --all-features
cargo fmt
cargo machete
```

## Feature 标志

| Flag | 用途 |
|------|---------|
| `s3`（默认） | S3/MinIO 对象存储（G4 层级） |
| `testing` | 测试工具、mock 基础设施、fixture |
| `nats` | 基于 NATS 的 pub/sub 传输 |
| `collectives` | NIXL + NCCL 多 GPU 集合通信 |
| `nccl` | 通过 cudarc 使用 NCCL |
| `nvtx` | NVIDIA Tools Extension 性能剖析标记 |

## 架构

kvbm-engine 在分层存储层级中为 KV cache（KV 缓存）块管理实现分布式协调：

- **G1**（GPU HBM）→ **G2**（pinned DRAM）→ **G3**（NVMe/SSD）→ **G4**（S3/MinIO）

Leader 拥有块元数据并做出放置决策。Worker 执行数据传输（RDMA、NVMe、对象存储）。Session 在 leader 与 worker 之间协调多实例块传输。

### 关键模块

- **`leader/`** — `InstanceLeader` 协调块查找（`find_matches`），通过 RAII `BlockHolder` 持有块，并管理分布式 session。`Leader` trait 是核心协调接口。
- **`leader/session/`** — 分布式 session 协议：`InitiatorSession`（请求方）、`ResponderSession`（提供方）、`ServerSession`（服务端块暴露，可选 G3→G2 暂存）。Session 跟踪 onboarding 状态：Searching → Holding → Staging → Ready → Complete。
- **`worker/`** — `PhysicalWorker` 拥有 `TransferManager` 和实际传输的 layout handle。`CoordinatedWorker` 用 leader 的协调状态包装任意 `Worker`。`Worker` 与 `WorkerTransfers` trait 定义执行契约。
- **`worker/group/`** — `SpmdParallelWorkers` 以 SPMD 模型并行向所有 worker 广播操作并聚合事件。
- **`worker/velo/`** — RPC 层（`VeloWorkerService`/`VeloWorkerClient`），通过 Velo 进行远程 worker 执行。
- **`offload/`** — 多阶段异步管道用于层级降级：PolicyEvaluator → PreconditionAwaiter → Batcher → TransferExecutor。支持每容器的取消 token。**修改本模块前请先阅读 `src/offload/AGENTS.md` 了解治理规则。**
- **`object/`** — `ObjectBlockOps` trait 用于 G4 存储。S3 实现支持并发上传/下载。`ObjectLockManager` 通过条件式 S3 PUT 实现分布式锁。
- **`runtime/`** — `KvbmRuntime` 打包 tokio、Velo messenger、NixlAgent（RDMA）和 EventManager。通过 `KvbmRuntimeBuilder` 或便捷构造器（`from_env_leader`、`from_env_worker`）构建。
- **`pubsub/`** — Publisher/Subscriber trait，提供 NATS 与内存 stub 实现。
- **`collectives/`** — `CollectiveOps` trait 用于多 GPU 同步。包含 NCCL 实现与测试用 stub。MLA 模式：仅 rank 0 需要 G2/G3，其他 rank 通过广播接收。
- **`testing/`** — feature 门控的测试工具：`TestManagerBuilder`、`MessengerPair`、`TestSession`、`EventsPipelineFixture`、`MultiInstancePopulator`、`TestAgent`。

### 文档

模块文档位于 `docs/`，通过 `#[doc = include_str!("../docs/...")]` 引入。修改某个模块时，请同步更新对应的文档文件。

### 关键模式

- **基于 trait 的抽象**：`Leader`、`Worker`、`WorkerTransfers`、`ObjectBlockOps`、`CollectiveOps`、`KeyFormatter`——实现可替换（真实实现 vs 测试 stub）。
- **RAII 资源管理**：`BlockHolder` 在 session 期间持有块，drop 时自动释放。`TransferHandle` 跟踪 offload 操作。
- **Builder 模式**：`InstanceLeaderBuilder`、`PhysicalWorkerBuilder`、`KvbmRuntimeBuilder`、`OffloadEngineBuilder`。
- **执行 vs 协调状态**：`PhysicalWorker` 拥有执行状态；`CoordinatedWorker` 增加 leader 的协调视图。无论 worker 位于本地还是远程，API 一致。

### 工作区依赖

内部 crate：`kvbm-common`、`kvbm-config`、`kvbm-kernels`、`kvbm-logical`、`kvbm-physical`、`velo`、`dynamo-tokens`、`dynamo-memory`。

## Offload 模块治理

offload 模块（`src/offload/`）有显式策略（P1–P6），记录在其 README 中。修改 offload 代码前，请阅读 `src/offload/AGENTS.md` 与 offload 文档（`docs/offload.md`、`docs/offload-developer.md`）。脱离策略的变更需在实现前获得用户批准。
