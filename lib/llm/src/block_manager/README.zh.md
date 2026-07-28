# KV Cache 块管理器（block_manager）

> 管理 LLM 注意力机制里 KV Cache 的“块”：在 GPU / CPU / 本地 NVMe / 远端存储这几级存储之间分配、复用、迁移这些块。

## 这个模块解决什么问题

大模型推理时，每个 token 的 Key/Value 张量（KV Cache）必须缓存下来供后续 token 复用。显存有限，长上下文 + 高并发会很快把显存撑爆。本模块把 KV Cache 切成固定大小的“块（block）”，按内容做哈希去重，并在多级存储（显存 → 内存 → 本地盘 → 远端）之间自动 offload（下沉）/ onboard（上调），从而在有限显存下支撑更长上下文和更高的 prefix 复用率。它是 dynamo 里实现 KVBM（KV Block Manager）与跨 worker KV 传输（配合 NIXL）的核心。

入口类型是同名父文件 `block_manager.rs` 中的 `KvBlockManager<Locality, Metadata>`。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `../block_manager.rs` | 模块入口（父文件）。定义对外门面 `KvBlockManager`、`CacheLevel`（G1=GPU / G2=CPU / G3=本地 NVMe / G4=远端）、`WorkerID`，以及 `disk()/host()/device()/onboard_blocks()` 等 API |
| `config.rs` | 构建配置：`KvBlockManagerConfig`、`NixlOptions`、各级 layout 配置（builder 模式） |
| `state.rs` | 内部真实状态 `KvBlockManagerState`，区分 `local` / `logical` 两种局部性实现；门面是它的轻量包装 |
| `block.rs` + `block/` | 块本身的数据结构与生命周期：`MutableBlock`/`ImmutableBlock`、`BlockMetadata`、locality（Local/Logical）、registry、NIXL 远程块描述 |
| `pool.rs` + `pool/` | 块池 `BlockPool` / `ManagedBlockPool`：负责分配、按 sequence hash 复用、注册块 |
| `layout.rs` + `layout/` | 块在某块 `Storage` 里的内存布局（层数、page 大小、stride、对齐）；含 NIXL 布局 |
| `storage.rs` + `storage/` | 统一存储抽象 `Storage` trait 及各实现：`DeviceStorage`/`PinnedStorage`/`DiskStorage`，并支持向 NIXL 注册 |
| `offload.rs` + `offload/` | offload 管理器：在各 `CacheLevel` 之间用优先级队列调度块的下沉/上调，含 CUDA / Disk 传输管理器 |
| `events.rs` | `EventManager`：块注册（Store）/移除（Remove）时向 dynamo 事件平面发布消息，供 KV router 索引 |
| `distributed.rs` + `distributed/` | 跨 worker 分布式协作：`KvbmLeader` / `KvbmWorker`、块传输、ZMQ、NCCL bootstrap（见其英文 `README.md`） |
| `connector.rs` + `connector/` | 面向推理框架的高层 connector 接口，便于对接外部 scheduler |
| `controller.rs` + `controller/` | 接入 dynamo runtime 的 client/handler，把块管理能力暴露为可远程调用的 endpoint |
| `kv_consolidator/` | KV 事件的发布/订阅/聚合（publisher/subscriber/tracker） |
| `metrics_kvbm.rs` | KVBM 的 Prometheus 指标（缓存命中率、各方向 offload/onboard 块数等） |
| `numa_allocator.rs` | NUMA 感知分配的开关与工具（默认关闭，靠环境变量 opt-in） |
| `v2.rs` + `v2/` | 新一代实现（kernels / memory / physical），逐步迁移中 |

## 核心概念（Java 视角）

- `KvBlockManager<Locality, Metadata>`：对外门面，可类比一个线程安全的 `Service` 对象。内部用 `Arc<KvBlockManagerState>` 持有真实状态——`Arc` 类似线程安全的共享引用（多处持有、引用计数归零才释放），`Clone` 仅复制指针。它带两个泛型参数（类似 Java 泛型），`Locality` 决定块在本地还是逻辑分布式。
- `CacheLevel`（enum：G1/G2/G3/G4）：带语义的存储层级枚举，类似 Java 的 `enum`，分别对应 GPU / CPU / 本地盘 / 远端。
- `MutableBlock` vs `ImmutableBlock`：可写块与只读块。块有状态机（Empty → Partial → Complete → Registered，见 `block_manager.md`）。注册完成后变为不可变，类似“先 builder 可变、build() 后变 immutable”的模式。
- `Storage` trait：统一不同内存类型的接口，类比 Java `interface`，由 `DeviceStorage`/`PinnedStorage`/`DiskStorage` 等实现。
- `Result<T>` + `?`：几乎所有构造/分配方法返回 `Result`，`?` 相当于“出错就自动 throw”，可类比受检异常的自动向上抛出。
- `async fn new(...)`：构造是异步的，`.await` 类比 `CompletableFuture.get()` / 虚拟线程的阻塞点。offload 内部用优先级队列调度，类似 `PriorityBlockingQueue`。

## 与其他模块的关系

- 复用 `crate::common::dtype::DType` 表示张量元素类型，复用 `crate::tokens` 的 `SequenceHash` 做块去重键。
- 依赖外部 `nixl_sys`（NIXL agent）做跨节点零拷贝/RDMA 传输，依赖 `dynamo_runtime` 接入 runtime、配置与指标。
- `events.rs` 向 dynamo 事件平面发消息，被 KV router（`kv_router`）用来构建前缀缓存索引以做亲和性路由。
- 被 KVBM 相关 binding 与上层引擎使用；`controller` 把它暴露为可远程调用的服务。

## 阅读建议

1. 先读父文件 `../block_manager.rs`：看 `KvBlockManager` 门面、`CacheLevel`、`new()` 与 `disk()/host()/device()/onboard_blocks()`，建立整体心智模型。
2. 再读 `block.rs` 顶部与 `../block_manager.md` 的块状态机图，理解一个块从 Empty 到 Registered 的生命周期。
3. 然后看 `pool.rs`（如何分配/按 hash 复用）与 `offload.rs` 顶部文档（offload/onboard 的优先级调度）。
4. 想了解跨 worker 协作再进 `distributed/`（已有英文 `README.md`）。
