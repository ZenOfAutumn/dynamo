# Dynamo 测试规范

本文档介绍如何在 Dynamo 项目中组织、标记并运行测试。请遵循这些规范，以保证整套测试套件的一致性与可维护性。

Dynamo 的测试与检查可以分为三大类：

1. **[Rust 测试](#rust-testing)** —— 覆盖 `lib/` 下的 Rust crate，包括单元测试与集成测试。CI 在合入之前还会强制执行格式、lint、license 检查。
2. **[Python 测试（pytest）](#python-testing-pytest)** —— 覆盖 Python 组件以及跨组件的工作流，包含单元、集成与端到端测试。通过 pytest marker 按生命周期阶段、硬件、框架等维度选择测试。
3. **其他检查** —— 格式（`cargo fmt`、`ruff`）、lint（`clippy`、`pre-commit`）、license（`cargo-deny`）、未使用依赖（`cargo machete`）、文档构建（`cargo doc`）。它们是 CI 的一部分，详见 [运行 Rust 检查与测试](#running-rust-checks-and-tests)。

所有测试都在容器中运行。请参见 [Container Development Guide](../container/README.md) 了解如何构建并启动开发容器。

每一类测试又可包含以下几种：

1. **单元测试** —— 隔离地验证单个函数、类或模块。不依赖外部服务，也不需要 GPU。每个用例通常以毫秒计；整体单元测试通常 <5 分钟。
2. **集成测试** —— 通过 **mock 引擎**（`dynamo.mocker`）和 **真实基础设施**（用于服务发现的 ETCD、可选的 NATS 消息总线）把多个组件串起来。验证 router、planner、frontend gRPC 等子系统之间的协作，但不会启动真实的推理引擎，也不需要 GPU。每个用例通常以秒计；整体集成测试通常 <30 分钟。
3. **端到端（E2E）测试** —— 启动真实的推理引擎（vLLM、SGLang 或 TRT-LLM），通过 frontend 发送请求并验证响应。需要 GPU。每个用例通常以分钟计；整套 E2E 套件可能耗时数小时。

写测试时务必关注用时，慢测试的成本会被复合放大：它会消耗 CI 中昂贵的共享 GPU 时长；会让工程师不愿在本地运行，从而把 bug 留到 CI 才暴露；还会拖慢整个团队的开发节奏。一个跑得太久的测试套件最终就成了"没人会跑"的套件。在新增或修改测试时，请在 PR 描述中给出该测试的预估时间——CI 的 GPU 资源有限，这些估时有助于团队把测试合理地编排到 pre-merge、nightly、weekly 各个流水线中。

本文中的耗时数字是大致值，基于一台 32 核机器、2026 年 Q1 的测量结果。具体耗时会随硬件和代码规模变化。

---

## 测试组织：测试代码放在哪里

### 目录结构
```
dynamo/
├── lib/
│   ├── runtime/
│   │   ├── src/
│   │   │   └── lib.rs              # Rust code + unit tests inside
│   │   └── tests/                  # Rust integration tests for runtime
│   ├── llm/
│   │   ├── src/
│   │   │   └── lib.rs              # Rust code + unit tests inside
│   │   └── tests/                  # Rust integration tests for llm
│   └── ...
├── components/
│   └── src/dynamo/
│       ├── vllm/
│       │   └── tests/              # Python unit/integration tests for vllm backend
│       ├── trtllm/
│       │   └── tests/              # Python unit/integration tests for trtllm backend
│       ├── sglang/
│       │   └── tests/              # Python unit/integration tests for sglang backend
│       ├── common/
│       │   └── tests/              # Python unit/integration tests for common utils
│       ├── planner/
│       ├── router/
│       ├── frontend/
│       ├── profiler/
│       └── ...
├── tests/                          # End-to-end and cross-component tests
│   ├── serve/                      # Serve E2E tests (vllm, sglang, trtllm)
│   ├── kvbm_integration/           # KVBM integration tests
│   ├── gpu_memory_service/         # GPU Memory Service E2E tests
│   ├── fault_tolerance/            # Fault tolerance, migration, cancellation
│   ├── deploy/                     # Deployment tests
│   ├── frontend/                   # Frontend HTTP/gRPC tests
│   ├── router/                     # Router E2E tests
│   ├── mm_router/                  # Multimodal router tests
│   ├── lmcache/                    # LM cache tests
│   ├── basic/                      # Basic backend tests
│   └── utils/                      # Shared test utilities
├── benchmarks/                     # Performance/load benchmarks
│   ├── router/
│   ├── llm/
│   └── ...
```
- 把组件的**单元/集成测试**放在该组件的 `tests/` 子目录下：`components/src/dynamo/<component>/tests/`。
- 把**端到端（E2E）**及跨组件测试放在仓库根目录下的 `tests/`。
- 测试文件名建议使用 `test_<component>_<flow>.py` 这种风格，便于辨识。

### 测试类型与位置

**Rust 测试**（`cargo test`）—— 单个用例通常 100 ms 到 30 s：

| 类型              | 描述                              | 位置                                     |
|-------------------|------------------------------------------|----------------------------------------------|
| 单元              | 单个函数/类，内联测试      | `lib/<crate>/src/`（`#[cfg(test)]` 模块）  |
| 集成       | 跨模块、按 feature 开关启用              | `lib/<crate>/tests/`                         |

**Python 测试**（`pytest`）：

| 类型               | 描述                           | 位置                                      |
|--------------------|---------------------------------------|-----------------------------------------------|
| 单元               | 隔离单个函数/类       | `components/src/dynamo/<component>/tests/`    |
| 集成        | 模块/服务之间的交互 | `components/src/dynamo/<component>/tests/`    |
| 端到端         | 用户工作流、CLI、API              | `tests/serve/`、`tests/deploy/` 等         |
| KVBM 集成   | KV block manager 集成          | `tests/kvbm_integration/`                     |
| GPU Memory Service | GPU Memory Service E2E                | `tests/gpu_memory_service/`                   |
| Router             | 路由与后端的 E2E              | `tests/router/`                               |
| Planner            | Planner 的单元 + 集成测试      | `components/src/dynamo/planner/tests/`        |
| Frontend           | Frontend HTTP/gRPC 测试              | `tests/frontend/`                             |
| Profiler           | Profiler 的单元 + 集成测试     | `components/src/dynamo/profiler/tests/`       |
| Global Planner     | Global planner 单元测试             | `components/src/dynamo/global_planner/tests/` |
| 容错    | 混沌、迁移、取消        | `tests/fault_tolerance/`                      |
| 部署         | 部署校验                 | `tests/deploy/`                               |
| 基准测试          | 性能/压力                      | `benchmarks/`                                 |

---

## 测试标记：如何为测试打 marker

所有测试都必须打 marker。CI 与本地运行都依赖它们来选择测试。

### Marker 要求
- 每个测试至少要有一个 **生命周期** marker，并要有 **测试类型** 与 **硬件** marker。
- 当与 **组件/框架** 相关时，必须打上对应 marker。

### Marker 表
| 类别                | Marker                                                        | 说明                        |
|-------------------------|------------------------------------------------------------------|------------------------------------|
| 生命周期 [必填]    | pre_merge, post_merge, nightly                                   | 测试应当在哪个阶段运行。各流水线的总预算：pre_merge < 30 分钟，post_merge < 1 小时，nightly < 3 小时。详见 [Pipeline Time Budgets](#pipeline-time-budgets)。 |
| 测试类型 [必填]    | unit, integration, e2e, benchmark, performance, stress, multimodal | 测试性质                  |
| 硬件 [必填]     | gpu_0, gpu_1, gpu_2, gpu_4, gpu_8, h100                         | 所需 GPU 数量/类型       |
| VRAM（已 profile）         | profiled_vram_gib(N)                                                         | profile 期间 nvidia-smi 实测的峰值显存（已包含 CUDA 开销）。用于 `--max-vram-gib=N` 过滤以及 GPU 并行调度的预算计算。 |
| vLLM KV cache 字节     | requested_vllm_kv_cache_bytes(N)                                             | （仅 vLLM）KV cache 的精确字节数。设置 `_PROFILE_OVERRIDE_VLLM_KV_CACHE_BYTES` → `--kv-cache-memory-bytes`。确定性、并发安全。 |
| SGLang KV tokens        | requested_sglang_kv_tokens(N)                                                          | （仅 SGLang）KV cache 最大 token 数。设置 `_PROFILE_OVERRIDE_SGLANG_MAX_TOTAL_TOKENS` → `--max-total-tokens`。确定性、并发安全。 |
| SGLang VRAM GiB         | requested_sglang_vram_gib(N)                                                           | （仅 SGLang）最大 VRAM（GiB），用于不适用 token 控制的非文本工作负载（视频/图像扩散模型）。 |
| TRT-LLM KV tokens      | requested_trtllm_kv_tokens(N)                                                          | （仅 TRT-LLM）KV cache 最大 token 数。通过 `--override-engine-args` 设置 `_PROFILE_OVERRIDE_TRTLLM_MAX_TOTAL_TOKENS` → `KvCacheConfig.max_tokens`。确定性、并发安全。 |
| TRT-LLM VRAM GiB       | requested_trtllm_vram_gib(N)                                                           | （仅 TRT-LLM）最大 VRAM（GiB）。通过 `--override-engine-args` 设置 `_PROFILE_OVERRIDE_TRTLLM_MAX_GPU_TOTAL_BYTES` → `KvCacheConfig.max_gpu_total_bytes`。用于不适用 token 控制的非文本工作负载（视频/图像扩散模型）。 |
| 组件/框架     | vllm, trtllm, sglang, kvbm, kvbm_concurrency, planner, router   | 后端或组件归属   |
| 基础设施          | k8s, deploy, fault_tolerance                                     | 基础设施/环境需求   |
| 执行               | parallel                                                         | 该测试可与 pytest-xdist 并行运行。必须使用动态端口分配（`alloc_ports`），且不与其他测试共享资源（例如文件系统）。 |
| 其他                   | slow, skip, xfail, custom_build, model, aiconfigurator           | 特殊处理                   |

### 示例（vLLM）
```python
@pytest.mark.pre_merge
@pytest.mark.integration
@pytest.mark.gpu_1
@pytest.mark.profiled_vram_gib(20.5)  # actual nvidia-smi peak
@pytest.mark.requested_vllm_kv_cache_bytes(942_054_000)  # KV cache cap (2x safety over min=471_027_000)
@pytest.mark.vllm
def test_kv_cache_behavior():
    ...
```

### 示例（SGLang，按 token 控制）
```python
@pytest.mark.pre_merge
@pytest.mark.e2e
@pytest.mark.gpu_1
@pytest.mark.profiled_vram_gib(3.7)   # actual nvidia-smi peak at recommended token count
@pytest.mark.requested_sglang_kv_tokens(96)     # KV cache cap (2x safety over min=48)
@pytest.mark.timeout(265)
@pytest.mark.sglang
def test_sglang_aggregated():
    ...
```

### 示例（TRT-LLM，按 token 控制）
```python
@pytest.mark.pre_merge
@pytest.mark.e2e
@pytest.mark.gpu_1
@pytest.mark.profiled_vram_gib(3.9)   # actual nvidia-smi peak at recommended token count
@pytest.mark.requested_trtllm_kv_tokens(2592)   # KV cache cap (2x safety over min=1296)
@pytest.mark.timeout(300)
@pytest.mark.trtllm
def test_trtllm_aggregated():
    ...
```

### 示例（TRT-LLM 扩散模型——无 KV cache）
```python
@pytest.mark.pre_merge
@pytest.mark.gpu_1
@pytest.mark.trtllm
# Diffusion models don't use KV cache, so requested_trtllm_kv_tokens doesn't apply
# and requested_trtllm_vram_gib (KvCacheConfig.max_gpu_total_bytes) has no effect —
# the VRAM is model weights + activations. Only profiled_vram_gib is meaningful.
@pytest.mark.profiled_vram_gib(17.1)  # actual nvidia-smi peak
@pytest.mark.timeout(600)
def test_trtllm_video_diffusion():
    ...
```

### VRAM marker 与过滤

不同引擎的 marker 不同：

**vLLM** 使用基于字节的 KV cache 控制：
- **`profiled_vram_gib(N)`** —— nvidia-smi 实测峰值，用于 `--max-vram-gib` 过滤与调度预算。
- **`requested_vllm_kv_cache_bytes(N)`** —— KV cache 精确字节数。设置 `_PROFILE_OVERRIDE_VLLM_KV_CACHE_BYTES` → `--kv-cache-memory-bytes`，确定性、并发安全。

**SGLang** 使用基于 token 的控制：
- **`profiled_vram_gib(N)`** —— 在推荐 token 数下 nvidia-smi 实测的峰值，用于 `--max-vram-gib` 过滤与调度预算。
- **`requested_sglang_kv_tokens(N)`** —— KV cache 最大 token 数。设置 `_PROFILE_OVERRIDE_SGLANG_MAX_TOTAL_TOKENS` → `--max-total-tokens N --mem-fraction-static 0.9`。token 上限决定真实的 KV 分配；`--mem-fraction-static 0.9` 同时被传入，是为了在小显存 GPU 上让 SGLang 的池创建门槛仍能通过（详见 PR #9238）。确定性、并发安全（详见 `examples/common/gpu_utils.md`）。
- **`requested_sglang_vram_gib(N)`** —— 用于不适用 token 控制的非文本工作负载（视频/图像扩散模型）的最大 VRAM（GiB）。仅用于调度准入与展示。

**TRT-LLM** 对文本模型采用基于 token 的控制，对扩散模型采用基于字节的控制：
- **`profiled_vram_gib(N)`** —— nvidia-smi 实测峰值，用于 `--max-vram-gib` 过滤与调度预算。
- **`requested_trtllm_kv_tokens(N)`** —— 文本模型 KV cache 的最大 token 数。通过 `--override-engine-args` 的 JSON 设置 `_PROFILE_OVERRIDE_TRTLLM_MAX_TOTAL_TOKENS` → `KvCacheConfig.max_tokens`。确定性、并发安全。
- **`requested_trtllm_vram_gib(N)`** —— 用于非文本工作负载（视频/图像扩散）的最大 VRAM（GiB）。通过 `--override-engine-args` 的 JSON 设置 `_PROFILE_OVERRIDE_TRTLLM_MAX_GPU_TOTAL_BYTES` → `KvCacheConfig.max_gpu_total_bytes`。注意：扩散模型不使用 KV cache，因此该参数可能没有任何作用——`profiled_vram_gib` 单独足以用于调度预算。
- TRT-LLM 的 `--override-engine-args` 需要 JSON 合并，由 `gpu_utils.sh` 中的 `build_trtllm_override_args_with_mem` 处理（区别于 `build_vllm_gpu_mem_args` / `build_sglang_gpu_mem_args`）。

`--max-vram-gib=N` 会取消选择 `profiled_vram_gib` 超过 N 的测试。没有打 VRAM marker 的测试也会被跳过（VRAM 未知 = 并行不安全）。要把测试加入并行池中，请先用 `tests/utils/profile_pytest.py` 进行 profile（详见 [GPU VRAM Profiler](#gpu-vram-profiler-profile_pytestpy)）。

### GPU 并行执行

GPU 测试通过自定义的、感知 VRAM 的调度器（`tests/utils/pytest_parallel_gpu.py`）并发运行。它独立于 `pytest-xdist`，原因如下：

1. **VRAM 预算**：xdist 不感知 GPU 显存——在一张 48 GiB 的 GPU 上同时跑两个 20 GiB 的测试会 OOM。
2. **profile 期间的竞态**：引擎在初始化时会快照空闲显存；并发启动会互相干扰。该调度器会错峰启动（VRAM 稳定性检查）并对偶发失败重试。
3. **引擎特定的内存分配**：每个测试都会拿到一个受限的内存预算，确保它只用自己那一份。xdist 没有这种机制。
   - **vLLM**：`_PROFILE_OVERRIDE_VLLM_KV_CACHE_BYTES = N` → `--kv-cache-memory-bytes`（来自 `requested_vllm_kv_cache_bytes` marker）。基于字节的上限是确定性的，不依赖当前可用显存，因此天然并发安全。使用 `gpu_utils.sh` 中的 `build_vllm_gpu_mem_args`。
   - **SGLang**：`_PROFILE_OVERRIDE_SGLANG_MAX_TOTAL_TOKENS = N` → `--max-total-tokens N --mem-fraction-static 0.9`（来自 `requested_sglang_kv_tokens` marker）。基于 token 的上限是确定性的，不依赖当前可用显存，因此天然并发安全；同时输出 `--mem-fraction-static 0.9` 是为了在小显存 GPU 上让 SGLang 池创建门槛仍能通过（详见 PR #9238）。非文本工作负载可使用 `requested_sglang_vram_gib`，仅用于调度准入。
   - **TRT-LLM**：`_PROFILE_OVERRIDE_TRTLLM_MAX_TOTAL_TOKENS = N` → 通过 `--override-engine-args` JSON 设置 `KvCacheConfig.max_tokens`（来自 `requested_trtllm_kv_tokens` marker）。基于 token 的上限是确定性、并发安全的。使用 `gpu_utils.sh` 中的 `build_trtllm_override_args_with_mem`（独立函数，因为 TRT-LLM 需要 JSON 合并）。

```bash
# Dry-run: preview which tests fit and the GPU plan
python3 -m pytest --max-vram-gib=24 --dry-run -m "gpu_1 and vllm" tests/serve/test_vllm.py

# Run pre-merge vllm tests in parallel
python3 -m pytest --max-vram-gib=6 -n auto -m "gpu_1 and vllm and not nightly and not post_merge" tests/serve/test_vllm.py

# Run all (pre+post merge) with live output
python3 -m pytest --max-vram-gib=48 -n auto -sv -m "gpu_1 and vllm and not nightly" tests/serve/test_vllm.py tests/frontend/test_vllm.py

# SGLang tests
python3 -m pytest --max-vram-gib=48 -n auto -m "gpu_1 and sglang" tests/serve/test_sglang.py

# Tests that still need profiling
python3 -m pytest --dry-run -m "(gpu_1 or gpu_2) and not profiled_vram_gib" tests/serve/
```

示例输出（6 个 SGLang 测试，RTX 6000 Ada 48 GiB）：
```
GPU parallel: 6 tests, 7 concurrent slots, GPU0 (48 GiB, 43 GiB multi-proc budget)

[w0] tests/serve/test_sglang.py::...completions_only-2]     profiled= 14.9 GiB  req_kv_tokens=  1024  timeout=420s
[w1] tests/serve/test_sglang.py::...multimodal_agg_qwen-2]  profiled= 20.2 GiB  req_kv_tokens=   512  timeout=280s
[w2] tests/serve/test_sglang.py::...aggregated-2]            profiled=  6.0 GiB  req_kv_tokens=  1024  timeout=240s
...

[w0] tests/serve/...completions_only-2] (GPU0, profiled 14.9 GiB, req_kv_tokens=  1024) RUNNING
[w1] tests/serve/...multimodal_agg_qwen-2] (GPU0, profiled 20.2 GiB, req_kv_tokens=   512) RUNNING
[elapsed 10s] GPU0: 0.6/48 GiB [w0(10s), w1(5s)] [queued: w2, w3, w4, w5]
[w1] tests/serve/...multimodal_agg_qwen-2] PASSED [31s]
[w0] tests/serve/...completions_only-2] PASSED [76s]
...
=============== 6 passed in 111.00s (1:51) (vs 228s seq, 2.1x) ===============
```

### 生命周期 marker 注意事项
请使用该测试**最早**应当运行的流水线阶段对应的 marker（例如 `@pytest.mark.pre_merge`）。这样 CI 会把它加入该阶段以及后续所有阶段（如 nightly、release），因为后续阶段也会选中前一阶段的测试。

**示例：**
如果某个测试打了 `@pytest.mark.pre_merge`，而 nightly 流水线运行：
```bash
pytest -m "e2e and (pre_merge or post_merge or nightly)"
```
那么该测试在 nightly 中也会被执行。

---

## Rust 测试

### 组织方式
- **单元测试** 写在对应的 Rust 源文件里（如 `lib.rs`），使用 `#[cfg(test)]` 模块。
- **集成测试** 放在 crate 的 `tests/` 目录下，且必须由 `integration` feature 控制是否启用。

### 运行 Rust 检查与测试

请按下面的顺序执行。格式与 lint 检查很快，先把它们修干净再跑测试。
这些命令来自 [`.github/workflows/pre-merge.yml`](../.github/workflows/pre-merge.yml)。

```bash
# Format check (typically <5s)
cargo fmt -- --check

# Clippy lint (typically <5min first run, faster with cache)
cargo clippy --no-deps --all-targets -- -D warnings

# License check (typically <15s)
cargo-deny -L error --all-features check licenses bans --config deny.toml

# Unused dependency check (typically <15s)
cargo machete

# Compile tests without executing (typically <5min first run; catches build errors early)
cargo test --locked --no-run

# Doc tests (typically <5min)
cargo doc --no-deps && cargo test --locked --doc

# Unit tests -- most important for code correctness (typically <5min)
cargo test --locked --all-targets

# Integration tests (may require ETCD/NATS running; typically <10min)
cargo test --features integration
```


### 其他选项
- **Feature 开关：** 使用 Cargo feature 来选择运行的测试子集，例如 `cargo test --features planner`。集成测试必须开启 `integration` feature 才会执行。
- **被忽略的测试：** 使用 `#[ignore]` 标记慢或特殊场景的测试。要执行它们，请显式运行 `cargo test -- --ignored`。

### 示例
```rust
#[cfg(test)]
mod kv_cache_tests {
    #[test]
    fn test_kv_cache_basic() {
        // ...
    }

    #[test]
    #[ignore]
    fn test_kv_cache_long_running() {
        // ...
    }
}
```

### 与 CI 的集成
- CI 会在 4 个工作区目录（`.`、`lib/bindings/python`、`lib/runtime/examples`、`lib/bindings/kvbm`）中执行 [运行 Rust 检查与测试](#running-rust-checks-and-tests) 列出的命令。具体步骤见 [`.github/workflows/pre-merge.yml`](../.github/workflows/pre-merge.yml)。

---

## Python 测试（pytest）

### 前置条件

本节假定你已经身处一个正在运行的 **runtime**、**local-dev** 或 **dev** 容器中。如果还没有，请先按 [Container Development Guide](../container/README.md) 构建并启动一个。典型工作流：

1. 构建开发容器（`render.py ...` + `docker build ...`）
2. 启动它（`run.sh ...`）
3. 在容器内编译代码并运行测试

下文所有命令都默认 **在容器内** 运行。

**Local-dev / dev 容器** —— 在跑 pytest 之前必须先编译 Rust 绑定。否则 `import dynamo._internal` 会以 `ImportError` 失败：
```bash
cargo build --locked --features dynamo-llm/block-manager --workspace
cd lib/bindings/python && maturin develop --uv && cd -
```

**Runtime 容器** —— 二进制是预编译好的，不需要再编译，直接 pytest 即可。

健全性检查（可选但推荐）—— 验证环境是否正确连通：
```bash
deploy/sanity_check.py                        # local-dev / dev containers
deploy/sanity_check.py --runtime-check-only   # runtime containers
```

### 环境配置
- 为了一致性，建议使用 dev 容器。
- 按 `pyproject.toml` 安装依赖。
- 设置 `HF_TOKEN` 环境变量，用于从 HuggingFace 下载：
  ```bash
  export HF_TOKEN=your_token_here
  ```
- 模型缓存位于 `~/.cache/huggingface`，避免重复下载。

### 运行 Python 测试

Python 这边 marker 多、变体也多。测试上会同时打 **生命周期** marker（`pre_merge`、`post_merge`、`nightly`，决定**何时**在 CI 中运行）和 **测试类型** marker（`unit`、`integration`、`e2e`，描述**测试什么**）。

**本地开发（快速反馈）** —— 提交到 CI 之前先在本地跑：
```bash
# Unit tests -- fastest (typically <15s)
pytest -m "unit and pre_merge" -v --tb=short

# Integration tests -- uses mock engines with real infrastructure (ETCD, NATS); no GPU needed (typically <10min)
pytest -m "integration and pre_merge" -v --tb=short

# E2E smoke test -- launches a full inference engine, sends requests, validates responses (typically <5min)
# vllm
pytest tests/serve/test_vllm.py::test_serve_deployment[aggregated] -v --tb=short
# sglang
pytest tests/serve/test_sglang.py::test_sglang_deployment[aggregated-2] -v --tb=short
# trtllm
pytest tests/serve/test_trtllm.py::test_deployment[aggregated-2] -v --tb=short
```

**等价于 pre-merge CI 的命令** —— 这就是 [`container-validation-dynamo.yml`](../.github/workflows/container-validation-dynamo.yml) 在每个 PR 上跑的内容。打了 `parallel` 的测试会用 `pytest-xdist` 并行执行，其余顺序执行：
```bash
# Parallel pre-merge tests (4 workers, CPU-only; typically <5min)
pytest -m "pre_merge and parallel and not (vllm or sglang or trtllm) and gpu_0" -n 4 --dist=loadscope -v --tb=short

# Sequential pre-merge tests (CPU-only; typically <10min)
pytest -m "pre_merge and not parallel and not (vllm or sglang or trtllm) and gpu_0" -v --tb=short
```

> **并行 vs 顺序：** 仅 CPU 的测试（`gpu_0`）若打了 `parallel`，会用 `pytest-xdist` 跑（`-n auto` 或 `-n <workers>`，配合 `--dist=loadscope`）。GPU 测试（`gpu_1`、`gpu_2` 等）默认顺序执行，但可通过 `--max-vram-gib=N -n auto` 转为并行（使用自定义的、感知 VRAM 的调度器，而不是 xdist）。详见 [`.github/actions/pytest/action.yml`](../.github/actions/pytest/action.yml)。

**完整 E2E 套件** —— 每个测试配置都会启动引擎；最慢，需要 GPU 与对应框架的容器（视框架与模型不同，通常 <30 分钟）：
```bash
# Sequential (default)
pytest -m "vllm and e2e and gpu_1" -v --tb=short
pytest -m "sglang and e2e and gpu_1" -v --tb=short
pytest -m "trtllm and e2e and gpu_1" -v --tb=short

# GPU-parallel (VRAM-aware scheduling, ~2x faster on 48 GiB GPU)
# Only tests with profiled_vram_gib markers are selected; -n auto calculates
# concurrent slots from GPU VRAM / smallest test. See "GPU-Parallel Execution" below.
python3 -m pytest --max-vram-gib=48 -n auto -m "gpu_1 and sglang" tests/serve/test_sglang.py -v
python3 -m pytest --max-vram-gib=48 -n auto -m "gpu_1 and vllm" tests/serve/test_vllm.py -v
```

**等价于 post-merge 的命令** —— 合入后 CI 跑 `(pre_merge or post_merge)`，会在 pre_merge 之外加入更慢的测试。**在本地完整跑一遍 post-merge 套件每个框架可能需要数小时**（模型下载、GPU 推理、多 GPU 协作）。日常开发中、提交 CI 前，请用上面的 `pre_merge` 命令获得更快反馈。具体 marker 见 [`.github/workflows/post-merge-ci.yml`](../.github/workflows/post-merge-ci.yml)：
```bash
pytest -m "(pre_merge or post_merge) and vllm and gpu_0" -n auto --dist=loadscope -v --tb=short
pytest -m "(pre_merge or post_merge) and vllm and gpu_1" -v --tb=short
```

- 按组件运行：
  ```bash
  pytest -m planner
  pytest -m kvbm
  ```
- 显示 print/log 输出：
  ```bash
  pytest -s
  ```
- CI 用类似指令在容器里运行。例如，在 post-merge 套件中跑 E2E 测试：
  ```bash
  ./container/run.sh --image $VLLM_IMAGE_NAME --name $VLLM_CONTAINER_NAME -- pytest -m "(pre_merge or post_merge) and vllm and e2e and gpu_1"
  ```

### 在容器外本地运行测试

要在开发容器之外运行测试，请确认你已经正确配置环境，并在 `venv` 中安装了以下依赖：

```bash
uv pip install pytest-mypy
uv pip install pytest-asyncio
```

---

## CI 流水线总览

强烈建议你在提交到 CI 之前，先在本地把测试彻底跑一遍。本地迭代更快、反馈即时，还能避免在共享的 CI GPU 资源上把可避免的失败"烧掉"。下面这些就是 CI 的执行阶段——你完全可以（也应当）在本机用相同命令跑一遍再提交。

源文件（完整列表见 [`.github/workflows/`](../.github/workflows/)）：
- **Pre-merge（Rust）：** [`.github/workflows/pre-merge.yml`](../.github/workflows/pre-merge.yml)
- **Pre-merge（Python）：** [`.github/workflows/container-validation-dynamo.yml`](../.github/workflows/container-validation-dynamo.yml)
- **Post-merge：** [`.github/workflows/post-merge-ci.yml`](../.github/workflows/post-merge-ci.yml) -> [`.github/workflows/build-test-distribute-flavor.yml`](../.github/workflows/build-test-distribute-flavor.yml)
- **Nightly：** [`.github/workflows/nightly-ci.yml`](../.github/workflows/nightly-ci.yml)
- **Pytest action：** [`.github/actions/pytest/action.yml`](../.github/actions/pytest/action.yml)

### Pre-merge（每个 PR）

每个 PR 上都会跑两条流水线，详见 [`pre-merge.yml`](../.github/workflows/pre-merge.yml) 与 [`container-validation-dynamo.yml`](../.github/workflows/container-validation-dynamo.yml)。

**Rust 检查**（仅当 Rust 文件有变更时）—— 先跑 `pre-commit`，再在 4 个工作区目录（`.`、`lib/bindings/python`、`lib/runtime/examples`、`lib/bindings/kvbm`）上完整执行 [运行 Rust 检查与测试](#running-rust-checks-and-tests)：format、clippy、cargo-deny、machete、compile、doc tests、单元测试。

**Python 测试**（与框架无关、仅 CPU、运行在 dynamo 容器内）：

| 阶段 | Marker 表达式 | 本地等效命令 |
|-------|------------------|-----------------|
| 并行（xdist，4 worker） | `pre_merge and parallel and not (vllm or sglang or trtllm) and gpu_0` | `pytest -m "pre_merge and parallel and not (vllm or sglang or trtllm) and gpu_0" -n 4 --dist=loadscope -v --tb=short` |
| 顺序 | `pre_merge and not parallel and not (vllm or sglang or trtllm) and gpu_0` | `pytest -m "pre_merge and not parallel and not (vllm or sglang or trtllm) and gpu_0" -v --tb=short` |

### Post-merge（推送到发布分支时）

按框架（vllm、sglang、trtllm）分别跑。每个框架都会经历：**Build** -> **Test** -> **复制到 registry**。完整 post-merge 套件每个框架要花费 **数小时**（模型下载、GPU 推理、多 GPU 测试）。

| 阶段 | 内容 | 本地等效命令 |
|-------|-------------|-----------------|
| 构建镜像 | 渲染 Dockerfile，构建 runtime 容器 | `container/render.py --framework=vllm --target=runtime && docker build ...` |
| 健全性检查 | 验证镜像内已安装的包 | `docker run --rm <image> /workspace/deploy/sanity_check.py --runtime-check --no-gpu-check` |
| 仅 CPU 测试（并行） | `(pre_merge or post_merge) and <framework> and gpu_0` | `pytest -m "(pre_merge or post_merge) and vllm and gpu_0" -n auto --dist=loadscope -v --tb=short` |
| 单 GPU 测试（顺序） | `(pre_merge or post_merge) and <framework> and gpu_1` | `pytest -m "(pre_merge or post_merge) and vllm and gpu_1" -v --tb=short` |
| 多 GPU 测试（顺序） | `(pre_merge or post_merge) and <framework> and (gpu_2 or gpu_4)` | `pytest -m "(pre_merge or post_merge) and vllm and (gpu_2 or gpu_4)" -v --tb=short` |

### Nightly（每天 PST 凌晨 0 点）

结构与 post-merge 相同，只是把 `(pre_merge or post_merge)` 换成 `nightly`：
```bash
pytest -m "nightly and vllm and gpu_1" -v --tb=short
```

### 在本地复现 CI

上文 "本地等效命令" 那一列的命令也可在 [运行 Rust 检查与测试](#running-rust-checks-and-tests) 与 [运行 Python 测试](#running-python-tests) 中找到。Rust 命令请在仓库根目录下执行，并在 4 个工作区目录（`.`、`lib/bindings/python`、`lib/runtime/examples`、`lib/bindings/kvbm`）依次跑。Python 命令在容器内运行。

---

## 其他要求

### 不稳定（flaky）测试

测试必须是确定性的。一个 flaky 测试——同样的代码在不同次运行有时通过、有时失败——会消耗 CI 时间，并削弱开发者对测试套件的信任。如果你遇到或引入了 flaky 测试：

1. **优先修掉它。** 移除非确定性来源：固定随机种子、消除竞态、mock 网络调用、不要依赖执行顺序。

   **特殊情形——基于 LLM 输出的断言。** 传 `temperature=0` 与 `seed=0` 让采样选择相同的 logits，但要把它当作**必要而非充分**条件：在 GPU 上服务的 LLM 推理仍可能因为张量并行 reduce 中的浮点非结合律、依赖 batch 规模的 kernel 选择（cuBLAS / flash-attn）、跨调用的前缀 cache 状态、非确定性的 attention kernel 等原因而存在不确定性。所以，对所有断言"模型返回内容"的测试，请同时配置重试（下面 2b）——把这视为必备项，而不是兜底。
2. **若确定性确实不可达**（如上 LLM 输出内容、模型本身就非确定、上游库存在我们不掌控的竞态等），那就重试。机制有两种，按测试性质选择：

   **2a. 整测试重试 —— `@pytest.mark.flaky`**（适用于单元测试、parser 测试、tool-calling 测试，以及任何不需要启动 server 的用例）

   `pytest-rerunfailures` 已固定在 `container/deps/requirements.test.txt`，且 `flaky` marker 已在 `pyproject.toml` 注册。请尽量缩小作用范围：
   ```python
   @pytest.mark.flaky(reruns=2, only_rerun=["AssertionError"])
   def test_named_tool_choice_forces_specific_function(...):
       ...
   ```
   - `reruns=2` 表示最多重试 2 次（共 3 次尝试）。挑能让用例稳定通过的最小数；不要拿来掩盖真 bug。
   - `only_rerun=[...]` 让重试只针对**内容/校验失败**。请使用纯异常类名。常见值：
     - `"AssertionError"` —— 直接的 pytest 断言（大多数 tool-calling 与 parser 测试）。
     - `"EngineResponseError"` —— `tests/utils/engine_process.py` 把 validator 的 `AssertionError` 包装成它。
   - **千万不要**匹配基础设施异常（`TimeoutError`、`ConnectionError`、`RuntimeError`）——那些代表真问题，重试会把它掩盖。
   - 对通过 dataclass 参数化的测试，把 marker 加到对应参数化用例的 `marks=[...]` 里，让它仅作用于出问题的那一组。
   - 代价：`@pytest.mark.flaky` 重试整个测试。对于启动 60-90s 的 e2e 测试，3 次尝试会消耗 5+ 分钟才报错。这种情况请改用 2b。

   **2b. 进程内重发请求 —— `payload.max_attempts`**（适用于走 `run_serve_deployment` 的测试 —— e2e 服务、多模态冒烟测试）

   server 只启动一次，多次尝试期间一直保持运行；只重新发请求与解析响应。在 payload 上设置 `max_attempts`：
   ```python
   # tests/serve/multimodal_profiles/vllm.py
   tests=[
       MmCase(
           payload=make_image_payload(
               ["green", "white", "black", "purple", "red", ...],
               max_attempts=3,  # known-flaky model output; see comment above
           )
       )
   ],
   ```
   关于 `MultimodalModelProfile → TopologyConfig → MmCase` 这个结构的背景，详见 [`tests/serve/multimodal_profiles/README.md`](serve/multimodal_profiles/README.md)。
   - 工厂函数（`make_image_payload`、`make_video_payload`、`chat_payload`、…）都接受 `max_attempts: int` 参数，最终落到 `BasePayload.max_attempts`。
   - `tests/serve/common.py:run_serve_deployment` 在内部把 `send_request` + `check_response` 包成一个简短的重试循环，捕获 `ResponseValidationError` 并按指数退避（1.0 → 1.5 → 2.25 → ... 秒，因子 1.5）。
   - 代价：每次尝试只是一次推理调用（数秒），不需要重启 server。60-90 秒的 server 启动开销分摊到所有尝试。
   - 取舍：只重发同一个请求——如果 flake 出在启动或模型加载阶段，它帮不上忙；那种场景请用 2a。

   **仓库内的其他重试点**（如有特殊需求，可以参考做自定义层）：
   - `tests/frontend/test_frontend_api_surface_compliance.py:_retry_network_op` —— 同步、仅针对网络异常列表。
   - `tests/router/helper.py:send_request_with_retry` —— 异步、按状态码（aiohttp）。
   - `tests/utils/managed_deployment.py` —— 同步连接重试，1.5 倍退避。
   - `components/src/dynamo/planner/connectors/remote_client.py` —— 同步指数退避（`2**attempt`）。
3. **如果重试仍不够稳定**，把测试隔离掉，避免它阻塞其他开发：
   - `@pytest.mark.skip(reason="Flaky: <ticket link>")` —— 完全禁用该测试。当测试在当前状态下根本提供不了有效信号时使用。
   - `@pytest.mark.xfail(reason="Flaky: <ticket link>", strict=False)` —— 测试仍然运行，但不会让套件失败。当你希望仍能看到通过/失败比例同时排查时使用。
   - 在 Rust 中，使用 `#[ignore]` 并附注释说明原因。
4. **为每一个重试或被隔离的测试开 ticket。** 即使是用 `@pytest.mark.flaky` 的测试也需要有人跟进——重试只是掩盖症状，根本的非确定性问题还在。
5. **隔离的测试不要超过一个 sprint。** 如果根因始终找不到，就把测试删掉重写。打了 `flaky` 的测试，目标应该是上游确定性问题修好之后把这个 marker 摘掉。

### 超时

长跑测试**必须**显式设置超时。一个挂住的测试（例如等永远不会启动的模型 server，或者死锁子进程）会卡住整个 CI 任务，浪费所有人的 GPU 时长。

- 使用 `pytest-timeout` 插件（已加入依赖）：
  ```python
  @pytest.mark.timeout(300)  # 5 minutes
  def test_e2e_inference():
      ...
  ```
- 把超时设为**实测平均耗时的 2~3 倍**。这给合理的方差（模型加载抖动、CPU 资源争抢）留够余量，又能在真挂住时及时发现。例如某个测试通常 90 秒完成，就设置 `@pytest.mark.timeout(240)`。
- Rust 中可以使用 `#[timeout(Duration::from_secs(300))]`，或在 `Cargo.toml` 中设置默认超时。
- CI 工作流也会强制设置整体 job 超时（详见 workflow YAML）。在每个测试上设置超时能比"任务整体被杀掉"更早、更清晰地定位问题。

### 何时**不应该**调高 pytest 超时（CI runner 中毒）

如果某个测试以 `pytest-timeout` 失败、或 `URL check failed after Ns`，**并不总是** "再等久一点" 的问题。在 L4 CI runner 上我们见过这样一种级联：表象像超时，根因却是 runner 环境被耗尽。

1. runner 内存耗尽（或在重测试时把磁盘塞满）。
2. OOM killer（或磁盘压力下的 kill）干掉了该测试本地的 ephemeral **etcd** 实例（动态端口如 `127.0.0.1:2387`，并不是默认的 `2379`）。
3. Dynamo worker 进程在尝试注册 endpoint / 上报指标 / 申请 lease 时反复失败：
   ```
   dynamo_runtime::transports::etcd::lease: Failed to establish keep-alive stream
   error=grpc request error: code: 'The service is currently unavailable',
   message: "tcp connect error",
   ConnectError("tcp connect error", 127.0.0.1:2387,
                Os { code: 111, kind: ConnectionRefused, message: "Connection refused" })
   ```
4. worker 永远启动不起来，测试的 URL 健康检查最终超时（往往 600 秒之后），并且**同一 runner 上后续的每个测试都继承这个被污染的状态，发生级联失败**。

如何区分级联问题和真正的"冷启动慢"：

- **级联信号：** 同一任务里多个不相关的测试一起失败（例如 `disaggregated`、`multi_node_tp_headless`、`lora_aggregated_router` 都在 URL 检查上正好 601.7s 超时）。CI 仪表盘会同时把任务标上 `oom`、`disk-space-error`、`etcd-error`。
- **真正冷启动信号：** 一个特定测试在多次运行里都在某个干净的预算边界附近失败（例如 147 秒预算下正好 146.79 秒）。调高那一个测试的超时就能解决。

如果看到的是级联信号，**不要**调高 pytest 超时。修复点在 CI 基础设施（更大 runner、测试间清理磁盘、按测试重置 runner）——而不是测试代码。

### 时间预算

- 如果某个测试超过了它对应的时间预算（见 [测试类型与位置](#test-types-and-locations)），请用 `pytest --durations=0` 做 profile，并考虑 mock 重依赖、换更小的模型 checkpoint，或用 `@pytest.mark.slow` 移到 nightly/weekly 流水线。

### 流水线时间预算

每个生命周期 marker 都对应一条 CI 流水线，并有总的 wall-clock 预算。在新增或重新打 marker 时，对应流水线必须仍能跑在预算内：

| Marker       | 流水线预算 | 理由                                                                 |
|--------------|-----------------|---------------------------------------------------------------------------|
| `pre_merge`  | < 30 分钟        | 每个 PR 都跑；快速反馈是开发者解除阻塞的关键。 |
| `post_merge` | < 1 小时          | 合入 `main` 之后跑；尽快捕获回归且不阻塞 PR。|
| `nightly`    | < 3 小时          | 每天跑一次；覆盖更长的集成与多 GPU 场景。     |

新增测试时的指引：

- 选用该测试**最轻量**的生命周期 marker。每天跑一次就够的测试不要打 `pre_merge`。
- 在把新测试打成 `pre_merge` 之前，估算它的预期耗时，并确认 pre-merge 整体仍能在 30 分钟内完成。如果不行，就降到 `post_merge` 或 `nightly`，或者给它瘦身（mock 重依赖、更小 checkpoint、更少用例）。
- 当某条流水线已逼近预算时，优先把现有的慢测试降级（`pre_merge` → `post_merge`、`post_merge` → `nightly`），而不是再加新测试。

### 时间预算的行业实践

我们的单测试时间目标参考了被广泛采用的测试规模分类：

- **Bazel test sizes** 按规模设定具体超时：small = 60 秒，medium = 300 秒（5 分钟），large = 900 秒（15 分钟），enormous = 3600 秒（1 小时）。超过该规模预期范围的测试会触发警告。（[Bazel Test Encyclopedia](https://docs.bazel.build/versions/2.0.0/test-encyclopedia.html)）
- **Software Engineering at Google**（Winters、Manshreck、Wright，2020）按资源范围分类：small 测试在单进程内运行且无 I/O；medium 测试在单机上运行；large 测试可能跨机器。Google 的目标比例大致是 80% 单元 / 15% 集成 / 5% E2E。（[Ch. 11](https://abseil.io/resources/swe-book/html/ch11.html)）
- **业内实践**（Fowler、Seemann）建议单元测试每个 1-10 ms、集成测试 ~100 ms、非 GPU 工作负载下 E2E ~1 s。一个 TDD 循环的单元套件应当在 10 秒以内跑完。（[Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)、[TDD in 10 seconds](https://blog.ploeh.dk/2012/05/24/TDDtestsuitesshouldrunin10secondsorless/)）

GPU 与模型加载开销让 Dynamo 的 E2E 测试天生比典型 web 服务的 E2E 慢。仅模型加载本身大模型常常要 30-120 秒，这就是为什么我们的 E2E 预算是 5 分钟而不是 1 秒。

---

## 故障排查

- 如果某个测试没有被跑到，请检查文件名、marker 与目录位置。
- 出现 flaky 测试时，参见上面 [Flaky 测试](#flaky-tests)：先修，必要时用 `skip`/`xfail` 隔离，并开 ticket。
- 出现慢或挂起的测试时，加上 `@pytest.mark.timeout()`（见 [超时](#timeouts)），并用 `pytest --durations=0` 做 profile。
- 模型下载失败时，确认 `HF_TOKEN` 已设置，且网络可达。
- 出现 `ImportError: cannot import name ... from 'dynamo._internal'`，说明你需要先编译 Rust 绑定（见 [前置条件](#prerequisites)）。
- 覆盖率不够时，请补充测试，或者重构代码使其更易测试。

---

## GPU VRAM Profiler（`profile_pytest.py`）

写或评审 GPU 测试时，请用 `tests/utils/profile_pytest.py` 来测量该测试真实需要多少 VRAM。该脚本会用不同的显存上限反复运行测试，通过二分搜索找到最低所需 VRAM，然后打印推荐的 pytest marker，可直接复制到测试中。

### 工作机制

profiler 会自动识别引擎类型并选择对应的二分搜索：

- **vLLM**：对 `_PROFILE_OVERRIDE_VLLM_KV_CACHE_BYTES`（字节数）做二分 → `--kv-cache-memory-bytes`。找出测试可通过的最小 KV cache 字节数后，再加 2x 安全系数。输出 `profiled_vram_gib` 与 `requested_vllm_kv_cache_bytes` marker。
- **SGLang**：对 `_PROFILE_OVERRIDE_SGLANG_MAX_TOTAL_TOKENS`（token 数）做二分 → `--max-total-tokens`。找出测试可通过的最小 KV cache token 数后，加 2x 安全系数，再以安全 token 数复跑一次以测量真实 VRAM。输出 `profiled_vram_gib` 与 `requested_sglang_kv_tokens` marker。
- **TRT-LLM**：对 `_PROFILE_OVERRIDE_TRTLLM_MAX_TOTAL_TOKENS`（token 数）做二分 → 通过 `--override-engine-args` 的 JSON 设置 `KvCacheConfig.max_tokens`。逻辑与 SGLang 相同（基于 token 二分、2x 安全系数）。输出 `profiled_vram_gib` 与 `requested_trtllm_kv_tokens` marker。对于不使用 KV cache 的非文本模型（视频/图像扩散），请用 `--no-find-min-vram` 做单次 VRAM 测量——因为模型不会输出 KV token 分配日志，二分无效。

**vLLM 要求：** 启动脚本必须遵循 `_PROFILE_OVERRIDE_VLLM_KV_CACHE_BYTES`。这由 `gpu_utils.sh` 中的 `build_vllm_gpu_mem_args` 处理（返回 `--kv-cache-memory-bytes N`）。

**SGLang 要求：** 启动脚本必须遵循 `_PROFILE_OVERRIDE_SGLANG_MAX_TOTAL_TOKENS`。这由 `gpu_utils.sh` 中的 `build_sglang_gpu_mem_args` 处理（返回 `--max-total-tokens N --mem-fraction-static 0.9`）。

**TRT-LLM 要求：** 启动脚本必须遵循 `_PROFILE_OVERRIDE_TRTLLM_MAX_TOTAL_TOKENS`（可选 `_PROFILE_OVERRIDE_TRTLLM_MAX_GPU_TOTAL_BYTES`）。这由 `gpu_utils.sh` 中的 `build_trtllm_override_args_with_mem` 处理（返回 `--override-engine-args` 所需 JSON）。注意：它独立于 `build_vllm_gpu_mem_args` / `build_sglang_gpu_mem_args`，因为 TRT-LLM 需要 JSON 合并。

**所有引擎共同要求：** 启动脚本中不要硬编码 `CUDA_VISIBLE_DEVICES`。profiler 与并行执行器会通过 `CUDA_VISIBLE_DEVICES` 把每个测试绑到特定 GPU。如果脚本里写死（如 `CUDA_VISIBLE_DEVICES=0`），就会无视分配，跑到错的 GPU 上。请改成从环境继承并带默认值：

```bash
CUDA_VISIBLE_DEVICES="${CUDA_VISIBLE_DEVICES:-0}"
```

然后把变量传给每个 worker：`CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES python3 -m dynamo.vllm ...`。对于把不同 GPU 分给不同 worker 的多 GPU 脚本，请使用带默认值的命名环境变量（如 `PREFILL_CUDA_VISIBLE_DEVICES="${PREFILL_CUDA_VISIBLE_DEVICES:-0}"`）。

### 各引擎映射

启动脚本会调用 `examples/common/gpu_utils.sh` 中的引擎专属函数，它们检查环境变量 override 并返回相应 CLI 参数：

```bash
# vLLM
GPU_MEM_ARGS=$(build_vllm_gpu_mem_args)
python -m dynamo.vllm --model "$MODEL" $GPU_MEM_ARGS &

# SGLang
GPU_MEM_ARGS=$(build_sglang_gpu_mem_args)
python -m dynamo.sglang --model-path "$MODEL" $GPU_MEM_ARGS &

# TRT-LLM (requires JSON merging, separate function)
OVERRIDE_JSON=$(build_trtllm_override_args_with_mem)
python -m dynamo.trtllm --model-path "$MODEL" ${OVERRIDE_JSON:+--override-engine-args "$OVERRIDE_JSON"} &
```

下列环境变量在 profile 与并行测试期间控制引擎的内存分配：

**`_PROFILE_OVERRIDE_VLLM_KV_CACHE_BYTES`**（整数）—— 仅 vLLM：

| 引擎  | 返回的 CLI 参数                | 备注 |
|---------|----------------------------------|-------|
| vLLM    | `--kv-cache-memory-bytes N`      | KV cache 的精确字节上限；确定性、并发安全 |

**`_PROFILE_OVERRIDE_SGLANG_MAX_TOTAL_TOKENS`**（整数）—— 仅 SGLang：

| 引擎  | 返回的 CLI 参数                                  | 备注 |
|---------|----------------------------------------------------|-------|
| SGLang  | `--max-total-tokens N --mem-fraction-static 0.9`   | 基于 token 的 KV cache 上限；`--mem-fraction-static 0.9` 让小显存 GPU 上的池创建门槛仍能通过（详见 PR #9238） |

**`_PROFILE_OVERRIDE_TRTLLM_MAX_TOTAL_TOKENS`**（整数）—— TRT-LLM 文本模型：

| 引擎  | 返回的 JSON                                          | 备注 |
|---------|--------------------------------------------------------|-------|
| TRT-LLM | `{"kv_cache_config": {"max_tokens": N}}`              | 通过 `--override-engine-args` 的基于 token 的 KV cache 上限 |

**`_PROFILE_OVERRIDE_TRTLLM_MAX_GPU_TOTAL_BYTES`**（整数）—— TRT-LLM 非文本模型：

| 引擎  | 返回的 JSON                                                    | 备注 |
|---------|------------------------------------------------------------------|-------|
| TRT-LLM | `{"kv_cache_config": {"max_gpu_total_bytes": N}}`               | 通过 `--override-engine-args` 的基于字节的上限。用于扩散模型。 |

它们都使用绝对上限——确定性、与当前可用显存无关，这是测试并行执行的关键。详见 `examples/common/gpu_utils.md`。

### 用法

```bash
# vLLM: binary search for minimum KV cache bytes
python tests/utils/profile_pytest.py tests/serve/test_vllm.py::test_serve_deployment[aggregated] -xvs

# Profile on a specific GPU (default: 0)
python tests/utils/profile_pytest.py --gpu 1 tests/serve/test_vllm.py::test_serve_deployment[aggregated] -xvs

# SGLang: binary search for minimum KV cache tokens (automatic)
python tests/utils/profile_pytest.py tests/serve/test_sglang.py::test_sglang_deployment[aggregated-2] -xvs

# TRT-LLM: binary search for minimum KV cache tokens (text models)
python tests/utils/profile_pytest.py tests/serve/test_trtllm.py::test_deployment[aggregated-2] -xvs

# TRT-LLM: single-pass for diffusion models (no KV cache, binary search won't work)
python tests/utils/profile_pytest.py --no-find-min-vram tests/serve/test_trtllm.py::test_deployment[video_diffusion-2] -xvs

# Single-pass profiling (no binary search, just measure one run using default RAM)
python tests/utils/profile_pytest.py --no-find-min-vram tests/serve/test_vllm.py::test_serve_deployment[aggregated]
```

### 示例输出（vLLM）

```bash
========================================================================
FIND MINIMUM KV CACHE BYTES (vLLM, deterministic) (binary search)
========================================================================
  GPU total : 48.0 GiB
  GPU free  : 47.4 GiB  (in use: 0.6 GiB)
  Test      : tests/serve/test_vllm.py::test_serve_deployment[aggregated] -x

  [probe 1] Validation run: kv_cache=23296 MiB (50% of free)
  [PASS] peak 2.9 GiB, wall 42s, iter took 49s
  ...
  [probe 6/15] kv_cache=449 MiB (471,027,000 bytes)
  [PASS] peak 2.9 GiB, wall 41s, iter took 49s

  [probe 7/15] kv_cache=224 MiB (235,513,856 bytes)
  [FAIL] OOM, iter took 30s

========================================================================
  Minimum KV cache : 449 MiB (471,027,000 bytes)
  Safe KV cache    : 898 MiB (942,054,000 bytes) (2x safety)
  Peak VRAM        : 2.9 GiB

  Recommended markers:
    @pytest.mark.profiled_vram_gib(2.9)
    @pytest.mark.requested_vllm_kv_cache_bytes(942_054_000),  # KV cache cap (2x safety over min=471_027_000)
========================================================================

========================================================================
Recommended markers to add to your pytest. You can copy-paste this:
========================================================================
# Measured using: tests/utils/profile_pytest.py tests/serve/test_vllm.py::test_serve_deployment[aggregated]
@pytest.mark.e2e  # wall time 41.2s, loads a real model
@pytest.mark.gpu_1  # 1 GPU(s) used, peak 2.9 GiB
@pytest.mark.profiled_vram_gib(2.9)  # actual nvidia-smi peak
@pytest.mark.requested_vllm_kv_cache_bytes(942_054_000)  # KV cache cap (2x safety over min=471_027_000)
@pytest.mark.timeout(124)  # 3x observed 41.2s

  WARNING: Wall time 41.2s is too slow for pre_merge (> 20s). Consider post_merge or nightly instead.
========================================================================
```

### 示例输出（SGLang —— 基于 token 的二分）

```bash
========================================================================
FIND MINIMUM KV TOKENS (SGLang) (binary search)
========================================================================
  GPU total : 48.0 GiB
  GPU free  : 47.4 GiB  (in use: 0.6 GiB)
  Test      : tests/serve/test_sglang.py::test_sglang_deployment[aggregated-2] -xvs

  [probe 1] Validation run (no token cap)
  [PASS] peak 43.0 GiB, wall 36s, max_total_tokens=366688, iter took 44s
  ...
  [probe 14/15] tokens=48  [~1 left, ETA ~45s]
  [PASS] tokens=48, peak 3.7 GiB, wall 26s, iter took 34s
  [final probe] Measuring VRAM at safe_tokens=96
  [PASS] tokens=96, peak 3.7 GiB, wall 27s

========================================================================
MINIMUM KV TOKENS RESULT
========================================================================
  Minimum tokens  : 16 (raw bisection result)
  Recommended     : 96 (2x safety)
  Peak VRAM       : 3.7 GiB (at 96 tokens)
  @pytest.mark.profiled_vram_gib(3.7)
  @pytest.mark.requested_sglang_kv_tokens(96),  # KV cache cap (2x safety over min=48)
========================================================================
```

### 示例输出（TRT-LLM —— 基于 token 的二分）

```bash
========================================================================
FIND MINIMUM KV TOKENS (TensorRT-LLM) (binary search)
========================================================================
  GPU total : 48.0 GiB
  GPU free  : 47.1 GiB  (in use: 0.9 GiB)
  Test      : tests/serve/test_trtllm.py::test_deployment[aggregated-2] -xvs

  [probe 1] Validation run (no token cap, default fraction)
  [PASS] peak 41.3 GiB, wall 48s, max_tokens=41472 (TensorRT-LLM), iter took 56s
  ...
  [probe 6/12] tokens=1296
  [PASS] tokens=1296, peak 3.7 GiB, wall 46s, iter took 54s
  [EARLY STOP] Peak VRAM stable for last 3 probes
  [final probe] Measuring VRAM at safe_tokens=2592
  [PASS] tokens=2592, peak 3.9 GiB, wall 46s

========================================================================
MINIMUM KV TOKENS RESULT (TensorRT-LLM)
========================================================================
  Minimum tokens  : 1296 (raw bisection result)
  Recommended     : 2592 (2x safety)
  Peak VRAM       : 3.9 GiB (at 2592 tokens)
  @pytest.mark.profiled_vram_gib(3.9)
  @pytest.mark.requested_trtllm_kv_tokens(2592),  # KV cache cap (2x safety over min=1296)
========================================================================
```

### 如何使用这些建议

1. **复制 `@pytest.mark.*` 行** 到你的测试函数或 `pytestmark` 列表。

2. **VRAM marker** —— `profiled_vram_gib(N)` 记录 nvidia-smi 实测峰值（用于过滤/调度），`requested_vllm_kv_cache_bytes(N)` 或 `requested_sglang_kv_tokens(N)` 控制引擎的 KV cache 分配以保证并发执行的确定性。可使用 `--max-vram-gib=N` 取消选择 profile 后 VRAM 超过 N 的测试（详见 [按 VRAM 过滤](#filtering-by-vram)）。profiler 输出中的 WARNING 行会告诉你哪些 GPU 档位太小（例如 "Will OOM on T4 (16 GiB)"）。

3. **生命周期 marker** —— profiler 只对 20 秒以内的测试推荐 `pre_merge`。对更慢的测试，它会提示你考虑 `post_merge` 或 `nightly`，但不替你决策——请根据该测试对捕获回归有多关键来判断。

4. **Timeout** —— 推荐值是观测 wall time 的 3 倍。如果你的测试方差较大（首次运行模型下载、网络抖动等），可适当上调。

5. **测试类型**（`unit`、`integration`、`e2e`）—— 由 wall time 与"是否加载真实模型"推断。如果你更清楚就直接覆盖（例如使用 mock 引擎的快速测试是 `integration`，不是 `e2e`）。

### 选项

| Flag | 描述 |
|------|-------------|
| `--kv-bytes` | 已废弃保留兼容。vLLM 始终对 `--kv-cache-memory-bytes` 做二分 |
| `--no-find-min-vram` | 跳过二分，仅做单次 profile |
| `--interval N` | GPU 采样间隔（秒），默认 1.0 |
| `--baseline-seconds N` | 在启动 pytest 之前的采样秒数，默认 3.0 |
| `--teardown-seconds N` | 在 pytest 退出之后的采样秒数，默认 5.0 |
| `--csv FILE` | 把原始 nvidia-smi 采样写入 CSV 文件 |
| `--no-recommend` | 不输出 marker 推荐 |

---

## 参考资料
- [pytest documentation](https://docs.pytest.org/en/stable/)
- [Bazel Test Encyclopedia — test sizes and timeouts](https://docs.bazel.build/versions/2.0.0/test-encyclopedia.html)
- [Software Engineering at Google — Testing Overview (Ch. 11)](https://abseil.io/resources/swe-book/html/ch11.html)
- [Martin Fowler — The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
- [Mark Seemann — TDD test suites should run in 10 seconds or less](https://blog.ploeh.dk/2012/05/24/TDDtestsuitesshouldrunin10secondsorless/)

如需进一步协助，请联系 Dynamo 开发团队。
