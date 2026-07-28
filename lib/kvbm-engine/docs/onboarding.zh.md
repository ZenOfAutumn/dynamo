# 上手指南

欢迎来到 `kvbm-engine`。本文档将带你了解本 crate 的核心抽象，帮助你快速建立全局认识并开始贡献。

`kvbm-engine` 是 KV 缓存（KV cache）块管理（KVBM）的分布式协调层。它位于 `kvbm-physical`（负责搬运字节）与 `kvbm-logical`（负责跟踪块元数据）之上，将两者编织为一套系统：**leader** 对块做出决策，**worker** 在分层存储体系上执行数据传输：

```text
G1 (GPU HBM)  →  G2 (Pinned DRAM)  →  G3 (NVMe/SSD)  →  G4 (S3/MinIO)
```

核心张力在于**逻辑（logical）**与**物理（physical）**之间。Leader 思考序列哈希与块身份 — 不会触碰原始内存。Worker 思考 layout 句柄、传输管理器与 DMA 描述符 — 不会做放置决策。Engine 把两者粘合在一起。

---

## Worker

Worker 是逻辑-物理二元中的物理一侧。核心实现是 `PhysicalWorker`，它是 `kvbm-physical` 之上的薄协调封装。

`PhysicalWorker` 拥有：

- 一个 **`TransferManager`** — 真正通过 NIXL（RDMA/UCX）、NVMe 或对象存储 API 在内存区之间搬运数据的 `kvbm-physical` 引擎。
- 至多三层的 **layout 句柄**（`g1_handle`、`g2_handle`、`g3_handle`）— 这些是物理内存区注册，传输管理器借此知道数据在本进程中的*位置*。
- 一份 **远端句柄** 映射 — 从对端 worker 导入的物理句柄，可启用 RDMA 拉取。

Worker 实现两个 trait：

**`WorkerTransfers`** 定义传输操作：
- `execute_local_transfer(src, dst, block_ids, ...)` — 在本 worker 内跨层搬块（如 G2 → G1）。
- `execute_remote_onboard(remote_desc, dst, block_ids, ...)` — 通过 RDMA 从远端 worker 拉到本地 layout。
- `execute_remote_offload(src, remote_desc, block_ids, ...)` — 将本地数据推送到远端描述符。
- `connect_remote(instance_id, metadata)` — 导入对端的 NIXL 元数据以便和它互相 RDMA。

**`Worker`** 在 `WorkerTransfers` 之上扩展了 layout 句柄访问与用于 RDMA 设置的元数据导入/导出。

所有传输操作返回 `TransferCompleteNotification` — 你 await 的异步句柄，用于得知数据搬运完成。这是系统在传输调度与执行之间实现重叠的方式。

---

## Worker 即远端服务（Velo）

在多进程部署中，每个 worker 运行在独立进程。我们不直接调 `PhysicalWorker` 的方法，而是把它封装为 Velo RPC 服务。

**`VeloWorkerService`** 接收一个 `PhysicalWorker`，并为每个 `WorkerTransfers` 与 `Worker` 方法（如 `kvbm.worker.local_transfer`、`kvbm.worker.remote_onboard` 等）注册处理器。该服务运行在 worker 进程中。

**`VeloWorkerClient`** 实现相同的 `Worker` trait，但每次调用都被序列化为 Velo 消息发送给远端服务，并返回由完成事件支撑的 `TransferCompleteNotification`。

关键洞见：**从 leader 视角看，本地与远端 worker 可互换。**两者都实现 `Worker`。Leader 永远不知道（也不关心）自己面对的是同进程的 `PhysicalWorker`，还是跨进程的 `VeloWorkerClient`。

```text
Leader process                          Worker process
┌───────────────────┐                   ┌───────────────────┐
│  InstanceLeader   │                   │                   │
│        │          │                   │                   │
│  CoordinatedWorker│                   │                   │
│        │          │                   │                   │
│  VeloWorkerClient │ ── Velo RPC ──▶  │ VeloWorkerService │
│                   │                   │        │          │
│                   │                   │  PhysicalWorker   │
│                   │                   │  (TransferManager)│
└───────────────────┘                   └───────────────────┘
```

还有一层封装：**`CoordinatedWorker`**。它运行在 leader 进程，在 `Worker`（本地或远端）之上叠加协调状态，跟踪 leader 视角下哪些 layout 句柄对应哪些远端实例与 rank。当 leader 说"从 Instance B 的 rank 0 拉块"时，`CoordinatedWorker` 解析出正确的物理句柄并委托给内部 `Worker`。

---

## Worker 组（Worker Groups）

Worker 可以组织成组，向 leader 暴露统一的"单 worker"接口。`ParallelWorkers` trait 是 `Worker` 在组层级的对应物。

### 张量并行（SPMD）

`SpmdParallelWorkers` 是默认组实现。它将每个操作并行广播到所有 N 个 worker — 即 SPMD（Single Program, Multiple Data）模式。

在典型张量并行部署中，每个 GPU 持有每个 KV 缓存块自己的分片（shard）。当 leader 说"将块 [1, 2, 3] 从 G2 传到 G1"时，SPMD 组会扇出（fan out）到每个 rank。每个 rank 在自己的分片上执行同一传输，结果聚合后再返回给 leader。

```text
Leader: "transfer blocks 1,2,3 from G2 → G1"
         │
   SpmdParallelWorkers
         │
    ┌────┼────┐
    ▼    ▼    ▼
  Rank0 Rank1 Rank2    (each transfers its own shard)
```

### 复制数据（MLA）

对于多头潜在注意力（MLA），KV 数据是复制（replicated）的，而非分片。`ReplicatedDataWorker`（由 `collectives` 特性门控）实现了不同策略：

- **Rank 0** 是唯一拥有 G2 与 G3 存储的 worker。所有跨层传输（G3 → G2 → G1）由它执行。
- **Rank 1..N** 仅有 G1。它们通过 NCCL `broadcast` 从 rank 0 接收数据。

这意味着 leader 仍可统一说"上线这些块"，组内部处理这种非对称 — rank 0 干重活，然后向其他人广播。

### 抽象之力

这两种策略 — 对称分片与复制广播 — 在物理层面非常不同，但 leader 都通过相同的 `ParallelWorkers` / `WorkerTransfers` 接口驱动。这正是 worker group 的核心价值：**在统一 API 之下承载不同的并行策略**。

抽象目前并不完整 — 更多并行模式将需要更多组实现 — 但已足以支撑给出的两种用例，并示范了扩展的模式。

---

## Leader

Leader 是 worker 在逻辑侧的对应物。`InstanceLeader` 拥有所有块数据的逻辑视图，无论它们在 worker 与各层之间如何物理分布。

`InstanceLeader` 持有：

- 一个用于去重的 **`BlockRegistry`** — 跟踪已见过的序列哈希。
- 一个 **`BlockManager<G2>`**（必需）与可选的 **`BlockManager<G3>`** — 主机 DRAM 与磁盘的逻辑块仓库。
- **worker** 列表（通过 `CoordinatedWorker`），以及可选的 **`SpmdParallelWorkers`** 组。
- 用于分布式 onboarding 的 **session** 映射（详见下文）。
- 可选的 **远端 leader** 引用，用于跨实例协调。

### find_matches

核心入口是 `find_matches(sequence_hashes)`。给定一组序列哈希，leader 决定哪些块已存在以及位于何处：

1. 在本地 G2 `BlockManager` 中搜索匹配。
2. 对剩余哈希在本地 G3 `BlockManager` 中搜索。
3. （可选）通过分布式 session 在远端 leader 上搜索。

结果或为：
- **`Ready`** — 全部所需块都在本地 G2 中找到；调用方立即获得 RAII `BlockHolder`。
- **`AsyncSession`** — 部分块需要 staging（G3 → G2）或远端传输；调用方拿到带状态 watch 通道的 session 句柄。

### BlockHolder（RAII 所有权）

`BlockHolder<T>`（其中 T 为 `G2` 或 `G3`）是 session 期间持有块的 RAII 守卫。持有期间，这些块不可被驱逐。Holder 被 drop 时块释放。即使 session 处理 panic，也不会泄漏。

### 块扫描（Block Scanning）

`InstanceLeader` 还提供 `scan_with_policy` — 一种灵活的迭代机制：调用方传入一个闭包，使用 `BlockAccessor`（同时包装 G2 与 G3 manager）搜索块，并通过 `PolicyContext` 输出结果。这能在不暴露 block manager 内部的前提下实现自定义扫描策略（连续段、按 LFU 排序的扫描）。

---

## 实例（Instances）

**Instance** 是部署单元：一个 leader 加上它的若干 worker。

```text
┌─ Instance (TP=2) ──────────────────────────┐
│                                             │
│   InstanceLeader                            │
│       │                                     │
│   SpmdParallelWorkers                       │
│       ├── Worker (rank 0, GPU 0)            │
│       └── Worker (rank 1, GPU 1)            │
│                                             │
└─────────────────────────────────────────────┘
```

单 GPU 部署中，instance 即一个 leader 加一个 worker。
张量并行下，则是一个 leader 驱动一个 SPMD 组。

Leader 决策；worker 执行。Leader 不碰字节；worker 不做放置决策。

---

## 传输分类

按作用域，传输分为三类：

### Local（worker 内、instance 内）

单个 worker 内的层间传输：G1 ↔ G2、G2 ↔ G3 等。

这是张量并行部署的家常便饭。每个 worker 独立地在层间搬运自己的分片。SPMD 组将同一逻辑操作广播到所有 rank，每个 rank 在自己物理 layout 上执行。

### Intra（worker 间、instance 内）

同一 instance 内 worker 之间的传输。代表性场景是 MLA/复制数据模式：rank 0 完成 G3 → G2 → G1，再 NCCL 广播 G1 数据到其他 rank。数据跨 worker 边界，但仍在同一 instance 内。

### Inter（worker 间、instance 间）

不同 instance 上 worker 之间的传输。这就是**分布式 KVBM** — 下一节讲述的对等（peer-to-peer）模型。

```text
           ┌──────────────────────────────┐
           │          Local               │
           │    (intra-worker, intra-inst) │
           │     G2 ←→ G1 on Rank 0      │
           └──────────────────────────────┘

  ┌──────────────────────────────────────────────┐
  │              Intra                            │
  │       (inter-worker, intra-inst)              │
  │   Rank 0 ──NCCL bcast──▶ Rank 1..N           │
  └──────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │                       Inter                                  │
  │              (inter-worker, inter-inst)                       │
  │   Instance A, Rank 0 ──RDMA──▶ Instance B, Rank 0           │
  └──────────────────────────────────────────────────────────────┘
```

---

## 分布式 KVBM（跨实例传输）

分布式 KVBM 是一种对等模型，两个或更多 instance 通过 **session** 协调块所有权，再触发直接的 worker-to-worker 传输。

### Session

Session 是两个 instance 之间短暂的协调协议。有两种角色：

- **`InitiatorSession`** — 请求方（如需要块的 Prefill instance）。
- **`ResponderSession`** — 提供方（如已缓存了块的 Decode instance）。

Session 通过状态机推进：

```text
Searching ──▶ Holding ──▶ Staging ──▶ Ready ──▶ Complete
                                                  │
                                             (or Failed)
```

- **Searching**：initiator 请求 responder 搜索其本地 block manager。
- **Holding**：responder 已找到块，并通过 `BlockHolder` 持有以防被驱逐。
- **Staging**：在 responder 上 G3 → G2 提升正在进行（若块在磁盘上）。准备 NIXL 描述符以便 RDMA。
- **Ready**：块在 responder 的 G2 中并已可 RDMA 访问。
- **Complete**：initiator 已拉完所有块。session 拆除。

### 实例：TP=2 的跨实例传输

设 Instance A（Prefill，TP=2）希望从 Instance B（Decode，TP=2）拿到序列哈希为 `[hash_1, hash_2]` 的 KV 块。

```text
Instance A (Prefill, TP=2)              Instance B (Decode, TP=2)
┌──────────────────────┐                ┌──────────────────────┐
│ Leader A             │                │ Leader B             │
│  ├─ Worker A0 (GPU0) │                │  ├─ Worker B0 (GPU0) │
│  └─ Worker A1 (GPU1) │                │  └─ Worker B1 (GPU1) │
└──────────────────────┘                └──────────────────────┘
```

流程：

1. **Leader A 与 Leader B 创建 session**，将其要找的序列哈希 `[hash_1, hash_2]` 发送过去。

2. **Leader B 接收请求**（`ResponderSession`）。它在 G2/G3 block manager 中搜索匹配。

3. **Leader B 通过 `BlockHolder` 获取所有权**，避免传输期间被驱逐。

4. **Leader B 响应** 找到的内容：哪些哈希命中、所在层级、以及允许 RDMA 访问 G2 块的 NIXL 描述符。

5. **Leader A 命令其 worker 拉取。** 由于两端都是 TP=2，映射是 1:1 — A 的 rank 0 从 B 的 rank 0 拉，A 的 rank 1 从 B 的 rank 1 拉。每次拉取都是用 NIXL 在 worker 进程间直接的 RDMA 传输。

6. **Session 完成。** Leader B 释放 `BlockHolder`。Leader A 现在已在自己的 G2 中拥有这些块。

rank 映射由 `LeaderState` 中的 `route_local_to_remote` 处理，也支持非对称配置（如 TP=4 拉自 TP=2）。

### 传输

Session 消息走 **Velo**（项目的 RPC 框架）。`VeloLeaderService` 注册 `kvbm.leader.onboard`、`kvbm.leader.remote_session`、`kvbm.leader.session` 处理器 — 它们将到来的消息派发到相应的每会话通道。

测试场景下，`LocalTransport` 提供进程内直连派发，无网络开销。

---

## 对象（Objects） vs 块（Blocks）

整个 crate 中你将看到 KV 缓存数据的两种不同表示：

### Block（块）

**Block** 是 G1–G3 层中的基本单位。由 `BlockId` 标识，与 `SequenceHash` 关联，由 `BlockManager` 管理。块有物理后端（GPU HBM、pinned DRAM 或 NVMe），并支持通过 NIXL 直接传输内存。`BlockManager` 处理分配、驱逐与频次跟踪。块是热路径、低延迟的表示。

### Object（对象）

**Object** 是 G4（S3/MinIO）的表示。对象通过**键**寻址（由 `KeyFormatter` 从 `SequenceHash` 派生），而非 `BlockId`。`ObjectBlockOps` trait 定义接口：`has_blocks`、`put_blocks`、`get_blocks`。

对象之所以存在，是因为 S3 不支持下层那种基于句柄的、面向块的访问模式。它们以更高延迟与键值访问模型为代价，提供了无限容量的冷存储。

对于 SPMD 部署，`RankPrefixedKeyFormatter` 会在每个对象键前加上 worker rank（`{rank}/{hash}`），从而每个 worker 的分片独立存储。

`ObjectLockManager` 通过条件 S3 PUT 提供分布式锁，防止并发 instance 间出现重复上传。

---

## 后续阅读

掌握了概念模型后，可继续阅读各模块的实现细节：

| 文档 | 涵盖内容 |
|----------|--------|
| [architecture.md](architecture.md) | 层级模型、模块图、特性开关、快速开始 |
| [leader.md](leader.md) | `Leader` trait、`InstanceLeader`、`FindMatchesResult`、staging 模式 |
| [worker.md](worker.md) | `Worker` / `WorkerTransfers`、`PhysicalWorker`、`CoordinatedWorker`、Velo 层 |
| [worker-group.md](worker-group.md) | `SpmdParallelWorkers`、扇出、rank 感知路由 |
| [session.md](session.md) | session 协议、initiator/responder/可控制（controllable）、消息类型、状态机 |
| [offload.md](offload.md) | offload 流水线阶段、策略、取消 |
| [object.md](object.md) | G4 存储、S3 客户端、锁管理器 |
| [runtime.md](runtime.md) | `KvbmRuntime` 构建与共享基础设施 |
| [testing.md](testing.md) | 测试工具、多 instance fixtures、RDMA 传输测试 |

运行测试套件：

```bash
cargo test -p kvbm-engine --features testing
```
