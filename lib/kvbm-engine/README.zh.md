# kvbm-engine

用于 KV 缓存（KV cache）块管理（KVBM）的分布式协调原语（primitives）。

本 crate 实现了在分层存储体系中跨层管理 KV 缓存块的 leader/worker 架构：

**G1**（GPU HBM）→ **G2**（Pinned DRAM）→ **G3**（NVMe/SSD）→ **G4**（S3/MinIO）

Leader 持有块元数据并做出放置决策。Worker 执行数据传输（RDMA、NVMe、对象存储）。Session 协调多实例之间的块传输。

## 特性开关（Feature Flags）


| 标志           | 用途                                       |
| -------------- | ------------------------------------------ |
| `s3`（默认）   | S3/MinIO 对象存储（G4 层）                 |
| `testing`      | 测试工具与 mock 基础设施                   |
| `nats`         | 基于 NATS 的发布/订阅传输                  |
| `collectives`  | NIXL + NCCL 多 GPU 集合通信                |
| `nccl`         | 通过 cudarc 使用 NCCL                      |
| `nvtx`         | NVIDIA Tools Extension 性能分析标记        |


## 文档

详细的模块文档位于 `[docs/](docs/)`：

- [Architecture](docs/architecture.md) — 整体系统设计
- [Leader](docs/leader.md) — 块协调与元数据管理
- [Session](docs/session.md) — 分布式上线（onboarding）协议
- [Worker](docs/worker.md) — 传输执行
- [Worker Group](docs/worker-group.md) — SPMD 并行 worker
- [Offload](docs/offload.md) — 异步层级降级（tier-demotion）流水线
- [Offload Developer Guide](docs/offload-developer.md) — 为 offload 模块贡献代码
- [Object Storage](docs/object.md) — S3/MinIO 集成
- [Runtime](docs/runtime.md) — 运行时打包（tokio、Velo、NIXL）
- [Testing](docs/testing.md) — 测试工具与 fixtures
