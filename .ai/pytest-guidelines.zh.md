# Pytest 指南

本仓库 Python 测试的规则与约定。

## 运行测试

**推到 CI 之前请始终在本地先跑测试。** 每次失败的 CI 运行都会浪费共享 GPU 计算资源，并阻塞他人的 PR。本地运行可在数秒内捕获大多数失败。

### 在本地复现 CI 故障

不要从日志片段中猜测。拉取与 CI 构建相同的容器镜像，在本地复现：

```bash
# 1. Find the image:tag in the CI job's "docker build" or "docker push" step.

# 2. Run it with GPU access:
docker run --rm -it --gpus all <ci-image>:<tag> bash

# 3. Run the failing test:
python3 -m pytest -xvv tests/path/to/test_that_failed.py::test_name
```

如果在 CI 容器中本地通过，可能是 CI 上的资源或时序问题。如果失败了，那你就有了精确的复现。

拉取容器、复现、修复、验证——按这个顺序执行。不要从日志做投机性修复。

始终使用感知 venv 的调用方式——绝不要直接 `pytest`：

```bash
export HF_HUB_OFFLINE=1 HF_TOKEN="$(cat ~/.cache/huggingface/token)"
python3 -m pytest -xvv --basetemp=/tmp/pytest_temp --durations=0 tests/
```

### 针对预先填充的本地模型缓存运行

如果你已经将模型下载到只读目录（如共享 NFS 挂载或 bind 挂载卷），传入 `--models-dir` 即可跳过所有网络下载，并避免任何对缓存的写入：

```bash
python3 -m pytest --models-dir=/path/to/hf_cache -xvv tests/serve/test_vllm.py
```

可接受 **裸 `HF_HUB_CACHE` 目录**（包含 `models--org--name/` 子目录）或 **`HF_HOME` 目录**（自动检测：若存在 `hub/` 子目录则使用 `HF_HOME`，否则使用 `HF_HUB_CACHE`）。检测到 `HF_HOME` 布局时会打印一条警告以便核对。

`--models-dir` 的作用：
- 将 `HF_HUB_CACHE`（或 `HF_HOME`）设置为给定路径。
- 启用 `HF_HUB_OFFLINE=1` 与 `TRANSFORMERS_OFFLINE=1`——不进行任何网络调用。
- 短路 `predownload_models` 与 `predownload_tokenizers`——不写入缓存目录。
- 设置 `DYNAMO_MODELS_DIR`——本会执行网络下载的代码（如 `download_lora()` 中的 LoRA 适配器下载）将通过 `pytest.skip()` 跳过而非失败。

**LoRA 测试与 `--models-dir` 不兼容**，因为它们在测试时从 HuggingFace Hub 下载适配器。当该参数生效时，调用 `download_lora()` 的测试会带着清晰提示自动跳过。要在本地运行 LoRA 测试，请省略 `--models-dir` 并确保设置了 `HF_TOKEN`。

- `python3 -m pytest` 确保 venv 中的 pytest 以正确的 `sys.path` 运行。系统的 `pytest`（位于 `/usr/local/bin/pytest`）位于 venv **之外**，无法看到 venv 安装的包（如 `dynamo`）。
- `-xvv` 在首次失败时停止，并以详细模式输出。
- `--durations=0` 显示所有测试的耗时（便于发现慢/不稳定测试）。

### 按 marker 过滤

```bash
python3 -m pytest -m "vllm and gpu_1 and pre_merge" -v
python3 -m pytest -m "vllm and e2e and gpu_1" -v
python3 -m pytest -m "vllm and unit and gpu_0" -v
python3 -m pytest tests/serve/ -m "vllm and gpu_1 and pre_merge" -vv --tb=short
```

本地用 `--durations=10` 找出最慢的 10 个测试。

### 按关键字过滤（`-k`）

使用 `-k` 按名称模式选择测试。注意 `-k` 是子串匹配：

```bash
# BAD -- also matches "disaggregated" tests
python3 -m pytest tests/serve/ -k "aggregated" -v

# GOOD -- excludes disaggregated
python3 -m pytest tests/serve/ -k "aggregated and not disagg" -v --tb=short
```

## 关键规则

这些是不稳定、非自洽测试最常见的来源。任何违反都会阻塞你的 PR。

### 不要硬编码端口

测试代码中的字面端口号（`port=8000`、`port=8081`）**始终予以标记**。共享任何资源（端口、文件、env var 等）的并行测试都会冲突。

使用 `dynamo_dynamic_ports`（按测试分配 `frontend_port` + `system_ports`）或 `tests.utils.port_utils` 中的 `allocate_port()` / `allocate_ports()`。

```python
# BAD
resp = requests.get("http://localhost:8000/v1/models")

# GOOD
def test_example(dynamo_dynamic_ports):
    port = dynamo_dynamic_ports.frontend_port
    resp = requests.get(f"http://localhost:{port}/v1/models")
```

### 不要硬编码临时路径

测试代码中固定路径（如 `/tmp/my-test.log` 或 `/tmp/output/`）**始终予以标记**。并行 worker 会互相覆盖文件。

使用 pytest 的 `tmp_path` fixture 或 Python 的 `tempfile` 模块——两者都提供唯一路径并自动清理。

```python
# BAD -- hardcoded path collides with parallel tests
with open("/tmp/test-output.json", "w") as f:
    json.dump(result, f)

# BAD -- "ghost fixture": accepts tmp_path but ignores it and writes to /tmp anyway.
# Flag any test that requests tmp_path but still references /tmp/ or hardcoded paths.
def test_example(tmp_path):
    with open("/tmp/test-output.json", "w") as f:
        json.dump(result, f)

# GOOD
def test_example(tmp_path):
    out = tmp_path / "test-output.json"
    out.write_text(json.dumps(result))
```

### 不要把输出文件写入仓库目录树

任何向相对于 `__file__` 或仓库根的路径写入的测试**始终予以标记**。这会污染工作树并在 `git status` 中产生未跟踪的噪声。

**例外**：autouse 的 `logger` fixture 按设计写入 `test_output/<test_name>/`——这是经过认可的共享基础设施，不是临时测试输出。请勿将其标记。

```python
# BAD -- writes into the repo alongside the test file; flag this
output = os.path.join(os.path.dirname(__file__), "scratch_output.txt")
with open(output, "w") as f:
    f.write("debug output\n")

# GOOD -- use tmp_path; cleaned up automatically
def test_example(tmp_path):
    output = tmp_path / "scratch_output.txt"
    output.write_text("debug output\n")
```

### 不要自己写引擎启动/停止逻辑

手撸的 `subprocess.Popen` / `os.system` / `time.sleep` 处理引擎或基础设施生命周期的代码**始终予以标记**。自制的生命周期代码会泄漏进程、在失败时遗漏清理，并与并行测试发生竞争。

使用既有的 fixture 与上下文管理器：

- **Fixture：** `runtime_services_dynamic_ports`、`start_services_with_http`、
  `start_services_with_grpc`、`start_services_with_mocker`
- **上下文管理器：** `DynamoFrontendProcess`、`DynamoWorkerProcess`、
  `ManagedProcess`、`EtcdServer`、`NatsServer`

它们会自动处理健康检查、端口分配、日志捕获与优雅关停。如有需要，请扩展共享 fixture——不要自行重新发明。

```python
# BAD -- hand-rolled subprocess management
proc = subprocess.Popen(["python3", "-m", "dynamo.mocker", ...])
time.sleep(10)  # hope it's ready
try:
    run_test()
finally:
    proc.kill()

# GOOD -- use the provided fixture
def test_example(start_services_with_mocker):
    frontend_port = start_services_with_mocker
    # engine is already up, health-checked, and will be cleaned up automatically
```

### 不要复制粘贴测试基础设施——复用并重构

跨测试文件重复的 setup 逻辑、辅助函数或 fixture 代码**始终予以标记**。复制粘贴出的基础设施意味着 bug 在一处修复后另一处仍然存在。

- 在新写之前先查看 `tests/conftest.py`、子目录 `conftest.py` 与 `tests/utils/`。
- 如果两个或更多测试共享 setup，将其抽取为 fixture 或放入 `tests/utils/`。
- 如果测试仅在配置上不同，使用带间接 fixture 的 `@pytest.mark.parametrize`，不要再写多个独立函数。

```python
# BAD -- same setup copy-pasted across three test files
def test_vllm_chat():
    proc = start_engine("vllm", model="Qwen/Qwen3-0.6B")
    wait_for_ready(proc)
    resp = send_chat_request(proc.port)
    assert resp.status_code == 200

def test_vllm_completion():       # 90% identical to above
    proc = start_engine("vllm", model="Qwen/Qwen3-0.6B")
    wait_for_ready(proc)
    resp = send_completion_request(proc.port)
    assert resp.status_code == 200

# GOOD -- shared fixture, parametrized payloads
@pytest.mark.parametrize("payload_fn", [chat_payload_default, completion_payload_default])
def test_vllm_requests(start_serve_deployment, payload_fn):
    resp = send_request(start_serve_deployment.port, payload_fn())
    assert resp.status_code == 200
```

请扩展共享代码，而非分叉私有副本。

---

## Markers

强制启用 `--strict-markers` 与 `--strict-config`。使用未注册的 marker
**会导致收集失败**。所有 marker 须同时在 `pyproject.toml` 与
`tests/conftest.py:pytest_configure` 中注册。

### 必需的 marker

每个测试至少要有：

1. **调度 marker** —— 决定测试在 CI 中何时运行：
   - `pre_merge` —— 每个 PR 合并前运行
   - `post_merge` —— 合并到 main 后运行
   - `nightly` —— 每夜运行
   - `weekly` —— 每周运行
   - `release` —— 在发布流水线运行

2. **GPU marker** —— 需要多少 GPU：
   - `gpu_0` —— 不需要 GPU
   - `gpu_1` —— 单 GPU
   - `gpu_2`、`gpu_4`、`gpu_8` —— 多 GPU

3. **类型 marker** —— 测试种类：
   - `unit` —— 单元测试
   - `integration` —— 集成测试
   - `e2e` —— 端到端测试

### 调度 marker 选择指引

CI 计算资源是有限的。请慎重选择放置位置：

- 仅对 **极其关键** 的测试使用 `pre_merge` —— 每条 pre-merge 测试都会拖慢每位贡献者的每个 PR。
- **平均超过 60 秒的测试默认应放到 `post_merge`**，除非它守护着关键路径，值得阻塞每个 PR。如果测试虽慢仍必须留在 `pre_merge`，请添加注释说明原因。
- E2E 测试涉及组件更多，更易抖动。除非守护关键路径，否则 E2E 应优先 `post_merge`。
- 昂贵、GPU 密集或压力测试可考虑 `nightly` 或 `weekly`。

### 框架 marker

当测试依赖特定推理后端时使用：
- `vllm`、`trtllm`、`sglang`

### 超时

超过 30 秒的测试 **必须** 带 `@pytest.mark.timeout(<seconds>)`。超时设为 **测得平均值的 3 倍** 以容忍方差。

任何运行超过 30 秒，或包含 `time.sleep()`、轮询、网络调用、子进程等待但缺少 `@pytest.mark.timeout(...)` 的测试**始终予以标记**。这是 **必改而非样式建议** —— 缺失超时可能让 CI 无限挂起。

也要 **标记** 在 `pre_merge` + `unit` 测试中真实使用 `time.sleep()` 的情况。单元测试不应消耗墙钟时间。请 mock sleep、使用共享 fixture，或重新分类为带 `@pytest.mark.slow` 的 `integration`/`e2e`。

```python
# BAD -- sleeps and loops with no timeout marker; can hang CI forever
@pytest.mark.pre_merge
@pytest.mark.gpu_0
@pytest.mark.unit
def test_poll_server():
    for _ in range(50):
        time.sleep(0.1)
    assert True

# GOOD -- timeout prevents infinite hangs
@pytest.mark.timeout(300)  # ~100s average, 3x buffer
@pytest.mark.pre_merge
@pytest.mark.gpu_0
@pytest.mark.unit
def test_vllm_aggregated(...):
    ...
```

时间注释让 AI/自动化在重排测试套件时能理解需求。

### 其他常用 marker

- `model("org/model-name")` —— 声明所用 HF 模型；`predownload_models` fixture 据此只下载所需。
- `slow` —— 已知较慢的测试。
- `parallel` —— 可与 pytest-xdist 并行。
- `h100` —— 需要 H100 硬件。
- `fault_tolerance`、`deploy`、`router`、`planner`、`kvbm` —— 组件 marker。
- `k8s` —— 需要 Kubernetes。

### 示例

```python
@pytest.mark.pre_merge
@pytest.mark.gpu_1
@pytest.mark.e2e
@pytest.mark.vllm
@pytest.mark.model("Qwen/Qwen3-0.6B")
@pytest.mark.timeout(300)
def test_vllm_aggregated(start_serve_deployment):
    ...
```

## 自洽（Hermetic）测试

测试必须互相隔离。每个测试都要能以任意顺序、在任意机器上运行，并产生确定性结果且没有副作用。多个测试必须能并行执行而不冲突。

### 其他反模式

- **模块级可变状态。** 模块作用域的可变对象（`{}`、`[]`、`set()`）若被测试读写则**始终予以标记**。这会让测试有顺序依赖，并产生 xdist 的幻觉性失败。

  ```python
  # BAD -- module-level dict shared across all tests; flag this
  _shared_results = {}

  def test_a():
      _shared_results["worker-1"] = "registered"

  def test_b():
      # Passes only if test_a ran first!
      assert _shared_results["worker-1"] == "registered"

  # GOOD -- each test gets its own state
  @pytest.fixture
  def results():
      return {}

  def test_a(results):
      results["worker-1"] = "registered"
      assert results["worker-1"] == "registered"
  ```

- **跨测试冲突的 `dyn://` 注册路径。** Dynamo worker 在 etcd/NATS 中以 `dyn://{namespace}.{component}.{endpoint}` 注册。硬编码 namespace、component、endpoint 字符串本身没问题——问题在于两个共享 etcd/NATS 实例的测试使用 **完全相同的路径**，并行执行时会发生抖动性冲突。

  对于完整 `dyn://` 路径可能与另一测试冲突的，**始终予以标记**。最简方案是至少随机化一个段（通常是 namespace）。

  ```python
  # BAD -- two tests using this identical path will collide
  namespace = "dynamo"
  component = "backend"
  endpoint = f"dyn://{namespace}.{component}.generate"

  # GOOD -- unique namespace prevents collisions; component/endpoint can stay fixed
  from tests.router.common import generate_random_suffix
  namespace = f"dynamo-{generate_random_suffix()}"
  component = "backend"
  endpoint = f"dyn://{namespace}.{component}.generate"
  ```

- **泄漏环境变量。** 在测试中直接 `os.environ[...] = ...` 或 `os.environ.update(...)`**始终予以标记**。这些修改会影响后续测试，导致顺序依赖性失败。

  ```python
  # BAD -- env var leaks into every test that runs after this one; flag this
  def test_service_discovery():
      os.environ["NATS_SERVER"] = "nats://rogue-server:4222"
      assert connect()

  # GOOD -- monkeypatch auto-restores after each test
  def test_service_discovery(monkeypatch):
      monkeypatch.setenv("NATS_SERVER", "nats://rogue-server:4222")
      assert connect()
  ```

- **测试辅助函数中可变默认参数。** 任何含可变默认值（`[]`、`{}`、`set()`）的函数**始终予以标记**。默认值只求值一次，调用之间共享，副作用会无声累积。

  ```python
  # BAD -- registry list is shared across all calls; flag this
  def register_workers(new_worker, registry=[]):
      registry.append(new_worker)
      return registry

  # GOOD -- None sentinel, fresh list each call
  def register_workers(new_worker, registry=None):
      if registry is None:
          registry = []
      registry.append(new_worker)
      return registry
  ```

  另请参阅：`python-guidelines.md` > "Mutable default arguments"。

### 优化建议

- 当多个测试共享相同部署配置时，把多个断言合并到一次引擎启动/拆除周期中。
- 当测试不需要真实推理时，使用 mock 引擎（`dynamo.mocker`）替代真实的 vLLM/SGLang/TRT-LLM 引擎。
- Mock 外部服务（API、数据库等），保持测试快速且确定性。

## Fixtures

### 服务基础设施

- **`runtime_services_dynamic_ports`** —— 适合 xdist 安全的测试。在动态端口上为每个测试启动 NATS 与 etcd，设置 `NATS_SERVER` / `ETCD_ENDPOINTS` env，结束后清理。
- **`runtime_services`** —— 更简单，使用默认端口。非 xdist 安全。
- **`runtime_services_session`** —— session 作用域，通过文件锁在 xdist worker 之间共享。适合按测试启动开销过大的大型套件。

### 端口分配

- **`dynamo_dynamic_ports`** —— 按测试分配 `frontend_port` + `system_ports`。请勿在测试中硬编码端口（8000、8081 等）。
- **`num_system_ports`** —— 默认 1。要更多请用间接 parametrize：
  `@pytest.mark.parametrize("num_system_ports", [2], indirect=True)`

### 模型管理

- **`predownload_models`**（session 作用域） —— 下载完整模型。读取所收集测试的 `@pytest.mark.model(...)`，仅下载所需。下载后设置 `HF_HUB_OFFLINE=1`，让 worker 跳过冗余 API 调用。
- **`predownload_tokenizers`**（session 作用域） —— 同上，但跳过权重文件。

### 后端特定 parametrize

- **`discovery_backend`** —— 默认 `"etcd"`。可 parametrize 为 `["file", "etcd"]`。
- **`request_plane`** —— 默认 `"nats"`。可 parametrize 为 `["nats", "tcp"]`。
- **`durable_kv_events`** —— 默认 `False`。设为 `[True]` 启用 JetStream 模式。

### 日志

autouse 的 `logger` fixture 将每个测试的日志写入 `test_output/<test_name>/test.log.txt`。
某些子套件（如 `tests/planner/`）会用一个 no-op fixture 覆盖它。

## xdist / 并行安全

- 使用 `runtime_services_dynamic_ports` + `dynamo_dynamic_ports` 实现端口隔离。
- 使用 `SharedEtcdServer` / `SharedNatsServer`（通过 `runtime_services_session`）以文件锁协调实现 session 作用域共享服务。
- 永远不要依赖跨 worker 的固定端口或全局状态。
- 每个 xdist worker 是独立进程——env 变量不会泄漏。

## 警告

全局设置了 `filterwarnings = ["error"]`，并对已知第三方废弃（CUDA、protobuf、pynvml、torchao 等）做了特定忽略。如果你的测试触发了新警告，要么解决根因，要么在 `pyproject.toml` 中添加针对性的忽略并附注释说明原因。

## 测试中的错误处理

- 不要使用泛 `except Exception` —— 让失败传播。
- 仅捕获你确实可处理的特定异常。
- 倾向使用 fixture 进行 setup/teardown，而不是测试体内的 try/finally。

## Linter 抑制（`# noqa`）

抑制本指南所记录反模式相关警告的 `# noqa`**始终予以标记**。如果 linter 抓到了真问题（`E711` 表 `== None`、`E712` 表 `== True`、`F841` 表未使用变量），请修代码。

```python
# BAD -- noqa hides the very bug the linter caught; flag this
assert error == None  # noqa: E711
result = compute()    # noqa: F841

# GOOD -- fix the code
assert error is None
result = compute()
assert result == expected
```

唯一可接受的 `# noqa` 用于真正的误报，并须始终解释原因：
`# noqa: F401 -- imported for side-effects`。

## 测试文件组织

```
tests/
  conftest.py              # Root fixtures: services, ports, model downloads, logging
  serve/                   # Backend serve tests (vllm, trtllm, sglang)
    conftest.py            # Image server, MinIO LoRA fixtures
  frontend/                # Frontend HTTP/gRPC tests
    conftest.py            # HTTP/gRPC service fixtures, mocker workers
    grpc/                  # gRPC-specific tests
  planner/                 # Planner component tests
    unit/                  # Planner unit tests
  router/                  # Router E2E tests
  fault_tolerance/         # Fault tolerance tests
    cancellation/
    migration/
    etcd_ha/
    gpu_memory_service/
    deploy/
  kvbm_integration/        # KV block manager integration tests
  deploy/                  # Deployment tests
  basic/                   # Basic smoke tests (wheel contents, CUDA version)
  dependencies/            # Import/dependency tests
  utils/                   # Shared test utilities (NOT test files)
    constants.py           # Model IDs, default ports
    managed_process.py     # ManagedProcess for subprocess lifecycle
    port_utils.py          # Dynamic port allocation
    test_output.py         # Test output path resolution
```

## Serve 测试模式

后端 serve 测试（`tests/serve/test_vllm.py` 等）遵循配置驱动模式：

```python
vllm_configs = {
    "aggregated": VLLMConfig(
        name="aggregated",
        directory=vllm_dir,
        script_name="agg.sh",
        marks=[pytest.mark.gpu_1, pytest.mark.pre_merge, pytest.mark.timeout(300)],
        model="Qwen/Qwen3-0.6B",
        request_payloads=[...],
    ),
}
```

通过 `params_with_model_mark()` 将配置参数化进测试函数，它会自动从配置的 model 字段应用 `model` marker。
