# Frontend Chat-Processor 测试用例

针对**前端 chat-processor 层**的单元测试参考分类 ——
即 `components/src/dynamo/frontend/` 下、位于 OpenAI 风格 HTTP 请求与
后端引擎（backend engine）之间的代码。该层的测试位于
`components/src/dynamo/frontend/tests/` 下。

这是 `lib/parsers/PARSER_CASES.md` 的**前端**对照文档。两套分类法覆盖
不同的接口面：

| 文件 | 范围 | 前缀 |
|---|---|---|
| `lib/parsers/PARSER_CASES.md` | 工具调用解析器（tool-call parser）对**模型输出**的行为 | `PARSER.batch.*`、`PARSER.stream.*`、`PARSER.fmt.*`、`PARSER.xml.*`、`PARSER.harmony.*` |
| `lib/parsers/REASONING_CASES.md` | 推理（reasoning）解析器对**模型输出**的行为 | `REASONING.batch.*`、`REASONING.stream.*` |
| `lib/parsers/PIPELINE_CASES.md` | 流水线边界契约（解析器输出与上游元数据的独立性） | `PIPELINE.*` |
| `components/src/dynamo/frontend/tests/FRONTEND_CASES.md` | Chat-processor 层：请求预处理、输出装配、错误暴露、worker 接线 | `FRONTEND.*` |

本分类法覆盖的后端：**vllm**（`prepost.py` + `vllm_processor.py`）以及
**sglang**（`sglang_prepost.py` + `sglang_processor.py`）。trtllm 在
`components/src/dynamo/trtllm/` 下有自己的架构，不在本文档范围内。

## 速查

- **`FRONTEND.1`** Chat-template 输入预处理 —— 多轮 assistant `tool_calls` 中带 JSON 字符串 `arguments`、消息物化、role 处理、tool 消息、system 合并。（Richard 在 #8792 中针对 qwen3.5 的修复就在这里。）
- **`FRONTEND.2`** 解析器构造与分发 —— 根据请求的 `chat_template_kwargs`、模型名、运行时（runtime）配置，实例化正确的工具调用 / 推理解析器；当"无解析器"时也要优雅处理。
- **`FRONTEND.3`** 请求整形与采样参数投影 —— OpenAI 字段 → 后端 kwargs，当 `tool_choice="none"` 时剥离工具，引导式解码（guided-decoding）的设置。
- **`FRONTEND.4`** 工具调用输出装配 —— 模型输出流 → OpenAI 风格的 `tool_calls` deltas。单个、多个、与正文混合的、回退（fallback）路径。
- **`FRONTEND.5`** finish-reason 映射 —— 前端层进行重映射（`stop`/`length`/`tool_calls`）。这与解析器层的 `PIPELINE.finish_reason` 不同 —— 后者关注解析器对原始信号的视角。
- **`FRONTEND.6`** 增量反 tokenize（detokenization） —— token-id 流 → 文本，prompt-token-id 归一化，纯文本快路径。
- **`FRONTEND.7`** Worker 子进程边界 —— 预处理在子进程中运行；结果对象的可 pickle 性、初始化、跨边界的错误传播。
- **`FRONTEND.8`** 错误暴露 —— `BackendError` / `InternalError` / 引擎错误处理、格式错误响应、流错误、弃用警告。
- **`FRONTEND.9`** 推理 ↔ 工具调用编排 —— 同一响应同时启用两个解析器；这与 `REASONING.batch.2` 不同（后者纯粹关注输出文本）。

## 注解约定

测试以一行尾随注释标注其覆盖的 FRONTEND.X：

```python
class TestMapFinishReason:  # FRONTEND.5
    def test_stop_to_tool_calls_when_emitted(self): ...
```

如果一个类跨多个分类，可以逐个测试标注：

```python
class TestUtilities:
    def test_make_backend_error(self): ...  # FRONTEND.8
    def test_normalize_prompt_token_ids(self): ...  # FRONTEND.6
```

`grep -r 'FRONTEND.1' components/src/dynamo/frontend/tests/` 一次性
返回 vllm + sglang 中所有 chat-template 预处理相关的测试。

---

## `FRONTEND.1` —— Chat-template 输入预处理

前端在把多轮消息历史交给后端之前会重新构建 prompt。这里有几个易错点：

- 一些 chat 模板期望 assistant 的 `tool_calls.function.arguments` 是
  **dict**（因为模板里写的是 `arguments | items`），但 OpenAI 线协议
  传过来的是 **JSON 字符串**。前端必须按后端 / 按模型进行归一化。
- 物化（Materialization）：pydantic 模型 / dataclass / mapping 类型在模板
  运行前都必须变成普通 dict。
- 对物化后的 dict 做的修改**绝不能**回流到调用方拥有的请求对象。

示例：`TestPreprocessChatRequest`（sglang）、
`TestPrepareRequestToolStripping`（vllm）。

## `FRONTEND.2` —— 解析器构造与分发

对于一个给定请求，前端必须基于模型名、`chat_template_kwargs` 与运行时
配置选出正确的工具调用解析器与推理解析器。测试钉住了这个分发矩阵。

示例：`TestCreateParsers`、`TestRuntimeConfigParserName`、
`TestNoReasoningParser`。

## `FRONTEND.3` —— 请求整形与采样参数投影

OpenAI 请求字段 → 后端 `SamplingParams` 或等价物。当
`tool_choice="none"` 时剥离 tool。当 `tool_choice` 要求时配置引导解码。

示例：`TestConvertTools`、`TestBuildToolCallGuidedDecoding`、
`TestPrepareRequestToolStripping`。

## `FRONTEND.4` —— 工具调用输出装配

后端输出流 → OpenAI 风格的 `tool_calls` deltas。单个、多个、并行、
"先正文后工具调用"、无解析器命中时的回退。

示例：`TestSingleToolCall`、`TestMultipleToolCalls`、
`TestContentWithToolCalls`、`TestSingleChunkFallback`、
`TestMalformedToolCalls`、`TestJsonArrayParserReparse`、
`TestKimiToolCallIds`。

## `FRONTEND.5` —— finish-reason 映射

前端层把引擎的原始 finish_reason 映射到 OpenAI 的枚举上，并在某个 choice
已经发出过工具调用后，把 `stop` → `tool_calls`（每个 choice 单独跟踪）。
这与解析器分类法中的 `PIPELINE.finish_reason` 不同：那一项关注的是
解析器的视角（输出与上游信号无关）；这里关注的是前端在解析器之后的重映射。

示例：`TestMapFinishReason`。

## `FRONTEND.6` —— 增量反 tokenize

来自引擎的 token-id 流 → 用户可见的文本 deltas。包含 prompt-token-id 归一化、
（在没有检测到 tool / reasoning 标记时跳过解析器的）纯文本快路径，以及
chunk 边界处理。

示例：`TestIncrementalDetokenization`、`TestFastPlainTextPath`、
`TestNormalizePromptTokenIds`、`TestParseJsonArrayBuffer`。

## `FRONTEND.7` —— Worker 子进程边界

预处理在 worker 子进程中运行（避免 tokenizer 调用受 GIL 影响）。
结果对象必须可 pickle；初始化必须健壮；错误必须能干净地跨边界传播。

示例：`TestBuildDynamoPreproc`、`TestWorkerResultPicklability`。

## `FRONTEND.8` —— 错误暴露

前端如何把错误反馈给客户端：后端问题用 `BackendError`，自身 bug 用
`InternalError`，引擎错误映射为对 HTTP 友好的形式。也包括对历史字段的
弃用警告。

示例：`TestMakeBackendError`、`TestMakeInternalError`、
`TestHandleEngineError`、`TestDeprecationWarning`。

## `FRONTEND.9` —— 推理 ↔ 工具调用编排

当推理解析器与工具调用解析器在同一响应上同时被触发时，前端编排路由
（文本 → 推理 vs. 文本 → 工具调用标记 vs. 文本 → 用户可见正文）。
这与 `REASONING.batch.2` 不同：那一项关注解析器内部视角（推理解析器
必须保留工具调用标记给下游）；这里关注的是前端的装配视角。

示例：`TestReasoningParsing`。
