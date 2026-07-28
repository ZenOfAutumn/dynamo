# KV 行为与模型确定性测试（kvbm）

## 概述

该测试套件用于在固定采样参数下（以及在 prefix cache 重置前后）验证基于 API 的 LLM 的确定性属性。这些测试可以自动启动一个本地 LLM 服务器（vLLM 服务器或 TensorRT-LLM 服务器），完成预热，并对相同 prompt 在多次迭代中的响应进行比较。该套件还会自动检测当前安装的是 vLLM 还是 TensorRT-LLM wheel，并启动相应的服务器。

## 文件

- `test_determinism.py` —— 全面的确定性测试，自动管理 LLM 服务器生命周期与预热。
  - `test_determinism_with_cache_reset` —— 先带预热运行测试，然后重置缓存，再不带预热运行一次，以验证跨缓存重置边界的确定性
  - `test_concurrent_determinism_with_ifeval` —— 以可控的并发度发送参数化数量的 IFEval prompt（默认：120），先带预热，再重置缓存并不带预热再次测试，以验证跨缓存重置的确定性

## 标记（Markers）

- `kvbm` —— KV 行为与模型确定性测试
- `e2e` —— 端到端测试
- `slow` —— 测试可能因预热/迭代而较慢
- `nightly` —— 适合 nightly 流水线运行

## 工作原理

- `LLMServerManager` fixture（`llm_server`）会启动 `vllm serve` 或 `trtllm-serve`，并加载 Dynamo connector 以及可选的 cache block 覆盖参数。
- `tester` fixture 将测试客户端绑定到正在运行的服务器 base URL。
- 测试会先在多个 prompt 上做全面预热，然后执行重复请求，并检查响应是否完全一致（确定性）。可选的缓存重置阶段会重新验证跨重置边界的确定性。

## 运行

运行所有 kvbm 测试：

```bash
pytest -v -m "kvbm" -s
```

直接在 dynamo 仓库内运行确定性测试文件：

```bash
pytest -v tests/kvbm_integration/test_determinism_agg.py -s

# disagg 需要 2 个 GPU 才能运行
pytest -v tests/kvbm_integration/test_determinism_disagg.py -s
```

## 配置

可以通过环境变量控制服务器设置与测试负载：

- 服务器 / 模型
  - `KVBM_MODEL_ID`（默认：`deepseek-ai/DeepSeek-R1-Distill-Llama-8B`）
  - `KVBM_SERVER_PORT`（默认：`8000`）
  - `KVBM_SERVER_START_TIMEOUT`（默认：`300` 秒）

- 缓存大小覆盖
  - `KVBM_CPU_BLOCKS`（通过测试参数化使用；默认：`10000`）
  - 当 `gpu_blocks` 被参数化时，会附加 `--num-gpu-blocks-override`

- 请求 / 测试参数
  - `KVBM_MAX_TOKENS`（默认：`48`）—— 每个请求的最大 token 数（单一整数）
  - `KVBM_SEED`（默认：`42`）
  - `KVBM_MAX_ITERATIONS`（默认：`500`）
  - `KVBM_WORD_COUNT`（默认：`200`）
  - `KVBM_CONTROL_INTERVAL`（默认：`10`）
  - `KVBM_SHAKESPEARE_INTERVAL`（默认：`1`）
  - `KVBM_RANDOM_INTERVAL`（默认：`7`）
  - `KVBM_HTTP_TIMEOUT`（默认：`30` 秒）
  - `KVBM_SHAKESPEARE_URL`（默认：MIT OCW 的 Shakespeare 文本）

- 并发测试（仅对 `test_concurrent_determinism_with_ifeval` 生效）
  - `KVBM_CONCURRENT_REQUESTS`（默认：`3`）—— 用于参数化最大并发 worker 数的逗号分隔列表
  - `KVBM_IFEVAL_PROMPTS`（默认：`120`）—— 用于参数化 IFEval prompt 数量的逗号分隔列表

### 示例

```bash
KVBM_MODEL_ID=deepseek-ai/DeepSeek-R1-Distill-Llama-8B \
KVBM_CPU_BLOCKS=10000 \
KVBM_MAX_ITERATIONS=100 \
KVBM_MAX_TOKENS=48 \
KVBM_CONCURRENT_REQUESTS="10,25,50" \
KVBM_IFEVAL_PROMPTS="50,120,200" \
pytest -v -m "kvbm" -s
```

## 依赖

- 测试环境的 PATH 中需可执行 `vllm`。
- connector 模块路径必须有效：`kvbm.vllm_integration.connector`。
- NATS 与 etcd 服务（由 `runtime_services` fixture 自动提供）。
- IFEval 并发测试需要 `datasets` 库（已包含在测试依赖中）。
- 对于容器化工作流，请按顶层 `tests/README.md` 的指引构建/运行合适的镜像，然后在容器内执行 pytest。

## 备注

- 预热（warmup）至关重要，可以避免初始化效应影响确定性。
- 如需更快的本地迭代，可减小 `KVBM_MAX_ITERATIONS` 并/或增加各 interval。
- 日志会写入 `tests/conftest.py` 创建的每个测试目录下，包括 LLM 服务器的 stdout/stderr。
- 测试使用由 `KVBM_SERVER_PORT` 定义的固定端口与 LLM 服务器通信。
