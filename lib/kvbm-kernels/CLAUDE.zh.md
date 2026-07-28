# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在本仓库中工作时提供指引。

## 这是什么

kvbm-kernels 是一个高性能 CUDA 传输库，提供 Dynamo KV 缓存（KV cache）系统所使用的批量 H2D、D2H 与 D2D 块拷贝。核心 API（`vectorized_copy`、`memcpy_batch`）始终可用，处理在主机与设备之间移动 KV 缓存块且不改变布局这一常见情形。用于在 **Block Stack**（vLLM）与 **Universal**（Dynamo storage）格式之间进行布局转换的融合 permute-and-copy kernel，则通过 `permute_kernels` feature 进行门控。

## 构建命令

```bash
# 默认构建（自动检测 nvcc -> 源码构建；无 nvcc -> stubs）
cargo build

# 使用自定义 GPU 架构从源码构建
CUDA_ARCHS="80,86,89,90,100" cargo build

# 静态链接（将 kernel 嵌入二进制而非 .so）
cargo build --features static-kernels

# 仅检查编译，不进行链接
cargo check

# 运行核心传输 API 的 CUDA 集成测试（需要 GPU + nvcc）
cargo test --features testing-cuda

# 运行包含 permute kernel 在内的所有 CUDA 集成测试
cargo test --features testing-cuda,permute_kernels

# 运行某个具体测试
cargo test --features testing-cuda,permute_kernels fused_copy_roundtrip -- --nocapture --test-threads=1

# 运行基准测试（Llama 3.1 70B KV 缓存配置）
cargo run --example kvbench --features kvbench
```

**环境变量**：`CUDA_ARCHS`（逗号分隔的 SM 版本）、`CUDA_PTX_ARCHS`（PTX 目标）、`KVBM_REQUIRE_CUDA`（缺少 nvcc 时报错）、`CUDA_PATH`/`CUDA_HOME`。

## 架构

### 双层构建系统（`build.rs`）

构建脚本会从两种模式中择一：**FromSource**（存在 nvcc，编译 CUDA，要求 CUDA >= 12.0）或 **Stubs**（无 nvcc，使用调用即 abort 的 C stub）。Stubs 会设置 `stub_kernels` cfg 标志，便于测试有条件地跳过。

### 核心传输 API（始终可用）

下列 API 位于 `src/tensor_kernels.rs`，可作用于任何设备可见的内存（设备分配，或通过统一寻址访问的 pinned 主机内存）：

- **`vectorized_copy`** —— 对 `(src, dst)` 指针对的批量拷贝。每对会在运行时进行对齐检测，选择最宽的安全向量宽度（int4/int2/int/char，对应 16/8/4/1 字节加载）。
- **`memcpy_batch`** —— 接收 src/dst 指针的 HOST 数组。在 CUDA 12.9+ 上分发到 `cudaMemcpyBatchAsync`，并回退到逐次 `cudaMemcpyAsync` 循环。三种模式：`BatchedWithFallback`、`FallbackOnly`、`BatchWithoutFallback`。
- **`is_using_stubs`** / **`is_memcpy_batch_available`** —— 运行时能力查询。

### Permute kernels（feature 门控：`permute_kernels`）

这些 kernel 将布局 permute 与拷贝融合，用于非标准的传输路径：

- **`universal_from_block`** / **`block_from_universal`** —— 在 block stack 布局（`nl*no` 个分别分配的块，每块为 `[nt, nh, hd]` NHD 或 `[nh, nt, hd]` HND）与 universal 布局（连续的 `[nh, nl, no, nt, hd]`）之间做 permute。

### 源码组织

- `cuda/tensor_kernels.cu` —— 全部 CUDA kernel。基于 dtype（F16/BF16/F32/F64）与布局（NHD/HND）的 C++ 模板，通过前缀为 `kvbm_kernels_launch_*` / `kvbm_kernels_memcpy_batch` 的 `extern "C"` 函数对外暴露。
- `cuda/stubs.c` —— 所有 `extern "C"` 符号的 abort-on-call 回退实现。
- `src/tensor_kernels.rs` —— Rust FFI 包装、枚举（`TensorDataType`、`BlockLayout`、`MemcpyBatchMode`）以及集成测试。
- `examples/kvbench.rs` —— 基准测试外壳（Llama 3.1 70B 配置，CSV 输出）。
- `scripts/plot_roofline.py` —— 基于 kvbench 输出绘制 roofline 带宽图。

### 维度约定

`nl` = 层数，`no` = 外层块数（2：K 与 V），`nh` = attention head 数，`nt` = 每块 token 数，`hd` = head 维度。

### 指针约定

所有指针列表参数（如 `universal_ptrs`、`src_ptrs`）必须是设备可访问的：通过 `cudaMalloc`（设备内存）或 `cudaMallocHost` / `cuMemHostRegister`（pinned/registered/page-locked 主机内存）分配。

### Cargo features

| Feature | 用途 |
|---------|---------|
| `permute_kernels` | 启用融合 permute-and-copy kernel（block<->universal） |
| `testing-cuda` | 启用 CUDA 集成测试 |
| `static-kernels` | 链接为 `.a` 而非 `.so` |
| `kvbench` | 启用基准测试示例（引入 `clap`） |

### 测试组织

- `tests/stub_build.rs` —— 验证 stub 行为（在 `stub_kernels` 下生效）。
- `tests/memcpy_batch.rs` —— 核心传输 API 的回环测试（通过 pinned 主机内存做 H2D + D2H）。在 `testing-cuda` 下生效。
- `tests/kernel_roundtrip.rs` —— 跨所有 dtype 与布局的 permute kernel 回环测试。在 `testing-cuda` + `permute_kernels` 下生效。
- `src/tensor_kernels.rs` 中的内联测试 —— 包括 `universal_roundtrip` 在内的集成测试。在 `testing-cuda` + `permute_kernels` 下生效。
