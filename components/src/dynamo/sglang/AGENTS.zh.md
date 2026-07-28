# SGLang 组件

Dynamo 的 SGLang 后端（backend）将 SGLang 的推理（inference）引擎
（`sgl.Engine`）以及 diffusion 生成器（`DiffGenerator`）包装在 Dynamo 分布式
运行时之下，负责模型注册、请求路由、指标（metrics）以及解耦
（disaggregated）服务。

## SGLang 向后兼容性

SGLang 处于 1.0 之前的状态，内部 API 经常在版本之间发生迁移或重命名。我们支持
当前版本以及上一版本（N 与 N-1）。模式如下：

1. **只有那些在版本升级时实际出问题的 SGLang import 才走 `_compat.py`。**
   不要预先把所有 `sglang.*` import 都路由到 shim —— 在出现真实兼容性问题之前，
   直接 import 就好。当某次升级真的破坏了 import 时，再把那个具体的符号搬到
   `_compat.py`，并在组件代码里把直接 import 替换为通过 shim。
2. `_compat.py` 使用 try/except ImportError：先走新路径，再 fallback 到旧路径。
3. 当 SGLang 引入了一个旧版本中不存在的新类 / 新函数（例如 `NetworkAddress`）时，
   在 except 分支里加一个最小的 polyfill —— 仅覆盖 Dynamo 实际会调用的接口
   即可。
4. `_compat.py` 中每个 fallback 分支都**必须**有一段注释，标明它支持哪个
   SGLang 版本以及何时可被移除。例如：
   `# Fallback for sglang <= 0.5.10. Remove when min supported version is 0.5.12+`
5. 每当新版本的 SGLang 发布、原本的 N-1 已不在支持窗口内时，从 `_compat.py`
   中删除对应的 fallback 分支与 polyfill。如果 `_compat.py` 退化成纯粹的
   re-export，就把 import 内联回去并删除该文件。

**当遇到新的 SGLang API 不兼容时**：请按既有模式把受影响的 import 加到
`_compat.py`。不要在组件文件里到处散落 try/except 块。也不要用
`sglang.__version__` 做版本判断 —— 探测 import 更可靠，因为 SGLang 的内部布局
并不总是和版本字符串严格对应。

## 入口

`__main__.py` -> `main.py:main()` -> `main.py:worker()`

`worker()` 解析参数、创建分布式运行时、安装优雅关闭逻辑，然后根据 CLI 标志
分派到 10 个 init 函数中的某一个：

```
args.py:parse_args() -> Config(server_args, dynamo_args)

Worker dispatch (main.py:60-132):
  --image-diffusion-worker    -> init_diffusion.init_image_diffusion()
  --video-generation-worker   -> init_diffusion.init_video_diffusion()
  --embedding-worker          -> init_embedding.init_embedding()
  --multimodal-encode-worker  -> init_multimodal.init_multimodal_encode_worker()
  --multimodal-worker         -> init_multimodal.init_multimodal_worker() or _prefill_worker()
  --dllm-algorithm <algo>     -> init_diffusion.init_llm_diffusion()
  (default, prefill mode)     -> init_llm.init_prefill()
  (default, decode/agg mode)  -> init_llm.init_decode()
```

## 配置 / 参数

`args.py:parse_args()` 是主要的解析函数，返回 `Config(server_args, dynamo_args)`。

**两条配置路径：**

1. **LLM worker**（decode、prefill、embedding、multimodal-worker、dllm）：通过
   `ServerArgs.from_cli_args()` 创建完整的 `sglang.srt.server_args.ServerArgs`。
   这会触发模型配置加载、tokenizer 检测等流程。

2. **Diffusion worker**（image、video）：构造一个最小化的
   `types.SimpleNamespace` 桩对象（args.py:350-366），仅包含 `DiffGenerator`
   所需的字段。该桩对象**不包含** `max_running_requests`、
   `dllm_algorithm_config` 或其他与 LLM 相关的字段。访问可能不存在的字段时请
   使用 `getattr()`。

**DynamoConfig** 把 `DynamoRuntimeConfig`（如 `--namespace`、
`--output-modalities`、`--media-output-fs-url` 等通用标志）与 `DynamoSGLangConfig`
（如 `--multimodal-encode-worker`、`--embedding-worker` 等 sglang 专用标志）组合
在一起。

一个关键坑点：`--output-modalities` 全局默认值为 `["text"]`。Image / video
diffusion worker 在它们的 init 函数中会把它覆盖为 `["image"]` / `["video"]`，
以保证向 Rust 侧的注册路径走对。

## Handler 继承层次

```
BaseGenerativeHandler (handler_base.py)
  Abstract base. Has config, publisher, tracing. No engine.
  Subclasses: ImageDiffusionWorkerHandler, VideoGenerationWorkerHandler

  BaseWorkerHandler (handler_base.py)
    Adds sgl.Engine, tokenizer, priority support, engine routes,
    cancellation, bootstrap (disagg), weight update APIs.
    Constructor accepts engine=None for encode-only workers.

    DecodeWorkerHandler (llm/decode_handler.py)
      Aggregated + disaggregated decode. Token/text streaming.
      Logprob passthrough via _build_logprob_kwargs() + _extract_logprobs().

      DiffusionWorkerHandler (llm/diffusion_handler.py)
        LLM diffusion (DLLM). Simplified decode without disagg.

    PrefillWorkerHandler (llm/prefill_handler.py)
      Disaggregated prefill. Yields bootstrap info first, then consumes.

    EmbeddingWorkerHandler (embedding/embedding_handler.py)
      Uses engine.async_encode() instead of async_generate().

    MultimodalWorkerHandler (multimodal/worker_handler.py)
      Multimodal inference. Aggregated or disaggregated paths.
      Has EmbeddingsProcessor for NIXL-transferred image embeddings.

    MultimodalPrefillWorkerHandler (multimodal/worker_handler.py)
      Multimodal prefill phase. Yields bootstrap info.

    MultimodalEncodeWorkerHandler (multimodal/encode_worker_handler.py)
      Front-facing. No engine. Uses MMEncoder from SGLang. Receives
      pre-tokenized requests (ModelInput.Tokens) from Rust frontend,
      encodes images, NIXL for embeddings transfer.
```

## 各 Worker 的引擎类型

| Worker | 引擎 | 备注 |
|--------|--------|-------|
| decode、prefill、dllm、embedding | `sgl.Engine` | 完整 SGLang 推理引擎 |
| multimodal-worker、multimodal-prefill | `sgl.Engine` | 额外加上 EmbeddingsProcessor |
| multimodal-encode-worker | None | SGLang 的 `MMEncoder`，输入为预先 tokenize 好的请求 |
| image-diffusion-worker | `DiffGenerator` | 来自 `sglang.multimodal_gen` |
| video-generation-worker | `DiffGenerator` | 来自 `sglang.multimodal_gen` |

`DiffGenerator.generate()` 返回 `GenerationResult | list[GenerationResult] | None`
（dataclass，**不是** dict）。通过 `result.frames` 访问图像 / 视频帧，通过
`result.samples` 访问原始张量。

## 注册

`register.py` 中有三条路径：

1. **LLM**（`register_model_with_readiness_gate`）：构造带有 bootstrap 信息、
   scheduler 统计、parser 配置的 `ModelRuntimeConfig`，调用 Rust 侧的
   `register_model()`，由其负责从 HuggingFace 下载 `config.json` 与 tokenizer。

2. **Image diffusion**（`register_image_diffusion_model`）：用
   `ModelType.Images` 调用 `register_model()`。Rust 侧对 Images / Videos /
   Tensor 类型会跳过 HF 下载（lib/bindings/python/rust/lib.rs:314），并使用
   `ModelDeploymentCard::with_name_only()`。

3. **Video generation**（`register_video_generation_model`）：与上面相同的快速
   路径，只是使用 `ModelType.Videos`。

## Init 流程（典型 LLM decode）

```
init_decode():
  engine = sgl.Engine(server_args)
  handler = DecodeWorkerHandler(engine, config, publisher, endpoint, shutdown_event)
  handler.register_engine_routes(runtime)  # profiling, weight updates, memory mgmt
  setup_sgl_metrics(engine, config, endpoint)  # Prometheus + KV events via ZMQ
  asyncio.gather(
    endpoint.serve_endpoint(handler.generate, ...),
    register_model_with_readiness_gate(engine, endpoint, ...),
  )
```

## 解耦（Disaggregated）服务

prefill 与 decode worker 通过 bootstrap 机制协调：

1. **Prefill handler** 生成 `bootstrap_room`（一个随机 63 位 ID）
2. Prefill 把 bootstrap 信息（host、port、room）作为第一个响应 yield 出去
3. **Decode handler** 接收到 bootstrap 信息后，把它传给 `engine.async_generate()`
4. SGLang 通过 NIXL/RDMA 在 worker 间传输 KV 缓存（KV cache）

关键函数：`BaseWorkerHandler._get_bootstrap_info()`、
`BaseWorkerHandler._generate_bootstrap_room()`。

## 指标与发布

`publisher.py:DynamoSglangPublisher` 负责管理：
- **调度器（scheduler）指标**：通过 ZMQ 从 SGLang 的调度器接收，并发布到
  Prometheus
- **KV 事件**：每个 DP rank 都有 ZMQ 订阅者，通过 `KvEventPublisher` 转发

只有 leader 节点（node_rank==0）才运行指标循环。非 leader 节点只是等待。

`setup_sgl_metrics()` 返回 `(publisher, metrics_task, metrics_labels)`。

## 优雅关闭

`shutdown.py:install_graceful_shutdown()` 通过 monkey-patch
`loop.add_signal_handler()` 来捕获 SGLang 内部的信号注册并将其延迟执行。在
SIGTERM/SIGINT 时：
1. 从服务发现中注销（停止接受新请求）
2. 等待优雅期，让在途请求自然结束
3. 执行被推迟的 SGLang 信号处理器

## 请求流程

```
Frontend (Rust, lib/llm/)
  -> Preprocessor (tokenizes, builds PreprocessedRequest with token_ids + sampling + stop + output_options)
  -> Dynamo RPC to endpoint (dyn://{namespace}.{component}.{endpoint})
  -> Python handler.generate(request_dict, context)
       handler._build_sampling_params(request) -> SGLang-native params
       handler._build_logprob_kwargs(request) -> {return_logprob, top_logprobs_num, logprob_start_len}
       engine.async_generate(**params, **logprob_kwargs) -> async iterator of dicts
       handler yields {token_ids, text, finish_reason, log_probs, top_logprobs, ...} back to frontend
  -> Frontend postprocesses into OpenAI-compatible response
```

请求格式取决于 `--skip-tokenizer-init`：
- **基于 token**（skip_tokenizer_init=True）：由前端（frontend）做 tokenize。
  请求中包含 `token_ids`、`sampling_options`、`stop_conditions`。Handler 把
  它们映射到 SGLang 参数。
- **基于文本**（skip_tokenizer_init=False）：由 SGLang 做 tokenize。请求是
  OpenAI 的 `ChatCompletionRequest`。仅 `/v1/chat/completions` 可用。

Image / video diffusion handler 会直接收到完整的 OpenAI 格式请求字典（不经过
preprocess），因为前端对 diffusion 请求是透传、不做 tokenize 的。

## Logprobs

`DecodeWorkerHandler` 支持 logprob 透传，与 vLLM 和 TRT-LLM 后端保持一致。它
受预处理请求中的 `output_options` 控制（来自 `lib/llm/src/protocols/common.rs`
中的 Rust `OutputOptions` 结构体）。

**OutputOptions 到 SGLang kwargs 的映射**（`_build_logprob_kwargs`）：

| OutputOptions 字段 | SGLang kwarg | 备注 |
|---------------------|-------------|-------|
| `logprobs: N` | `return_logprob=True, top_logprobs_num=N` | 每个输出 token 的 N 个 top logprob |
| `prompt_logprobs: M` | `return_logprob=True, logprob_start_len=0` | 从 prompt 起始位置开始计算 |
| 同时设置 | `top_logprobs_num=max(N, M)` | SGLang 用同一个 top_logprobs_num 控制两者 |

`logprob_start_len` 是 SGLang 内部参数，未在 OutputOptions 中暴露。它控制
logprob 计算的绝对序列起始位置：`-1`（默认）= 仅输出 tokens（即
`len(prompt) - 1`），`0` = 从 prompt 起始。当请求设置了 `prompt_logprobs` 时，
我们将其设为 0。

**Top-logprobs 闸门**：默认下 `logprobs >= 1`（或 `prompt_logprobs >= 1`）会
抛出 `ValueError`。SGLang 的 tokenizer manager 会按位置串行 detokenize top-k
tokens，这会带来严重的延迟劣化（每个生成 token 上 O(N)）。调用方必须使用
`logprobs=0` 来仅获取被选 token 的 logprob。一旦上游对
`detokenize_top_logprobs_tokens` 实现了批处理，可以通过设置
`DYN_SGL_ALLOW_TOP_LOGPROBS=1` 解除限制。

**流式行为**（`_extract_logprobs`）：

Dynamo 强制 `stream_output=True`（args.py:374），使每次 chunk 的 `output_ids`
互不重叠。然而，SGLang 的 `meta_info["output_token_logprobs"]` 与
`meta_info["output_top_logprobs"]` 始终是**累计的** —— 它们随着每个 chunk
不断增长。Handler 通过 `num_output_logprobs_so_far` 跟踪进度，从中切出每次
chunk 的新增条目。

SGLang logprob 的格式：`(logprob, token_id, text_or_None)` 元组。
Dynamo 输出格式：`log_probs` = float 列表，`top_logprobs` = list of lists of
`{rank, token_id, token, logprob}` 字典（与 vLLM/TRT-LLM 一致）。

## 健康检查

每种 worker 类型都有自定义的健康检查 payload（`health_check.py`）：
- **Decode/Aggregated**：`SglangHealthCheckPayload` —— 发送 BOS token，期望返回 1 个 token
- **Prefill（disagg）**：`SglangPrefillHealthCheckPayload` —— 包装为 `{request, sampling_params}`
- **Image diffusion**：`ImageDiffusionHealthCheckPayload` —— 512x512、1 次推理步、b64_json
- **Video generation**：`VideoGenerationHealthCheckPayload` —— 256x256、8 帧、1 步、b64_json

可通过环境变量 `DYNAMO_HEALTH_CHECK_PAYLOAD`（JSON）覆盖健康检查 payload。

## 启动脚本

示例位于 `examples/backends/sglang/launch/`。每个脚本会在同一个终端中启动一个
前端 + 若干 worker。GPU 需求在脚本头部有说明。

```
agg.sh              # 1 GPU  - Single aggregated worker
agg_embed.sh        # 1 GPU  - Embedding model
agg_vision.sh       # 1 GPU  - Multimodal (vision + LLM)
agg_router.sh       # 2 GPUs - Two workers behind KV-aware router
disagg.sh           # 2 GPUs - Prefill + decode on separate GPUs
disagg_router.sh    # 4 GPUs - 2 prefill + 2 decode with KV routing
disagg_same_gpu.sh  # 1 GPU  - Both workers on single GPU (16+ GB VRAM)
multimodal_epd.sh   # 2 GPUs - Encoder + PD worker
multimodal_disagg.sh # 3 GPUs - Encoder + prefill + decode
diffusion_llada.sh  # 1 GPU  - Diffusion language model
image_diffusion.sh  # 1 GPU  - Text-to-image (~38 GB VRAM for FLUX.1-dev)
text-to-video-diffusion.sh  # 1-2 GPUs - Text-to-video (Wan2.1)
```

## 常见坑点

- **SimpleNamespace vs ServerArgs**：image / video diffusion worker 使用
  SimpleNamespace 桩对象。访问可能不存在的字段时，永远使用
  `getattr(server_args, field, default)`。
- **engine=None**：multimodal encode worker 会把 `engine=None` 传给
  BaseWorkerHandler。基类中任何会触碰 engine 的代码都必须用
  `if engine is not None` 守护。
- **GenerationResult 是 dataclass**：SGLang 的 `DiffGenerator.generate()`
  返回 `GenerationResult`（不是 dict）。请使用 `result.frames`，不要用
  `result["frames"]`。
- **output_modalities 默认值**：全局默认是 `["text"]`。image / video
  diffusion worker 必须把它覆盖为 `["image"]` / `["video"]`，否则 Rust 注册
  路径会尝试加载 `config.json`（diffusers 模型并不存在该文件）。
- **流式中累计的 logprobs**：尽管 `stream_output=True` 时 `output_ids` 是互不
  重叠的，SGLang 的 `meta_info` 中 `output_token_logprobs` /
  `output_top_logprobs` 仍然是累计的。一定要带偏移做切片，不要假定每个 chunk
  的 logprob 是独立的。
- **僵尸 GPU 进程**：`sgl_diffusion::scheduler` 会 spawn 一个会在父进程被 kill
  之后仍存活的子进程。teardown 后请用 `nvidia-smi` 检查。
- **会话控制（session control）的优雅降级**：会话控制是请求驱动的 —— 路由器
  中的 `AgentController` 与 `StickySessionRouter` 总是被创建，但是惰性激活的。
  如果没有任何 worker 启用 `--enable-streaming-session`，路由器会发出一次警告
  并忽略请求中的 `session_control`。Handler 侧，`_session_kwargs()` 会先检查
  `enable_streaming_session`，再决定是否向 SGLang 调用注入 `session_params`。
  两层必须保持一致：路由器会跳过生命周期 RPC，handler 会跳过 session 参数。
  如果两层守护没有同时存在，SGLang 会报错 "session id does not exist"。

故障排查（CuDNN、config.json 报错、OOM、disagg 连接性）请见
`docs/backends/sglang/sglang-examples.md#troubleshooting`。

## 添加一种新的 Worker 类型

新增 worker（如新模态或新服务模式）的清单：

1. **CLI 标志**：在 `backend_args.py`（DynamoSGLangConfig）中新增标志，并在
   `args.py` 中解析
2. **Init 函数**：创建 `init_<type>.py`，其中的 `init_<type>(config, runtime)`
   要：
   - 创建引擎（sgl.Engine、DiffGenerator，或对仅编码 worker 为 None）
   - 创建 handler
   - 设置指标（如适用，调用 `setup_sgl_metrics`）
   - 调用 `endpoint.serve_endpoint(handler.generate, ...)`
   - 注册模型
3. **Handler**：继承 `BaseWorkerHandler`（如有 engine）或
   `BaseGenerativeHandler`（如无 engine）。实现
   `async generate(request, context) -> AsyncGenerator`
4. **注册**：在 `register.py` 中新增对应函数。选择正确的 `ModelType`：
   - `Chat | Completions` 表示 LLM（Rust 会下载 config.json + tokenizer）
   - `Images`、`Videos`、`Tensor` 表示非 LLM（Rust 跳过 HF 下载）
5. **健康检查**：在 `health_check.py` 中新增一个 payload 类
6. **派发**：在 `main.py:worker()` 派发块中新增对该标志的检查
7. **output_modalities**：如果不是 text，请在 init 函数中覆盖（默认是 `["text"]`）
8. **启动脚本**：在 `examples/backends/sglang/launch/` 中新增脚本，在头部
   注明 GPU 数量

## 给 AI 助手的小提示

- **修改前先读源码**：在改动 handler 或注册代码之前，请先阅读 handler_base.py
  以及对应的 init_*.py。继承链很重要。
- **用启动脚本测试**：验证改动最快的方式就是运行
  `examples/backends/sglang/launch/` 下对应的启动脚本。
- **测试间清理僵尸进程**：重启之前先 `pkill -9 -f sglang; sleep 3`。
  diffusion worker 会 spawn 一个能在 kill 之后存活的子进程
  （`sgl_diffusion::scheduler`）。
- **检查 nvidia-smi**：如果启动 OOM，检查是否有上次运行残留的孤儿 GPU 进程。
- **SimpleNamespace 桩对象**：每当改动 args.py 或读取 server_args 的代码时，
  始终使用 `getattr(server_args, field, default)` —— image / video worker
  并没有完整的 ServerArgs。
- **engine 可能为 None**：仅编码 worker（multimodal-encode-worker）会传
  `engine=None`。在共享基类中访问 engine 时务必加守护。
- **修改 Rust 后重新构建**：如果你改动了注册逻辑（register.py 与 Rust binding
  交互），请重新构建：
  `cd lib/bindings/python && maturin develop --uv && cd <root> && uv pip install -e .`
- **故障排查**：CuDNN、config.json、OOM 与 disagg 连接性问题请见
  `docs/backends/sglang/sglang-examples.md#troubleshooting`。

## 文件索引

```
sglang/
  _compat.py               # SGLang version compat shim (signature probing for async_generate kwargs)
  __main__.py              # Entry point
  main.py                  # Worker dispatch
  args.py                  # Config parsing (ServerArgs vs SimpleNamespace)
  backend_args.py          # Dynamo-specific SGLang CLI flags
  init_llm.py              # init_decode(), init_prefill()
  init_diffusion.py        # init_llm_diffusion(), init_image_diffusion(), init_video_diffusion()
  init_multimodal.py       # init_multimodal_{encode_worker,worker,prefill_worker}()
  init_embedding.py        # init_embedding()
  register.py              # Model registration (LLM, image, video)
  publisher.py             # Metrics + KV event publishing
  protocol.py              # Request/response Pydantic models
  health_check.py          # Health check payloads per worker type
  shutdown.py              # Graceful shutdown with deferred signal handling
  request_handlers/
    handler_base.py        # BaseGenerativeHandler, BaseWorkerHandler
    llm/
      decode_handler.py    # DecodeWorkerHandler (agg + disagg)
      prefill_handler.py   # PrefillWorkerHandler (disagg prefill)
      diffusion_handler.py # DiffusionWorkerHandler (DLLM)
    embedding/
      embedding_handler.py # EmbeddingWorkerHandler
    image_diffusion/
      image_diffusion_handler.py  # ImageDiffusionWorkerHandler (DiffGenerator)
    video_generation/
      video_generation_handler.py # VideoGenerationWorkerHandler (DiffGenerator)
    multimodal/
      encode_worker_handler.py   # MultimodalEncodeWorkerHandler (MMEncoder, front-facing)
      worker_handler.py          # MultimodalWorkerHandler + PrefillWorkerHandler
```
