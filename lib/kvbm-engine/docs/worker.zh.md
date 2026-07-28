# Worker 模块

Worker 模块定义了在存储层级之间执行数据传输的执行原语。Worker 拥有
执行物理资源（传输管理器、布局句柄），通过 RDMA、本地拷贝或对象存储
来移动 block。

## Trait 层级

```text
WorkerTransfers          Worker
  (执行)              (元数据 + 句柄)
       │                    │
       └────────┬───────────┘
                │
         ObjectBlockOps
          (G4 存储)
```

- **`WorkerTransfers`** —— 核心执行 trait。提供 `execute_local_transfer`、
  `execute_remote_onboard`、`execute_remote_offload`、`connect_remote`
  以及 `execute_remote_onboard_for_instance`。
- **`Worker`** —— 扩展 `WorkerTransfers + ObjectBlockOps`。新增布局
  句柄访问器（`g1_handle`、`g2_handle`、`g3_handle`）和元数据导入/导出。

## PhysicalWorker（亦称 DirectWorker）

`PhysicalWorker` 是基础的单 worker 实现。它直接拥有一个 `TransferManager`
和用于执行数据移动的布局句柄。

### Builder

```rust,ignore
let worker = PhysicalWorker::builder()
    .manager(transfer_manager)   // 必填
    .g1_handle(g1)               // 可选 —— GPU 层
    .g2_handle(g2)               // 可选 —— host 层
    .g3_handle(g3)               // 可选 —— 磁盘层
    .rank(0)                     // 可选 —— 用于 SPMD key 前缀
    .object_client(s3_client)    // 可选 —— 用于 G4 操作
    .build()?;
```

| 字段 | 必填 | 用途 |
|-------|----------|---------|
| `manager` | 是 | 用于执行传输的 `TransferManager` |
| `g1_handle` | 否 | GPU/HBM 布局句柄 |
| `g2_handle` | 否 | Host/pinned-DRAM 布局句柄 |
| `g3_handle` | 否 | 磁盘/NVMe 布局句柄 |
| `rank` | 否 | SPMD key 前缀使用的 worker rank |
| `object_client` | 否 | G4 对象存储客户端 |

`DirectWorker` 是 `PhysicalWorker` 的兼容别名。

### 执行状态 vs 协调状态

PhysicalWorker 维护**执行状态** —— 真正执行传输所需的句柄和管理器。
这与 leader 在 `CoordinatedWorker` 中跟踪的**协调状态**有所区别。
当 leader 用 CoordinatedWorker 包装 PhysicalWorker 时，句柄会有意
存在于两处：PhysicalWorker 需要它们调用 TransferManager，而
CoordinatedWorker 提供本地与远程 worker 的统一 API。

## CoordinatedWorker

`CoordinatedWorker` 是 leader 视角下的 worker。它包装任何 `Worker`
实现，并增加协调状态：

- 本地布局句柄（通过 `apply_layout_response` 填充）
- 用于跨 leader RDMA 传输的远程句柄映射
- worker rank 和宿主 instance 跟踪

该包装让 leader 无论底层 worker 是本地（`PhysicalWorker`）还是
远程（`VeloWorkerClient`）都能使用相同的 API。

## VeloWorkerClient / VeloWorkerService

Velo（RPC）层支持远程 worker 执行：

- **`VeloWorkerService`** —— 包装一个 `PhysicalWorker`，对外暴露
  `execute_local_transfer`、`export_metadata`、`import_metadata` 等
  RPC 处理器。
- **`VeloWorkerClient`** —— 通过向远程 `VeloWorkerService` 发送 RPC
  请求来实现 `WorkerTransfers`。

二者结合让 leader 能像驱动本地 worker 那样驱动远程节点上的 worker。
