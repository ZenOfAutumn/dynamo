# CUDA 内核（kernels）

> 一份用 CUDA C++ 写的 KV block 拷贝内核，通过 FFI 被 Rust 侧的 `kv` 模块调用。

## 这个模块解决什么问题

LLM 推理的 KV cache 以「block」为单位散落在 GPU 显存 / pinned 主存里。当需要把若干个 block 从一处搬到另一处（比如 prefill 完成后做 offload、或 host↔device 之间倒腾）时，逐个 `cudaMemcpy` 太慢。本目录用一个手写 CUDA kernel 一次性、按维度感知（prefix/block/suffix 三维布局）地批量拷贝多对 block，并提供一个有状态的 `CopyStream` 复用 CUDA stream 与 pinned 缓冲，降低反复分配与同步的开销。

这里**只有 C++ 内核实现**，没有 `.rs` 文件；它是 `lib/llm/src/kv` 模块的底层依赖。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `block_copy.cu` | 全部实现：CUDA kernel `copy_blocks_kernel` + 一组 `extern "C"` 启动函数 + `CopyStream` 类 + 主存分配/拷贝辅助函数 |

`block_copy.cu` 内对外暴露（`extern "C"`，供 Rust FFI 链接）的主要符号：

| C 符号 | 作用 |
|---|---|
| `copy_blocks_3d` / `copy_blocks_launcher_3d` / `copy_blocks_memcpy_3d` | 一次性拷贝多对 block（按 prefix/block/suffix 三维 stride 寻址），分别走 kernel 与 `cudaMemcpyAsync` 两条路径 |
| `copy_stream_create` / `copy_stream_destroy` | 创建/销毁有状态的 `CopyStream`（持有 CUDA stream 与预分配的 block-id 缓冲） |
| `copy_stream_prepare_block_ids` | 把源/目标 block 索引上传到设备，供后续 launch 复用 |
| `copy_stream_launch` / `copy_stream_memcpy` / `copy_stream_scatter` | 在已准备好的 stream 上发起拷贝（kernel / memcpy / 散射拷贝三种模式） |
| `copy_stream_sync` | 等待该 stream 上的拷贝完成 |
| `cuda_malloc_host` / `cuda_free_host` / `cuda_memcpy_async` / `cuda_memcpy_sync` | pinned 主存分配与异步/同步拷贝的薄封装 |

## 核心概念（Java 视角）

- `extern "C" cudaError_t copy_blocks_3d(...)`：可以理解为 Java 里用 JNI（`native` 方法）暴露给上层的 C 函数签名。返回的 `cudaError_t` 类似一个错误码，Rust 侧拿到后会翻成 `Result`。
- `__global__ void copy_blocks_kernel(...)`：GPU 上由成千上万线程并行执行的函数体，没有 Java 直接对应物——可类比「把一个 for 循环体下发给一个超大线程池里的每个线程各跑一份」，每个线程靠 `blockIdx`/`threadIdx` 算出自己负责哪一段数据。
- `class CopyStream`：一个有状态、可复用的拷贝会话对象，内部持有 CUDA stream 和设备端 block-id 缓冲。类比 Java 里一个持有底层连接/缓冲、需要显式 `close()` 释放的资源对象（这里由 Rust 侧的 `Drop` 负责调用 `copy_stream_destroy`）。

## 与其他模块的关系

- **被谁用**：`lib/llm/src/kv/layer.rs` 通过 `extern "C" { fn copy_blocks_3d(...); fn copy_stream_create(...); ... }` 声明并调用这些符号，再用 Rust 类型 `CopyStream` / `CopyStreamContext` / `CopyStreamBlockMap` 做安全封装。
- **如何编译进来**：见 `lib/llm/build.rs`。注意 build.rs 中直接编译本 `.cu` 的逻辑目前大多是注释状态；实际链接的拷贝 kernel 走的是 `src/block_manager/block/transfer/kernels/vectorized_copy.fatbin`（由 `make fatbin` 产出）。因此本目录更像是 `kv` 模块对应的内核源码，与 `block_manager` 那套 transfer kernel 是两条独立路径。
- **不依赖任何 Rust crate**：纯 C++/CUDA，只依赖 CUDA Runtime。

## 阅读建议

1. 先看 `block_copy.cu` 文件末尾的 `extern "C"` 区块（约 `copy_stream_create` 到 `copy_stream_scatter`），它定义了对 Rust 暴露的全部接口。
2. 再回到 `copy_blocks_kernel`（文件开头）理解三维 stride 寻址与 cache-line 分块的拷贝逻辑。
3. 最后对照 `lib/llm/src/kv/layer.rs` 顶部的 `extern "C"` 声明，看 Rust 侧是怎么把这些函数包成 `CopyStream` 安全类型的。
