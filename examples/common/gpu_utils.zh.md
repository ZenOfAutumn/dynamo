# GPU 内存控制

vLLM、SGLang 与 TensorRT-LLM 如何分配 GPU 内存，以及我们如何对其进行覆盖以
实现可重现的并行测试执行。

---

## 为什么使用绝对上限而非分数

内存分数（`--gpu-memory-utilization`、`--mem-fraction-static`）在并行/CI 工作负载下
不可靠：

- **不确定性** —— 同样的分数在初始化时根据 GPU 上其他占用情况会产生不同的 KV 缓存
  大小。
- **剖析竞争（profiling race）** —— 多个并发引擎都会看到"几乎全部空闲内存"，并据此
  分配，进而 OOM。
- **可移植性差** —— 为 48 GiB 调好的分数到 24 或 80 GiB 上就不再合适。
- **语义不同** —— vLLM/SGLang 使用 *总* VRAM 的分数；TensorRT-LLM 使用模型加载后
  *剩余* VRAM 的分数。

我们改为使用 **绝对 KV 缓存上限**：

| 引擎 | 确定性覆盖 | 环境变量 |
|--------|----------------------|---------|
| vLLM | `--kv-cache-memory-bytes N` | `_PROFILE_OVERRIDE_VLLM_KV_CACHE_BYTES` |
| SGLang | `--max-total-tokens N` | `_PROFILE_OVERRIDE_SGLANG_MAX_TOTAL_TOKENS` |
| TensorRT-LLM | `--override-engine-args '{"kv_cache_config":{"max_tokens":N}}'` | `_PROFILE_OVERRIDE_TRTLLM_MAX_TOTAL_TOKENS` |

---

## 速查

| | vLLM | SGLang | TensorRT-LLM |
|---|---|---|---|
| 分数参数 | `--gpu-memory-utilization` | `--mem-fraction-static` | `free_gpu_memory_fraction` |
| 分数基准 | 总 VRAM | 总 VRAM | 加载后剩余 VRAM |
| 默认值 | 0.90 | 0.90 | 0.90 |
| 最大序列长度 | `--max-model-len` | `--context-length` | `max_seq_len` |
| KV 缓存覆盖 | `--kv-cache-memory-bytes` | `--max-total-tokens` | 通过 `--override-engine-args` 设置 `KvCacheConfig.max_tokens` |

---

## 各引擎说明

### vLLM

`--gpu-memory-utilization` 将预算设为总 VRAM 的分数。
KV 缓存 = 预算 - 权重 - 激活 - 开销。该池在启动时固定。

`--kv-cache-memory-bytes` 覆盖自动计算并 **跳过内存剖析**（[PR #21489]）。KV 缓存
被锁定为精确字节数 —— 没有剖析竞争，没有 CUDAGraph 估算误差，可安全用于并发实例
（[#10643]）。设置后，`--gpu-memory-utilization` 仅影响激活的余量，不影响 KV
缓存大小。

`--max-model-len` 限定序列长度。当模型可放下而 KV 缓存放不下时，减小该值是
削减显存最快的方式。

[PR #21489]: https://github.com/vllm-project/vllm/pull/21489
[#10643]: https://github.com/vllm-project/vllm/issues/10643

### SGLang

`--mem-fraction-static` 将预算设为总 VRAM 的分数。
KV 缓存池 = 预算 - 权重。激活与 CUDA graph 缓冲区在该预算 *之外*（与 vLLM 不同）。

`--max-total-tokens` 直接限定 KV token 池，与分数无关。设置后，token 上限即为
绑定约束。

`--context-length` 与 `--max-running-requests` 仅影响请求调度——它们 **不会**
改变 KV 缓存的分配。

### TensorRT-LLM

`free_gpu_memory_fraction` 是模型加载后 **剩余** VRAM 的分数。
通过 YAML 或 `--override-engine-args '{"kv_cache_config":{"free_gpu_memory_fraction": 0.24}}'` 设置。

确定性的 KV 缓存控制使用 `gpu_utils.sh` 中的 `build_trtllm_override_args_with_mem`
函数构建用于 `--override-engine-args` 的 JSON。支持基于 token
（`_PROFILE_OVERRIDE_TRTLLM_MAX_TOTAL_TOKENS`）或字节
（`_PROFILE_OVERRIDE_TRTLLM_MAX_GPU_TOTAL_BYTES`）的上限。如果启动脚本已经
传递了 `--override-engine-args`，该函数会通过 `--merge-with-json` 将 GPU 配置
合并到现有 JSON 中。

---

## 各引擎专用的 GPU 内存函数

启动脚本会 source `gpu_utils.sh`，并调用各引擎专用函数，在剖析与并行执行期间
应用环境变量覆盖：

```bash
source "$SCRIPT_DIR/../../../common/gpu_utils.sh"

# vLLM
GPU_MEM_ARGS=$(build_vllm_gpu_mem_args)
python -m dynamo.vllm --model "$MODEL" $GPU_MEM_ARGS &

# SGLang
GPU_MEM_ARGS=$(build_sglang_gpu_mem_args)
python -m dynamo.sglang --model-path "$MODEL" $GPU_MEM_ARGS &

# TRT-LLM (JSON merging, separate function)
OVERRIDE_JSON=$(build_trtllm_override_args_with_mem)
python -m dynamo.trtllm --model-path "$MODEL" ${OVERRIDE_JSON:+--override-engine-args "$OVERRIDE_JSON"} &
```

当环境变量被设置时，函数返回相应参数；否则返回空，引擎使用默认分配。

| 环境变量 | 函数 | 输出 |
|---------|----------|--------|
| `_PROFILE_OVERRIDE_VLLM_KV_CACHE_BYTES` | `build_vllm_gpu_mem_args` | `--kv-cache-memory-bytes N --gpu-memory-utilization 0.01` |
| `_PROFILE_OVERRIDE_SGLANG_MAX_TOTAL_TOKENS` | `build_sglang_gpu_mem_args` | `--max-total-tokens N` |
| `_PROFILE_OVERRIDE_TRTLLM_MAX_TOTAL_TOKENS` | `build_trtllm_override_args_with_mem` | `{"kv_cache_config": {"max_tokens": N}}` (JSON) |
| `_PROFILE_OVERRIDE_TRTLLM_MAX_GPU_TOTAL_BYTES` | `build_trtllm_override_args_with_mem` | `{"kv_cache_config": {"max_gpu_total_bytes": N}}` (JSON) |

所有函数返回的都是单进程参数。在每 GPU 多 worker 的部署（如 `disagg_same_gpu.sh`）
中，每个 worker 拿到相同的覆盖值。Profiler 直接寻找单 worker 的预算。

**Profiler**（`profile_pytest.py`）：对 KV 上限做二分搜索，找到能通过的最小值，
再施加 2x 安全系数，输出 pytest marker
（`@pytest.mark.requested_vllm_kv_cache_bytes(N)`、
`@pytest.mark.requested_sglang_kv_tokens(N)` 或
`@pytest.mark.requested_trtllm_kv_tokens(N)`）。

**调度器**（`pytest_parallel_gpu.py`）：在运行时读取这些 marker，并按测试设置
对应的环境变量。详见 `tests/README.md`。
