## Dynamo KV Block Manager Kernels

用于在 LLM 推理框架使用的三种内存布局之间转换 KV 缓存（KV cache）block 的 GPU kernel。所有转换均通过融合 CUDA kernel 完全在设备端运行。

### 维度

| 符号 | 含义                          | 示例             |
|--------|--------------------------------|------------------|
| `nb`   | 批次中的 block 数              | 1–128            |
| `nl`   | 层数                            | 32（Llama-70B）   |
| `no`   | 外层分块（K 与 V）              | 2                |
| `nh`   | 注意力头数                       | 32 或 64         |
| `nt`   | 每个 block 的 token 数          | 128 或 256       |
| `hd`   | 头维度                          | 128              |

### 布局

#### Block Stack（NHD 或 HND）

每个 block 包含 `nl * no` 个独立的 GPU 分配。每个分配存放某一层的 K 或 V。

- **NHD 形状**：`[nt, nh, hd]` —— 索引：`(nt_idx * nh + nh_idx) * hd + hd_idx`
- **HND 形状**：`[nh, nt, hd]` —— 索引：`(nh_idx * nt + nt_idx) * hd + hd_idx`

以长度为 `nb * nl * no` 的扁平指针表传入 kernel。

#### Operational

每个 block 一段连续缓冲：`[nl, no, inner]`，其中 `inner = nt * nh * hd`。

最内的三个维度（`nt`、`nh`、`hd`）被融合为单一的 `inner` 维度。当不需要布局置换时（相同 TP 配置、相同的头布局），block 到 operational 是一次扁平拷贝——开销最低的转换。与其他布局之间的转换需要知晓其各组成维度。

#### Universal

每个 block 一段连续缓冲：`[nh, nl, no, nt, hd]`。

将 head 作为最外层维度，使得张量并行（tensor-parallelism）重切片只是沿 `nh` 的连续切片。从 TP=4 部署中保存的 block 可通过对 head 维度的不同切分加载到 TP=8 中。

### 布局速查表

| 布局              | 逻辑形状                    | 存储为                              | 备注                          |
|---------------------|----------------------------|------------------------------------|-------------------------------|
| NHD block stack     | `[nl][no][nt, nh, hd]`     | `nl * no` 个指针组成的列表          | 内部布局 = NHD                |
| HND block stack     | `[nl][no][nh, nt, hd]`     | `nl * no` 个指针组成的列表          | 内部布局 = HND                |
| Operational block   | `[nl, no, inner]`          | 每个 block 一段连续缓冲             | `inner = nt * nh * hd`        |
| Universal block     | `[nh, nl, no, nt, hd]`     | 每个 block 一段连续缓冲             | head 在最外层，便于 TP 切片   |

### Kernel 函数

所有 kernel 都是批量化的：单次 launch 处理由主机代码准备好的扁平指针表中的 `nb` 个 block。

#### 布局置换 kernel

| C API                                        | 转换                          |
|----------------------------------------------|-----------------------------|
| `kvbm_kernels_launch_universal_from_block`   | Block stack → Universal     |
| `kvbm_kernels_launch_block_from_universal`   | Universal → Block stack     |

二者都接受 `layout_value`（NHD=0, HND=1）和 `dtype_value`（F16=0, BF16=1, F32=2, F64=3）。内部分发到按 dtype 与 layout 模板特化的 C++ kernel。

#### 独立的拷贝工具

| C API                                    | 描述                                                       |
|------------------------------------------|----------------------------------------------------------|
| `kvbm_kernels_launch_vectorized_copy`    | 跨 `num_pairs` 指针对的自适应向量化拷贝（16/8/4 字节或标量） |
| `kvbm_kernels_memcpy_batch`              | 从主机指针数组发起的批量 `cudaMemcpyAsync`                 |
| `kvbm_kernels_has_memcpy_batch_async`    | 若可用 `cudaMemcpyBatchAsync` 则返回 `true`                |
| `kvbm_kernels_is_stub_build`             | 若构建时未启用 CUDA（stub 模式）则返回 `true`              |


### Python 绑定（计划中）

Python kernel 绑定尚未实现。`lib/bindings/kvbm/` crate 当前仅暴露 block manager 功能。未来将为置换与拷贝 kernel 添加 Python 包装。

### 开发

```bash
# Default build (auto-detects nvcc → source; no nvcc → stubs)
cargo build

# Custom GPU architectures
CUDA_ARCHS="80,86,89,90,100" cargo build

# Static linking
cargo build --features static-kernels

# Run CUDA integration tests (requires GPU + nvcc)
cargo test --features testing-cuda,permute_kernels

# Specific test with output
cargo test --features testing-cuda,permute_kernels fused_copy_roundtrip -- --nocapture

# Python bindings
cd lib/bindings/kvbm
uv pip install -e ".[dev]"
pytest tests/
```

**环境变量**：`CUDA_ARCHS`（逗号分隔的 SM 版本，默认 `80,86,89,90,100,120`）、`CUDA_PATH`/`CUDA_HOME`（toolkit 根目录）、`KVBM_REQUIRE_CUDA`（缺少 nvcc 时构建失败）。

### 基准测试

```text
root@9eb240f7ded8:/workspace/lib/kvbm-kernels# cargo run --release --example kvbench --features testing-cuda,kvbench -- --num-blocks=1,128 --tokens-per-block=16,64 --
backend vectorized,batched --direction h2d
...
     Running `/workspace/target/release/examples/kvbench --num-blocks=1,128 --tokens-per-block=16,64 --backend vectorized,batched --direction h2d`
KV Cache Transfer Benchmark
  Model: Llama 3.1 70B (bf16)
  Layers: 80, KV heads: 8, Head dim: 128, Outer dim: 2
  Warmup: 10, Timed: 100
  Batch API available: true
  tokens_per_block: [16, 64]
  num_blocks: [1, 128]
  directions: [h2d]
  patterns: [fc_to_fc, lw_to_fc]
  backends: [vectorized, batched]
  Total tests: 16

tokens_per_block,num_blocks,pattern,direction,backend,total_bytes,inner_bytes,copy_size,num_copies,median_ms,bandwidth_gbps
--- tokens_per_block=16, inner=32768 bytes (32 KB), block=5242880 bytes (5.0 MB) ---
  [1/16] tpb=16 N=  1 fc_to_fc h2d    vectorized   ... 16,1,fc_to_fc,h2d,vectorized,5242880,32768,5242880,1,1.8686,2.81
2.81 GB/s (1.8686 ms)
  [2/16] tpb=16 N=  1 fc_to_fc h2d    batched      ... 16,1,fc_to_fc,h2d,batched,5242880,32768,5242880,1,0.2105,24.91
24.91 GB/s (0.2105 ms)
  [3/16] tpb=16 N=  1 lw_to_fc h2d    vectorized   ... 16,1,lw_to_fc,h2d,vectorized,5242880,32768,32768,160,0.2171,24.15
24.15 GB/s (0.2171 ms)
  [4/16] tpb=16 N=  1 lw_to_fc h2d    batched      ... 16,1,lw_to_fc,h2d,batched,5242880,32768,32768,160,0.2775,18.89
18.89 GB/s (0.2775 ms)
  [5/16] tpb=16 N=128 fc_to_fc h2d    vectorized   ... 16,128,fc_to_fc,h2d,vectorized,671088640,32768,5242880,128,26.6097,25.22
25.22 GB/s (26.6097 ms)
  [6/16] tpb=16 N=128 fc_to_fc h2d    batched      ... 16,128,fc_to_fc,h2d,batched,671088640,32768,5242880,128,26.6180,25.21
25.21 GB/s (26.6180 ms)
  [7/16] tpb=16 N=128 lw_to_fc h2d    vectorized   ... 16,128,lw_to_fc,h2d,vectorized,671088640,32768,32768,20480,26.6034,25.23
25.23 GB/s (26.6034 ms)
  [8/16] tpb=16 N=128 lw_to_fc h2d    batched      ... 16,128,lw_to_fc,h2d,batched,671088640,32768,32768,20480,30.3346,22.12
22.12 GB/s (30.3346 ms)
--- tokens_per_block=64, inner=131072 bytes (128 KB), block=20971520 bytes (20.0 MB) ---
  [9/16] tpb=64 N=  1 fc_to_fc h2d    vectorized   ... 64,1,fc_to_fc,h2d,vectorized,20971520,131072,20971520,1,7.5837,2.77
2.77 GB/s (7.5837 ms)
  [10/16] tpb=64 N=  1 fc_to_fc h2d    batched      ... 64,1,fc_to_fc,h2d,batched,20971520,131072,20971520,1,0.8334,25.16
25.16 GB/s (0.8334 ms)
  [11/16] tpb=64 N=  1 lw_to_fc h2d    vectorized   ... 64,1,lw_to_fc,h2d,vectorized,20971520,131072,131072,160,0.8407,24.95
24.95 GB/s (0.8407 ms)
  [12/16] tpb=64 N=  1 lw_to_fc h2d    batched      ... 64,1,lw_to_fc,h2d,batched,20971520,131072,131072,160,0.9020,23.25
23.25 GB/s (0.9020 ms)
  [13/16] tpb=64 N=128 fc_to_fc h2d    vectorized   ... 64,128,fc_to_fc,h2d,vectorized,2684354560,131072,20971520,128,106.3677,25.24
25.24 GB/s (106.3677 ms)
  [14/16] tpb=64 N=128 fc_to_fc h2d    batched      ... 64,128,fc_to_fc,h2d,batched,2684354560,131072,20971520,128,106.3199,25.25
25.25 GB/s (106.3199 ms)
  [15/16] tpb=64 N=128 lw_to_fc h2d    vectorized   ... 64,128,lw_to_fc,h2d,vectorized,2684354560,131072,131072,20480,106.3158,25.25
25.25 GB/s (106.3158 ms)
  [16/16] tpb=64 N=128 lw_to_fc h2d    batched      ... 64,128,lw_to_fc,h2d,batched,2684354560,131072,131072,20480,110.0665,24.39
24.39 GB/s (110.0665 ms)

Done.
```

### 故障排查

| 现象                                  | 可能原因 / 修复                                                       |
|---------------------------------------|--------------------------------------------------------------------|
| 启动时报 `cudaErrorInvalidValue`      | 指针数量不匹配（`nb`、`nl`、`no`）或输入非连续                     |
| 使用 HND 布局时数值错误               | 内部张量在传入前未整形为 `[nh, nt, hd]`                            |
| Python 绑定抱怨 dtype                 | 批次中存在混合精度；将张量转换为统一 dtype                          |
| Kernel 用时异常                       | 检查 `CUDA_ARCHS` 与你的 GPU 是否匹配，避免运行时 JIT             |
