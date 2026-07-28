# KV Block 管理（kv）

> 一套以「block」为单位管理 KV cache 显存的底层原语：复用池、在途引用、底层存储与拷贝。

## 这个模块解决什么问题

LLM 推理时，每个 token 的 Key/Value 张量按固定大小的 block 缓存在 GPU/主存里。同一段前缀（prefix）在不同请求间可以**复用**已经算好的 block，省去重复 prefill。本模块提供这套机制的 Rust 原语：用 `sequence_hash` 标识 block 内容、维护一个「空闲但保留状态」的复用池（`AvailableBlocks`）、跟踪「正在被请求使用」的在途 block（`ReservedBlocks`），并封装底层显存 slab 与跨 device/host 的拷贝（依赖 `kernels/block_copy.cu`）。

> 注意：本模块当前**未**在 `lib/llm/src/lib.rs` 中用 `pub mod kv;` 声明，crate 内也没有 `crate::kv::` 的引用。它是一套较早期/实验性的 KV 管理原语（`sequence.rs` 为空、部分测试被注释），阅读时应把它当成「设计参考与原语集合」，而非线上主路径。生产路径上的 block 管理见 `lib/llm/src/block_manager`。

## 目录结构

| 文件 | 职责 |
|---|---|
| `kv.rs` | 模块入口（`mod kv`）。定义核心类型 `KvBlock`（一个 block：`TokenBlock` + 优先级 + 回收时刻），以及类型别名 `UniqueBlock = PoolItem<KvBlock>`、`SharedBlock = SharedPoolItem<KvBlock>` |
| `reuse.rs` | `AvailableBlocks`：空闲 block 复用池。按优先级 FIFO 返还、可按 `sequence_hash` 做状态匹配命中、支持 fence 同步。内部用 mpsc channel + 后台 task 串行处理请求 |
| `reserved.rs` | `ReservedBlocks` / `ReservedBlock`：在途 block 表。用 `Weak` 引用 + `Drop` 自动从表里摘除，实现「最后一个使用者释放即归还」 |
| `manager.rs` | `KvStorageManager`：把上面两者组合起来，提供 `prepare_prefill_sequence` / `prepare_prefill_offload`——先匹配在途、再匹配空闲池、剩下的才真正新建 |
| `storage.rs` | `StorageType`（`Device`/`Pinned`/`System`）、`DType`、`Storage`/`OwnedStorage`：对显存/pinned/系统内存大块 slab 的非安全封装 |
| `layer.rs` | `KvLayer` / `KvLayout` / `CopyStream` 等：按模型层布局组织 block，并通过 FFI 调用 `kernels/block_copy.cu` 做批量拷贝 |
| `sequence.rs` | 目前仅有版权头，无内容（占位） |

## 核心概念（Java 视角）

- `struct KvBlock`：一个 KV cache block 的元数据（持有 `TokenBlock`、`priority`、`return_tick`）。类比一个普通 POJO，但它的生命周期由对象池托管。
- `UniqueBlock` / `SharedBlock`（`PoolItem<KvBlock>` / `SharedPoolItem<KvBlock>`）：从对象池借出的句柄。`Unique` 类似独占借用、`Drop` 时自动归还池子；`Shared` 类似引用计数共享（`Arc`）。比 Java 强的地方是「归还池」这件事编译期就靠所有权保证，不用手动 `release()`。
- `AvailableBlocks`（`reuse.rs`）：典型的 actor 模式——对外是若干 `async fn`（`match_blocks`/`take_blocks`/`insert`/`fence`），内部把请求塞进 `mpsc` channel（≈ `BlockingQueue`）交给一个后台 task 串行处理，从而避免显式加锁。每个 `async fn` 都返回 `Result<...>`（≈ 受检异常），`?` 即「自动 throw」。
- `ReservedBlocks`（`reserved.rs`）：用 `Arc<RwLock<HashMap<SequenceHash, Weak<...>>>>`（≈ 线程安全的共享 Map，值是弱引用）登记在途 block。`ReservedBlockInner` 的 `Drop` impl 在最后一个强引用消失时把条目从表中删除——相当于 Java 里把「引用计数归零时的回调」直接写进了析构逻辑。
- `enum StorageType`（`storage.rs`）：`Device(Arc<CudaContext>)` / `Pinned` / `System` 三选一的带数据枚举，类比 Java 的 sealed class，模式匹配时编译器强制你处理每种情况。

## 与其他模块的关系

- **依赖**：`crate::tokens`（`TokenBlock`/`SequenceHash`/`Tokens`）、`dynamo_runtime::utils::pool`（对象池原语 `PoolItem`/`Returnable` 等）、`cudarc`（CUDA 驱动绑定）；`layer.rs` 通过 FFI 依赖同 crate 的 `kernels/block_copy.cu`。
- **未被 crate 内引用**：如上所述，`kv` 没有挂到 `lib.rs`。线上的 KV block 管理在 `lib/llm/src/block_manager`（功能更完整，含分布式/NIXL transfer）。可把本模块视作其设计前身或独立原语库。

## 阅读建议

1. 从 `kv.rs` 的 `KvBlock` 与两个类型别名 `UniqueBlock` / `SharedBlock` 入手，建立「block 是池中对象」的心智模型。
2. 看 `manager.rs::KvStorageManager::prepare_prefill_sequence`——它把整个匹配流程串起来（在途 → 空闲池 → 新建），是理解本模块协作方式的最佳切入点。
3. 再分别下钻 `reuse.rs::AvailableBlocks`（复用池的 actor 实现）和 `reserved.rs::ReservedBlocks`（在途表 + Drop 归还）。
4. 需要看底层显存与拷贝时，再读 `storage.rs` 的 `StorageType`/`Storage` 和 `layer.rs` 顶部的 `extern "C"` FFI 声明（对应 `kernels/README.zh.md`）。
