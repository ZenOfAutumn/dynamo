# kvbm-engine

`kvbm-engine` 为 KV Block Management（KVBM）提供分布式协调原语。
它实现了一个分层存储模型，KV cache（KV 缓存）块在 GPU 内存、host
DRAM、本地磁盘和对象存储之间流动。该 crate 协调 leader（拥有块元数据并做出放置决策）与 worker（通过 RDMA、NVMe 或对象存储 API 执行数据传输）。

## 存储层级模型

| 层级 | 介质 | 时延 | 容量 | 描述 |
|------|--------|---------|----------|-------------|
| G1 | GPU HBM | ~ns | 最小 | attention kernel 使用的活跃 KV 缓存 |
| G2 | Pinned DRAM | ~us | 中等 | RDMA 传输与层级提升的暂存区 |
| G3 | NVMe/SSD | ~ms | 大 | 持久化的温块存储 |
| G4 | S3/MinIO | ~100ms | 无限 | 冷/归档对象存储 |

## 架构

```text
                    +-----------------+
                    | InstanceLeader  |
                    |  (find_matches, |
                    |   BlockAccessor)|
                    +--------+--------+
                             |
               +-------------+-------------+
               |                           |
      +--------v--------+        +--------v--------+
      | CoordinatedWorker|       | CoordinatedWorker|
      |   (rank 0)       |       |   (rank 1)       |
      +--------+---------+       +--------+---------+
               |                           |
      +--------v--------+        +--------v--------+
      | PhysicalWorker   |       | PhysicalWorker   |
      | (TransferManager)|       | (TransferManager)|
      +-----------------+        +-----------------+
```

leader 通过 `ParallelWorkers` trait（`SpmdParallelWorkers`
用于 SPMD 执行）驱动 worker。对于 onboarding，leader 创建 session 并经历多个阶段：search、hold、prepare（G3->G2）以及 pull（远程 G2->本地 G2，通过 RDMA）。

## 模块

| 模块 | 用途 |
|--------|---------|
| `leader` | 块协调：匹配、onboarding session、基于策略的扫描 |
| `worker` | 传输执行：本地、RDMA 与对象存储数据迁移 |
| `object` | G4 存储：用于冷层块持久化的 S3/MinIO 客户端 |
| `offload` | 层级降级管道：批量 G2->G3 与 G2->G4 offload |
| `runtime` | 共享基础设施：`KvbmRuntime`、tokio handle、NIXL agent |
| `pubsub` | 事件 pub/sub：用于跨实例协调的块级通知 |
| `collectives` | 多 GPU 同步的 NCCL 集合通信（feature 门控） |
| `testing` | 测试工具：mock worker、内存块管理器（feature 门控） |

## Feature 标志

| Flag | 依赖 | 描述 |
|------|-------------|-------------|
| `default` | `["s3"]` | 默认 feature |
| `s3` | `aws-sdk-s3`、`aws-config`、`rayon`、`tokio-rayon`、`chrono` | S3/MinIO 对象存储支持 |
| `collectives` | `nixl-sys`、`nccl` | NIXL + NCCL 多 GPU 集合通信 |
| `nccl` | `cudarc` | 通过 cudarc 提供 NCCL 支持 |
| `testing-nccl` | `collectives` | 为测试启用 collectives |
| `nats` | `async-nats`、`flume` | 基于 NATS 的 pub/sub 传输 |
| `testing` | `kvbm-logical/testing`、`kvbm-physical/testing` | 测试工具与 mock 基础设施 |
| `nvtx` | `kvbm-config/nvtx` | NVIDIA Tools Extension 性能剖析标记 |

## 快速开始

```rust,ignore
use kvbm_engine::{KvbmRuntime, leader::InstanceLeader};

// 从环境构建 runtime
let runtime = KvbmRuntime::from_env_leader().await?;

// 创建 leader 实例
let leader = InstanceLeader::new(/* ... */);

// 搜索缓存的块
let result = leader.find_matches(&sequence_hashes)?;
```
