# SGLang 组件

Dynamo 的 SGLang 后端（backend）将 SGLang 推理（inference）引擎（`sgl.Engine`）与扩散
生成器（`DiffGenerator`）封装在 Dynamo 的分布式运行时（runtime）之后。它负责模型
注册、请求路由（router）、指标（metrics）以及解耦（disaggregated）服务。

## SGLang 向后兼容

SGLang 处于 1.0 之前阶段，会在不同发行之间频繁移动/重命名内部 API。我们
支持当前版本以及上一版本（N 与 N-1）。模式如下：

1. **只有真正在版本升级中失效的 SGLang 引用才走 `_compat.py`。** 不要把每个
   `sglang.*` 引用都预先路由到 shim —— 在出现真正失败之前直接 import。
   当某次升级让 import 失败时，再把那个具体符号搬到 `_compat.py`，并把组件代码
   中的直接 import 替换为 shim。
2. `_compat.py` 使用 try/except ImportError：先新路径，再回退老路径。
3. 当 SGLang 引入老版本不存在的新类/函数（例如 `NetworkAddress`）时，在 except 分支
   中加上一个最小 polyfill —— 仅覆盖 Dynamo 实际调用的接口面。
4. `_compat.py` 中的每个回退分支都**必须**有注释，说明它支持哪个 SGLang
   版本，以及何时可以移除，例如：
   `# Fallback for sglang <= 0.5.10. Remove when min supported version is 0.5.12+`
5. 当新的 SGLang 版本发布、原本的 N-1 落出支持窗口时，从 `_compat.py` 删除对应
   回退分支与 polyfill。如果 `_compat.py` 只剩平凡 re-export，请把 import 内联回去
   并删掉该文件。

**当你遇到新的 SGLang API 破坏**：按既有模式把受影响的 import 加进
`_compat.py`。不要在组件文件中到处散布 try/except 块。也不要靠
`sglang.__version__` 做版本判断 —— 用 import 探测更可靠，因为 SGLang 内部布局
与版本字符串并不总是吻合。

## 入口

`__main__.py` -> `main.py:main()` -> `main.py:worker()`

`worker()` 解析参数、创建分布式运行时、安装优雅停机，然后基于 CLI flag
分发到 10 个 init 函数之一：

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

## Config / Args

`args.py:parse_args()` 是主解析函数。返回 `Config(server_args, dynamo_args)`。

**两条配置路径：**

1. **LLM worker**（decode、prefill、embedding、multimodal-worker、dllm）：通过
   `ServerArgs.from_cli_args()` 创建完整的 `sglang.srt.server_args.ServerArgs`。这会触发
   模型配置加载、tokenizer 探测等。

2. **扩散 worker**（image、video）：创建一个最小化的 `types.SimpleNamespace` 桩
   （args.py:350-366），只带 `DiffGenerator` 需要的字段。该桩**不**包含
   `max_running_requests`、`dllm_algorithm_config` 或其他 LLM 专用字段。
   访问可能不在桩上的字段时使用 `getattr()`。

**DynamoConfig** 把 `DynamoRuntimeConfig`（如 `--namespace`、
`--output-modalities`、`--media-output-fs-url` 等通用 flag）与 `DynamoSGLangConfig`（如
`--multimodal-encode-worker`、`--embedding-worker` 等 sglang 专用 flag）组合到一起。

关键陷阱：`--output-modalities` 全局默认 `["text"]`。Image/video 扩散 worker 会在自己的
init 函数里把它覆盖为 `["image"]`/`["video"]`，确保在 Rust 侧能被正确注册。

## Handler 继承层级

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
| multimodal-worker、multimodal-prefill | `sgl.Engine` | 加上 EmbeddingsProcessor |
| multimodal-encode-worker | None | 来自 SGLang 的 `MMEncoder`，输入已预 tokenize |
| image-diffusion-worker | `DiffGenerator` | 来自 `sglang.multimodal_gen` |
| video-generation-worker | `DiffGenerator` | 来自 `sglang.multimodal_gen` |

`DiffGenerator.generate()` 返回 `GenerationResult | list[GenerationResult] | None`
（dataclass，**不是** dict）。访问 `result.frames` 拿图像/视频帧，
`result.samples` 拿原始张量。

## 注册

`register.py` 有三条路径：

1. **LLM**（`register_model_with_readiness_gate`）：构建带 bootstrap 信息、
   scheduler stats、parser 配置的 `ModelRuntimeConfig`。调用 Rust `register_model()`，
   该函数会从 HuggingFace 下载 `config.json` 与 tokenizer。

2. **图像扩散**（`register_image_diffusion_model`）：以 `ModelType.Images` 调用
   `register_model()`。Rust 侧对 Images/Videos/Tensor 类型跳过 HF 下载
   （lib/bindings/python/rust/lib.rs:314），并使用 `ModelDeploymentCard::with_name_only()`。

3. **视频生成**（`register_video_generation_model`）：以 `ModelType.Videos` 走同样的快路径。

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

## 解耦服务

prefill 与 decode worker 通过 bootstrap 机制协作：

1. **Prefill handler** 生成一个 `bootstrap_room`（随机 63 位 ID）
2. Prefill 把 bootstrap 信息（host、port、room）作为第一条响应 yield 出去
3. **Decode handler** 收到 bootstrap 信息，传给 `engine.async_generate()`
4. SGLang 通过 NIXL/RDMA 在 worker 之间传输 KV 缓存（KV cache）

关键函数：`BaseWorkerHandler._get_bootstrap_info()`、
`BaseWorkerHandler._generate_bootstrap_room()`。

## 指标与发布

`publisher.py:DynamoSglangPublisher` 管理：
- **scheduler 指标**：通过 ZMQ 从 SGLang 的 scheduler 接收，发布到 Prometheus
- **KV 事件**：每个 DP rank 一个 ZMQ 订阅者，通过 `KvEventPublisher` 转发

只有 leader 节点（node_rank==0）跑指标循环。非 leader 节点只是等待。

`setup_sgl_metrics()` 返回 `(publisher, metrics_task, metrics_labels)`。

## 优雅停机

`shutdown.py:install_graceful_shutdown()` 通过 monkey-patch
`loop.add_signal_handler()` 捕获 SGLang 内部信号注册并将其延迟。在 SIGTERM/SIGINT 时：
1. 从 discovery 注销（停止接收新请求）
2. 等待宽限期完成进行中的请求
3. 运行被延迟的 SGLang 信号处理器

## 请求流

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

依据 `--skip-tokenizer-init` 有两种请求格式：
- **基于 token**（skip_tokenizer_init=True）：frontend 做 tokenize。请求包含
  `token_ids`、`sampling_options`、`stop_conditions`。handler 映射为 SGLang 参数。
- **基于文本**（skip_tokenizer_init=False）：SGLang 做 tokenize。请求是 OpenAI
  `ChatCompletionRequest`。仅可用 `/v1/chat/completions`。

图像/视频扩散 handler 直接接收完整的 OpenAI 格式请求 dict（未预处理），
因为 frontend 对扩散请求不做 tokenize 直接透传。

## Logprobs

`DecodeWorkerHandler` 支持 logprob 透传，与 vLLM 和 TRT-LLM 后端保持一致。
由预处理请求中的 `output_options` 控制（来自 `lib/llm/src/protocols/common.rs` 中 Rust 的 `OutputOptions`
结构体）。

**OutputOptions 到 SGLang kwargs 的映射**（`_build_logprob_kwargs`）：

| OutputOptions 字段 | SGLang kwarg | 备注 |
|---------------------|-------------|-------|
| `logprobs: N` | `return_logprob=True, top_logprobs_num=N` | 每个输出 token 的 N 个 top logprob |
| `prompt_logprobs: M` | `return_logprob=True, logprob_start_len=0` | 从 prompt 起始计算 |
| 同时设置 | `top_logprobs_num=max(N, M)` | SGLang 两边共享一个 top_logprobs_num |

`logprob_start_len` 是 SGLang 内部字段，未在 OutputOptions 中暴露。它控制开始计算
logprob 的绝对序列位置：`-1`（默认）= 仅输出 token（`len(prompt) - 1`），`0` = 从 prompt 起始。
当请求 `prompt_logprobs` 时我们将其设为 0。

**Top-logprobs 门控**：`logprobs >= 1`（或 `prompt_logprobs >= 1`）默认会抛
`ValueError`。SGLang 的 tokenizer manager 会按位置串行 detokenize top-k token，
导致严重延迟劣化（每生成 token O(N)）。调用方必须使用
`logprobs=0` 仅获取所选 token 的 logprob。在上游对
`detokenize_top_logprobs_tokens` 实现批处理之后，可设置 `DYN_SGL_ALLOW_TOP_LOGPROBS=1` 覆盖。

**流式行为**（`_extract_logprobs`）：

Dynamo 强制 `stream_output=True`（args.py:374），让每个 chunk 的 `output_ids` 不重叠。
然而 SGLang 的 `meta_info["output_token_logprobs"]` 与 `meta_info["output_top_logprobs"]`
始终是 **累积的** —— 每个 chunk 都在增长。handler 跟踪
`num_output_logprobs_so_far`，按 chunk 切出新增条目。

SGLang logprob 格式：`(logprob, token_id, text_or_None)` 元组。
Dynamo 输出格式：`log_probs` = float 列表，`top_logprobs` = 由
`{rank, token_id, token, logprob}` dict 组成的列表的列表（与 vLLM/TRT-LLM 相同）。

## 健康检查

每种 worker 类型都有自定义健康检查负载（`health_check.py`）：
- **Decode/Aggregated**：`SglangHealthCheckPayload` —— 发送 BOS token，期望返回 1 个 token
- **Prefill（disagg）**：`SglangPrefillHealthCheckPayload` —— 包装好的 `{request, sampling_params}`
- **Image diffusion**：`ImageDiffusionHealthCheckPayload` —— 512x512，1 步推理，b64_json
- **Video generation**：`VideoGenerationHealthCheckPayload` —— 256x256，8 帧，1 步，b64_json

健康检查负载可通过 `DYNAMO_HEALTH_CHECK_PAYLOAD` 环境变量（JSON）覆盖。

## 启动脚本

示例位于 `examples/backends/sglang/launch/`。每个脚本在一个终端中启动 frontend +
worker。GPU 需求记录在脚本头部。

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

## 常见陷阱

- **SimpleNamespace vs ServerArgs**：图像/视频扩散 worker 使用 SimpleNamespace 桩。
  对可能不存在的字段始终使用 `getattr(server_args, field, default)`。
- **engine=None**：multimodal-encode worker 向 BaseWorkerHandler 传入 `engine=None`。
  基类中触及 engine 的代码必须以 `if engine is not None` 守护。
- **GenerationResult 是 dataclass**：SGLang 的 `DiffGenerator.generate()` 返回
  `GenerationResult`（不是 dict）。使用 `result.frames`，不要用 `result["frames"]`。
- **output_modalities 默认值**：全局默认是 `["text"]`。图像/视频扩散 worker 必须
  覆盖为 `["image"]`/`["video"]`，否则 Rust 注册路径会尝试加载 `config.json`
  （diffusers 模型并没有该文件）。
- **流式中累积的 logprob**：尽管 `output_ids` 是不重叠的（stream_output=True），
  SGLang 在 `meta_info` 中的 `output_token_logprobs`/`output_top_logprobs` 仍是累积的。
  始终用偏移做切片，不要假设 logprob 是按 chunk 给出的。
- **僵尸 GPU 进程**：`sgl_diffusion::scheduler` 会派生一个在父进程被 kill 后仍存活的
  子进程。teardown 后请始终 `nvidia-smi` 检查。
- **Session 控制的优雅降级**：session 控制是请求驱动的 ——
  router 的 `AgentController` 与 `StickySessionRouter` 始终被创建，但延迟激活。
  如果没有 worker 启用 `--enable-streaming-session`，router 会一次性告警，
  并忽略请求里的 `session_control`。在 handler 侧，
  `_session_kwargs()` 会在向 SGLang 调用注入 `session_params` 之前检查
  `enable_streaming_session`。两层必须一致：router 跳过生命周期 RPC，
  handler 跳过 session 参数。少了任何一层守护，SGLang 都会以 "session id does not exist" 报错。

故障排查（CuDNN、config.json 错误、OOM、disagg 连通性）参见
`docs/backends/sglang/sglang-examples.md#troubleshooting`。

## 添加新的 Worker 类型

添加新 worker（如新模态或新服务模式）的清单：

1. **CLI flag**：在 `backend_args.py`（DynamoSGLangConfig）中添加，并在 `args.py` 中解析
2. **Init 函数**：创建 `init_<type>.py`，其中 `init_<type>(config, runtime)`：
   - 创建引擎（sgl.Engine、DiffGenerator，或仅 encode 时的 None）
   - 创建 handler
   - 设置指标（如适用 `setup_sgl_metrics`）
   - 调用 `endpoint.serve_endpoint(handler.generate, ...)`
   - 注册模型
3. **Handler**：继承 `BaseWorkerHandler`（带 engine）或 `BaseGenerativeHandler`
   （无 engine）。实现 `async generate(request, context) -> AsyncGenerator`
4. **注册**：在 `register.py` 加一个函数。选择正确的 `ModelType`：
   - `Chat | Completions` 用于 LLM（Rust 下载 config.json + tokenizer）
   - `Images`、`Videos`、`Tensor` 用于非 LLM（Rust 跳过 HF 下载）
5. **健康检查**：在 `health_check.py` 添加一个 payload 类
6. **分发**：在 `main.py:worker()` 分发块中加上 flag 检查
7. **output_modalities**：如非文本，在 init 函数中覆盖（默认 `["text"]`）
8. **启动脚本**：在 `examples/backends/sglang/launch/` 添加，脚本头标注 GPU 数量

## 给 AI 助手的提示

- **修改前先读**：在改动 handler 或 registration 代码前，始终先读 handler_base.py
  与对应的 init_*.py。继承链很重要。
- **用启动脚本测**：验证改动最快的方式是运行 `examples/backends/sglang/launch/`
  下对应的启动脚本。
- **测试间杀僵尸**：重启之前 `pkill -9 -f sglang; sleep 3`。
  扩散 worker 会派生子进程（`sgl_diffusion::scheduler`），它能在 kill 之后存活。
- **检查 nvidia-smi**：如果启动 OOM，检查是否有上一次运行残留的孤立 GPU 进程。
- **SimpleNamespace 桩**：在改动 args.py 或读取 server_args 的代码时，始终
  使用 `getattr(server_args, field, default)` —— 图像/视频 worker 没有完整 ServerArgs。
- **engine 可能为 None**：仅 encode 的 worker（multimodal-encode-worker）传入
  engine=None。共享基类代码中任何 engine 访问都要守护。
- **Rust 改动后重建**：如果改动了 registration（register.py 与 Rust 绑定交互），
  重建：`cd lib/bindings/python && maturin develop --uv && cd <root> && uv pip install -e .`
- **故障排查**：CuDNN、config.json、OOM、disagg 连通性等问题参见
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
