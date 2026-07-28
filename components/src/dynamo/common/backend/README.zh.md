# Dynamo Python 后端（Backend）

> **进行中。** 当前统一后端仅支持极简的聚合（aggregated）推理（inference）。剩余待实现项参见底部 [Feature Gaps](#feature-gaps)。

一种两类抽象，将**运行时集成**（所有后端共有）与**引擎逻辑**（vLLM、SGLang、TensorRT-LLM 等）分离。

## 架构

```
LLMEngine (ABC)                <-- engine boundary (engine.py)
    |   - from_args(argv) -> (LLMEngine, WorkerConfig)  (factory)
    |   - start() -> EngineConfig        (start engine, return metadata)
    |   - generate(request, context)    (streaming inference)
    |   - abort(context)                (cancel request, optional)
    |   - cleanup()                     (shutdown)
    |
    +-- VllmLLMEngine          <-- vllm/llm_engine.py
    +-- SglangLLMEngine        <-- sglang/llm_engine.py
    +-- TrtllmLLMEngine        <-- trtllm/llm_engine.py
    +-- SampleLLMEngine        <-- sample_engine.py

Worker                  <-- runtime integration (worker.py)
    - receives WorkerConfig from from_args()
    - creates DistributedRuntime
    - sets up endpoints, signal handlers
    - calls engine.start(), registers model
    - serves generate endpoint with cancellation monitoring
    - calls engine.cleanup() on shutdown
```

## 快速开始

### 运行示例引擎

```bash
python -m dynamo.common.backend.sample_main \
    --model-name test-model \
    --namespace dynamo \
    --component sample \
    --endpoint generate
```

该后端会生成轮转的 token ID。将前端（frontend）指向 `dynamo.sample.generate`，可在不依赖任何 ML 依赖的情况下测试完整请求流。

### 运行真实引擎

```bash
# vLLM
python -m dynamo.vllm.unified_main --model Qwen/Qwen3-0.6B ...

# SGLang
python -m dynamo.sglang.unified_main --model-path Qwen/Qwen3-0.6B ...

# TensorRT-LLM
python -m dynamo.trtllm.unified_main --model Qwen/Qwen3-0.6B ...
```

每个 `unified_main.py` 都从公共 `run.py` 模块调用 `run(MyLLMEngine)`。

## 实现新引擎

继承 `LLMEngine` 并实现必需方法：

```python
from dynamo.common.backend import LLMEngine, EngineConfig, WorkerConfig

class MyEngine(LLMEngine):
    @classmethod
    async def from_args(cls, argv=None):
        # Parse CLI args, construct engine and worker_config.
        engine = cls(...)
        worker_config = WorkerConfig(
            namespace="dynamo", component="my-backend", ...
        )
        return engine, worker_config

    async def start(self) -> EngineConfig:
        # Start the engine, return metadata for model registration.
        # After this returns, generate() MUST be ready to accept calls.
        return EngineConfig(
            model="my-model",
            context_length=4096,
            kv_cache_block_size=16,
        )

    async def generate(self, request, context):
        # Yield streaming response dicts.
        async for result in my_engine.run(request):
            yield {"token_ids": result.token_ids, "index": 0}
        yield {
            "token_ids": result.token_ids,
            "index": 0,
            "finish_reason": "stop",
            "completion_usage": {
                "prompt_tokens": prompt_tokens,
                "completion_tokens": completion_tokens,
                "total_tokens": prompt_tokens + completion_tokens,
            },
        }

    async def abort(self, context):
        # Cancel an in-flight request (optional, default is no-op).
        await my_engine.cancel(context.id())

    async def cleanup(self):
        # Shut down the engine.
        pass
```

然后创建一个入口：

```python
# my_backend/unified_main.py
from dynamo.common.backend.run import run
from my_backend.llm_engine import MyEngine

def main():
    run(MyEngine)
```

完整可运行的参考实现见 `sample_engine.py`。

## 请求 / 响应类型

`GenerateRequest` 与 `GenerateChunk`（定义于 `engine.py`）是 `TypedDict`，用于描述所有引擎的共享字段。

```python
class GenerateRequest(TypedDict, total=False):
    token_ids: Required[list[int]]
    sampling_options: dict[str, Any]
    stop_conditions: dict[str, Any]
    output_options: dict[str, Any]

class GenerateChunk(TypedDict, total=False):
    token_ids: Required[list[int]]
    index: Required[int]           # choice index; use 0 for single-choice chunks
    finish_reason: str             # final chunk only
    completion_usage: dict[str, int]  # final chunk only
```

引擎可从请求字典中读取额外的后端特定键；如要在响应块中写入后端特定键，应先在此处扩展共享契约。

直接内联构建 `completion_usage` 字典。`finish_reason` 的归一化（如 `"abort"` → `"cancelled"`）由 Rust 层处理。

## 请求取消（Request Cancellation）

`Worker.generate()` 会通过 `context.async_killed_or_stopped()` 自动监控客户端断连与请求取消。被触发时，它会：

1. 调用 `engine.abort(context)` 释放引擎资源（KV 缓存、调度器（scheduler）槽位等）
2. 退出生成循环
3. 清理监控任务

引擎实现应覆盖 `abort(context)` 完成后端特定清理：

| 引擎 | abort 方法 | 使用的 ID |
|--------|-------------|---------|
| vLLM | `engine_client.abort(request_id)` | `context.id()` |
| SGLang | `tokenizer_manager.abort_request(rid=...)` | `context.trace_id` |
| TRT-LLM | `generation_result.abort()` | 通过 `context.id()` 按请求跟踪 |
| Sample | *(默认 no-op)* | — |

不支持取消的引擎可不覆盖 `abort()` — 默认实现为 no-op。生成循环仍会在 `context.is_stopped()` 时退出。

## 错误处理

`Worker` 会将错误包装为 `dynamo.llm.exceptions` 中的 `DynamoException` 子类，便于 Rust 桥接层将其映射为带完整错误链的、强类型的 `DynamoError::Backend(...)` 响应。

| 阶段 | 抛出的异常 | 触发场景 |
|-------|-----------------|------|
| 运行时创建 | `CannotConnect` | etcd/NATS 不可达 |
| 引擎初始化 | `EngineShutdown` | 引擎启动失败（OOM、配置错误等） |
| Generate | `Unknown` | 引擎 `generate()` 抛出未类型化异常 |
| Generate | *(直通)* | 引擎直接抛出 `DynamoException` 子类 |

引擎实现可在 `generate()` 中直接抛出 `DynamoException` 子类以做精细化错误上报 — 这些异常会原样透传。任何非 `DynamoException` 错误会被包装为 `Unknown`。

可用异常类型（来自 `dynamo.llm.exceptions`）：

```python
from dynamo.llm.exceptions import (
    DynamoException,     # Base class
    Unknown,             # Uncategorized error
    InvalidArgument,     # Bad input (e.g., prompt too long)
    CannotConnect,       # Connection failed
    Disconnected,        # Connection lost
    ConnectionTimeout,   # Timeout
    Cancelled,           # Client cancelled
    EngineShutdown,      # Engine crashed or shutting down
    StreamIncomplete,    # Response stream cut short
)
```

## 文件索引

```
common/backend/
    __init__.py          # Re-exports: LLMEngine, EngineConfig,
                         #   Worker, WorkerConfig
    engine.py            # LLMEngine ABC + EngineConfig dataclass
    worker.py            # Worker + WorkerConfig
    run.py               # Common entry point: run(engine_cls)
    sample_engine.py     # SampleLLMEngine (reference impl)
    sample_main.py       # Entry point for sample engine

vllm/llm_engine.py       # VllmLLMEngine
vllm/unified_main.py     # Entry point -> run(VllmLLMEngine)

sglang/llm_engine.py     # SglangLLMEngine
sglang/unified_main.py   # Entry point -> run(SglangLLMEngine)

trtllm/llm_engine.py     # TrtllmLLMEngine
trtllm/unified_main.py   # Entry point -> run(TrtllmLLMEngine)
```

## Feature Gaps

统一路径目前仅支持**极简的聚合推理**。下面汇总现有（非统一）后端已具备但统一路径尚未支持的能力。

### 已可工作功能

- 基础聚合 token-in-token-out 推理（三种引擎均支持）
- 带端点类型的模型注册
- 通过 `abort()` + `context.is_stopped()` 监控的请求取消
- `DynamoException` 错误链包装
- 带信号处理的优雅关闭（graceful shutdown）
- 由 Rust 层完成 finish_reason 归一化

### 通用 Gap（所有引擎）

| 功能 | 说明 |
|---------|-------------|
| 解耦（Disaggregated）服务 | Prefill/decode worker 拆分、bootstrap 协调、KV 传输 |
| 指标（Metrics）与 Prometheus | 引擎级指标、KV 缓存利用率 gauge、Prometheus 多进程注册表 |
| KV 事件发布 | 通过 ZMQ/NATS 向路由（router）发布前缀缓存事件（BlockStored/Removed） |
| 健康检查负载 | 各引擎自定义健康检查负载（BOS token 探针等） |
| Logprobs | 选定 token + top-k 对数概率提取与流式传输 |
| 受控解码 / 结构化输出 | JSON schema、正则、文法、choice 约束 |
| OpenTelemetry 跟踪 | `build_trace_headers()`、请求性能指标、OTEL 传播 |
| 引擎路由（routes） | 性能分析（profiling，启停）、内存释放/恢复、权重更新（disk/tensor/distributed/IPC） |
| 数据并行（DP）路由 | 从路由 hint 中提取 DP rank、DP 感知调度 |
| Text-in-text-out 模式 | 兼容 OpenAI 的 chat/completion，引擎侧分词 |
| 自定义 Jinja 聊天模板 | `--custom-jinja-template` 用于按模型定制 prompt 格式 |
| Snapshot/checkpoint | 基于 CRIU 的引擎状态保存/恢复、身份重载 |

### vLLM 专属 Gap

| 功能 | 说明 |
|---------|-------------|
| LoRA 适配器 | 动态 load/unload/list、ModelDeploymentCard 发布、按 LoRA 序列化锁 |
| 多模态（图像/视频） | 图像/视频加载、嵌入缓存、NIXL RDMA 传输、Qwen VL mRoPE |
| 独立 encode worker | 用于多模态仅编码解耦的 `EncodeWorkerHandler` |
| Sleep/wake/quiesce | 三级引擎生命周期控制（权重、缓冲、全部） |
| 弹性 EP 扩缩 | `scale_elastic_ep` 与 Ray 节点管理 |
| GMS shadow 模式 | 带故障切换锁的 GPU Memory Service 集成 |
| ModelExpress P2P | 通过 P2P 进行分布式模型加载 |
| KV 块清空 | 前缀缓存重置端点 |

### SGLang 专属 Gap

| 功能 | 说明 |
|---------|-------------|
| Embedding 推理 | `async_encode()` 路径、OpenAI embedding 响应格式 |
| 图像扩散 | `DiffGenerator` 用于 text-to-image（FLUX 等），支持 TP/DP |
| 视频生成 | `DiffGenerator` 用于 text-to-video（Wan2.1 等） |
| LLM 扩散（DLLM） | 扩散语言模型算法支持 |
| 多模态 encode worker | 前置 `MMEncoder`、嵌入 LRU 缓存、NIXL 传输 |
| 多模态 worker | 聚合/解耦多模态推理，含 `EmbeddingsProcessor` |
| 延迟信号处理 | 捕获 SGLang 内部信号注册以协调关闭 |
| 输出模态覆盖 | 扩散 worker 必需（默认 `["text"]` -> `["image"]`/`["video"]`） |

### TRT-LLM 专属 Gap

| 功能 | 说明 |
|---------|-------------|
| 自定义 logits 处理器 | 支持 CUDA stream 的 `TrtllmDynamoLogitsAdapter` |
| 注意力 DP 调度 | 含 `attention_dp_rank` 与 `attention_dp_relax` 的 `SchedulingParams` |
| 视频扩散 | 自动从 `model_index.json` 检测 pipeline，MP4 编码、MediaOutput |
| 多模态处理 | `MultimodalRequestProcessor`、图像 URL 处理、嵌入注入 |
| Encode 助手（EPD） | 通过 `encode_client` 远端编码、NIXL 张量读取 |
| KV 缓存 connector | KVBM connector 配置、整合器 ZMQ 集成 |
| 致命错误 vs 单请求错误 | 区分 `RequestError`（可恢复）与致命引擎错误 |

### 推荐迁移顺序

1. **指标与健康检查** — 生产可观测性（observability）必备
2. **解耦服务** — 最大的架构变更，解锁 PD 拆分
3. **KV 事件发布** — KV 感知路由所需
4. **Logprobs + 受控解码** — 最受欢迎的推理特性
5. **多模态 / LoRA / 扩散** — 模态相关，可在多负责人间并行推进
