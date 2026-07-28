---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
subtitle: "Matej Kosec, Ishan Dhanani, Benjamin Klieger, Dan Gil and Alec Flowers — April 2026"
description: "Streaming Tokens and Tools: Multi-Turn Agentic Harness Support in Dynamo"
keywords: agentic inference, responses api, messages api, tool calling, interleaved reasoning, prefix caching, agent hints, Dynamo
last-updated: Apr 30, 2026
---

# 流式 token 与工具调用：Dynamo 中的多轮 Agentic Harness 支持

一次 agentic 交互必须保持一个结构化的会话：assistant 轮把推理（reasoning）与一个或多个工具调用（tool call）交错呈现，随后的 user 轮把对应的工具结果反馈到模型上下文中。推理回放（reasoning replay）是模型相关、轮次相关的：有些推理需要保留，有些则需要丢弃。推理引擎（inference engine）有责任承担这种更具表达力的交互，并产出正确切分的 API 结果。工具调用解析与推理解析必须在挂接的 harness 消费响应之前发生。诸如代码生成这类高价值 agentic 工作流还依赖于响应及时的 harness 体验：推理段、工具调用事件、请求元数据需要在该轮展开过程中流式回传，而不是等待最终文本响应才到达。本文回顾我们在 Dynamo 上跑真实 agentic 客户端时学到的经验：我们如何加固解析器与 API 覆盖，以及这些解析器层是如何演化为可独立复用的 crate 的。

这些改动建立在我们[第一篇文章](./agentic-inference.md)的性能讨论之上 —— 那一篇关注的是 agentic 推理底下的服务架构：frontend、router、KV 缓存（KV cache）管理。这篇后续文章关注正确性、用户体验等价性以及性能。

Agentic harness 仍在快速演进。Claude Code、Codex 和 OpenClaw 通过不同的 API 接口暴露出相同的压力点，因此下面的示例聚焦于自定义服务栈所需复刻的核心行为。

![Standard server vs Dynamo across two turns of an agent loop. Each turn crosses the network at two edges: harness-to-server (where prompt stability decides cache reuse) and server-to-harness (where streaming tool dispatch decides when the harness can act). Dynamo changes both edges.](./images/fig-1-agent-loop.svg)

## 面向 Harness 的 Dynamo 设置

我们的实验使用了新发布的 `nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4` 模型，但同样的问题也存在于不同模型、不同 reasoning parser 与 tool-call parser 之间。

要复现我们的结果，请用兼容 Anthropic 的 API 配置 frontend，并启用保留 prompt、推理、工具状态的开关：

- `--enable-anthropic-api` 把 Anthropic Messages API 暴露给 harness。许多 harness 可以回退到默认的 Messages API，但体验会下降。
- `--strip-anthropic-preamble` 移除会破坏 KV 复用的 Anthropic 计费 header。
- `--enable-streaming-tool-dispatch` 让完整的工具调用一旦解码出来即可立即开始执行，而不必等到该轮结束。

把这些放在一起：
```bash
python -m dynamo.frontend \
  --http-port 8000 \
  --enable-anthropic-api \
  --strip-anthropic-preamble \
  --enable-streaming-tool-dispatch
```

worker 侧本次部署的关键设置：

- `--dyn-tool-call-parser <parser>` 与 `--dyn-reasoning-parser <parser>` 以模型期望的特定格式重建工具调用与推理块。这两个 parser 也决定了是否保留、转换或丢弃前几轮的推理。

## Prompt 稳定性是缓存复用的关键

Claude Code 会发送数千 token 的可复用 prompt 脚手架，其中大部分被设计成在不同用户与会话间保持一致。然而，每个 prompt 最前面是一个会话相关的计费 header，它会在被指向不去除该 header 的自定义 endpoint 时导致缓存未命中：

```text
x-anthropic-billing-header: cc_version=0.2.93; cch=abc123def456==;
You are Claude Code, an interactive CLI tool...
```

这些 header 会污染 KV 缓存，使其无法被复用，即便是同一用户跨会话也无法。位置零的一行变化意味着每一次新会话都从一个不同的 token 前缀开始，因此其后真正稳定的指令与工具定义就再也无法整齐复用了。

为了恢复 KV 缓存复用，Dynamo 增加了 `--strip-anthropic-preamble`。这一修复在机制上很小，但在运维上很重要：在 tokenize 之前移除不稳定的计费 header，让稳定的 prompt 从 token 0 开始。

实测影响很大。在 Dynamo B200 部署上跑一个 52K token 的 prompt：稳定前缀的 TTFT 为 `168ms`；在前缀里保留每个会话不同的 header，则 TTFT 升至 `912ms`；在 tokenize 前移除计费 header，则回到 `169ms`。在该负载下，不稳定 header 的代价是每次请求 `744ms`，把一个本可复用的 system prompt 变成了完全的冷启动 prefill。这意味着对于打到同一部署的新用户、或同一用户开启的新会话，TTFT 大约缩短 `5x`。

![TTFT across three prompt-prefix conditions on a 52K-token prompt: a stable prefix lands at 168 ms, the stripped Anthropic preamble at 169 ms, while a varying per-session billing header pushes TTFT to 912 ms — a single token at position zero is the difference between hot KV reuse and a full cold prefill.](./images/fig-2-ttft-prefix-stability.svg)

## 推理与工具调用解析的一些细节

把推理回放到下一轮没有一个普适正确的形式。某些模型在普通的 assistant 轮中故意丢弃前面的 thinking。带有交错工具调用的 agentic 轮则不同：在那种情况下，推理段往往需要继续附着在它所解释的工具调用之上。真正的契约是模型相关且轮次相关的。

Anthropic 的[4 月 23 日 Claude Code 事故复盘](https://www.anthropic.com/engineering/april-23-postmortem)给出了一个具体的生产例子：在缓存的 prompt 失效后，会话恢复时可以清除前几轮的 thinking 以减轻 prefill 负担。

当代推理模型一般会产生两种 assistant 轮：
- 推理后直接回答用户
- 推理后跟一个或多个工具调用

agentic 模型尤其擅长产生在单条响应中交错出现多段推理与工具调用的轮次：

```text
<think>reasoning_0</think> tool_call_0 <think>reasoning_1</think> tool_call_1
```

进入下一轮时，每一段推理都需要继续附着在它所解释的工具调用旁边。Dynamo 现已完整支持这种交错格式。此前，同一轮可能被重建为：

```text
<think>reasoning_0 reasoning_1</think> tool_call_0 tool_call_1
```

如果 assistant 轮被重建为一个笼统的推理块再接一坨工具调用，模型仍然有相同的全部 token，但失去了让它们有意义的顺序与分隔。这种"分组排序"源于早期模型 —— 那时一个轮只发出一段推理与一次工具调用。

除了上述顺序错乱的问题，我们还发现推理在下一轮之前经常被过度激进地丢弃。对某些模型来说，在没有工具调用的轮中丢弃前面的 thinking 是已经确立的行为，是模型微调的一部分（DeepSeek-R1 是最清晰的例子）。但对于交错的 agentic 轮 —— 前面的推理解释了工具调用序列 —— 这一行为就是错误的。这个问题很难发现，因为用户在出向响应中能看到推理被正确解码，而它在进入下一轮之前却*悄悄地*被破坏或丢弃。

我们在 Dynamo + TRT-LLM 的部署中验证了这一点：在 4x B200、TP=4 上跑 Nemotron-3-Super-120B-A12B-NVFP4，启用 `--enable-anthropic-api`、`--strip-anthropic-preamble`、`--enable-streaming-tool-dispatch`，使用 `nemotron_deci` 推理 parser 与 `qwen3_coder` 工具调用 parser。

### 推理 + 工具调用的组合

在调用工具之前先推理的模型，会产生先输出 `<think>` 内容、再输出 `<tool_call>` XML 的响应。在 Nemotron 这一例中，要由两个不同的 parser —— 用于推理的 `nemotron_deci` 与用于工具调用的 `qwen3_coder` —— 把这条流切分成正确的 Anthropic Messages API 内容块，且彼此互不干扰。

我们通过 Anthropic Messages API 把同一 prompt 发送了 5 次：一段指示模型按步骤思考的 system prompt、两个工具定义（calculator 与 weather），以及用户消息 "Think carefully about what 15 * 23 equals, then use the calculator to verify."。其中一次代表性回合的响应结构：

```json
{
  "content": [
    {
      "type": "thinking",
      "thinking": "I need to calculate 15 * 23. Let me think: 15 * 20 = 300, and 15 * 3 = 45, so 300 + 45 = 345. I'll use the calculator to verify.\n"
    },
    {
      "type": "tool_use",
      "id": "call-a3364797-3160-4e84-b567-5c495694d502",
      "name": "calculator",
      "input": { "expression": "15 * 23" }
    }
  ],
  "stop_reason": "tool_use",
  "usage": { "input_tokens": 403, "output_tokens": 95 }
}
```

### 同时用两个 parser 流式处理

在流式路径上，两个 parser 的相互作用更加可见。一个流式请求会产生一系列 SSE 事件，事件类型序列清楚展示了两个 parser 是如何切分 token 流的：

```text
   1ms  message_start
  82ms  content_block_start  type=thinking
  82ms  content_block_delta  (thinking tokens stream here, ~7ms apart)
   ...  (~70 thinking deltas over ~520ms)
 602ms  content_block_stop
 602ms  content_block_start  type=text
 602ms  content_block_delta
 800ms  content_block_stop
 800ms  content_block_start  type=tool_use
 800ms  content_block_delta
 800ms  content_block_stop
 814ms  message_delta        stop_reason=tool_use
 814ms  message_stop
```

thinking 块从 `82ms` 起逐 token 流式输出，到 `602ms` 结束。然后出现一个短暂的 text 块（原始 token 流中 thinking 与 tool call 区域之间的空白）。然后在 `800ms` 出现 tool_use 块，作为单一结构化单元到达。`message_stop` 在 `814ms` 紧随其后。

直到 [PR #7358](https://github.com/ai-dynamo/dynamo/pull/7358) 之前，这一往返并不会产生正确的 Anthropic 事件序列。修复包含三部分：

1. **推理解析的归属唯一**：以前推理解析在多个相互竞争的层都会发生。后端 parser 可以把模型输出切成 `reasoning_content` 与普通 `content`，而 Anthropic 流式转换器在把同一条流映射成 Anthropic 内容块时仍会尝试推断 `<think>` 边界。PR #7358 把归属明确化：如果后端路径已经产生了结构化的推理 delta，Anthropic 转换器就信任它们，只负责把它们映射到响应格式。

2. **可用时使用模板原生的 reasoning**：Dynamo 现在会检查当前激活的 chat template 是否懂得读取 `reasoning_content`。Nemotron 与 Qwen3 这类模板会直接读取该字段，因此 Dynamo 不再插手，让模板自己决定保留多少前置 thinking。如果模板只懂 `content`，Dynamo 会回退到旧表示：通过把 `<think>` 块插入 `content` 来保留推理；当模型/parser 策略说前置 thinking 不应进入下一轮时则丢弃。Rust 预处理路径（`ModelInput::Tokens`）与 Python worker 路径（`ModelInput::Text`）使用同一条件规则。

3. **尊重每个请求的 thinking 控制**：许多模板默认 `truncate_history_thinking=true` 以节省上下文。在普通对话中这是合理的，但在 agent 工作流里它会移除前几次工具调用背后的推理。Dynamo 现在只会针对真正涉及推理的请求改变这一行为：当配置了推理 parser 且客户端没有禁用 thinking 时，Anthropic 路径会设置 `enable_thinking=true` 与 `truncate_history_thinking=false`。这样保留了 agent 所需的下一轮上下文，又不会改变那些应当不带 thinking 运行的请求或模型的默认行为。

在我们 B200 实验中，使用 52K-token system prompt 与一个包含约 500 token thinking 的 assistant 轮：未被改动的下一轮前缀的 TTFT 为 `167ms`，而被篡改的 thinking 让 TTFT 升至 `322ms`。这是 `1.9x` 的增加，约为每次请求 `155ms`，源自下一轮前缀里推理内容被改动。

关键启示是：harness、parser 与 template 路径必须就每个模型期望的推理行为达成一致。对一个模型而言，丢弃普通轮的 thinking 可能正确，对另一个模型则可能错误。即使普通轮允许剥离推理，工具调用轮上的交错推理也可能必须保留。**实践中你不应假设第 `N` 轮产生的 token 会原封不动地成为第 `N+1` 轮的前缀。** 这是否成立取决于你正在服务的模型的推理 parser、工具 parser 以及 chat template。

## 流式工具调用

流式 token 让用户体验更具响应性与动感。难点在于既要保持这种流式行为，又要把工具调用作为完整一致的块输出。在较旧的 Dynamo 路径中，推理 token 可以正常流式回传，但工具调用会一直被缓冲到该轮结束才一次性释放给 harness。这降低了响应性，并延迟了工具执行 —— 哪怕模型已经决定要调用什么。

| 状态 | harness 看到的内容 | 工具就绪何时可见 |
|-------|------------------------|-------------------------------------|
| 缓冲 | 工具调用 chunk 被扣留 | 仅在 `finish_reason: "tool_calls"` 时 |
| 内联流式 | 常规的工具调用 delta | 模型一发出就能看到 |
| Dispatch | 类型化的 `event: tool_call_dispatch` 旁路 | 在结构性完成点处即可见，且已被解析 |

关键变化是从第一行到后两行。这正是 harness 不再需要等待流结束才知道自己要行动的拐点。
没有 dispatch 时，harness 看到的是常规 token 流，必须靠累积 delta、等到结构足够完整时才推断出工具调用已完成。启用 dispatch 后，Dynamo 可以发出一个类型化的 SSE 旁路：

```text
event: tool_call_dispatch
data: {"choice_index":0,"tool_call":{"index":0,"id":"call-...","type":"function","function":{"name":"calculator","arguments":"{\"expression\":\"42 * 17\"}"}}}
```

该事件一次性告诉 harness：这次工具调用已可执行。无需 harness 端重组 delta、不必猜测参数是否完整、也不必在 harness 内部内嵌自定义 parser。这让 Dynamo 更容易兼容自定义 harness。

![Tool dispatch timing on a single turn. Standard servers surface the tool call only after the entire stream finishes; Dynamo emits a typed tool_call_dispatch event the moment the call is parsed, so the tool can run in parallel with the rest of the stream. Δt is the time saved per tool call.](./images/fig-3-streaming-dispatch-timeline.svg)

## 面向 Claude Code 与 OpenClaw 的 Anthropic API 兼容性

Claude Code 与 OpenClaw 都使用 Anthropic Messages API，而不是仅在某个 endpoint 后做文本生成。要匹配 harness 的体验，依赖于一组在临时测试中很容易遗漏的小行为：

- `GET /v1/models` 与 `GET /v1/models/{model_id}` 都要返回模型元数据
- 正确处理带斜杠的 model ID
- 在 `message_start` 中提供有用的 `input_tokens`
- 接受 `cache_control`

一旦 frontend 可达且合规，两种 harness 都可以指向 Dynamo 的 Anthropic 兼容 endpoint：

```bash
ANTHROPIC_API_KEY=local-dev-token \
ANTHROPIC_BASE_URL=http://localhost:8000 \
ANTHROPIC_CUSTOM_MODEL_OPTION=nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4 \
ANTHROPIC_CUSTOM_MODEL_OPTION_NAME="Dynamo NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4" \
claude --model nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4

ANTHROPIC_API_KEY=local-dev-token \
ANTHROPIC_BASE_URL=http://localhost:8000 \
npx openclaw agent --local -m "Say ok" --json
```

这里的修复让自定义部署更接近原生后端的行为。一个具体例子比一长串清单更能体现这些问题的味道。在启动期间，harness 直接询问选定模型的详情，而 Dynamo 当时并未提供该 endpoint：

```text
GET /v1/models/nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4
HTTP/1.1 404 Not Found
```

另一个例子是 `message_start` 报告 `input_tokens: 0`，而最终响应里又包含真实的 token 数。这会导致 harness 中的 token 计数在每个新轮开始时短暂掉到 `0`。[PR #7234](https://github.com/ai-dynamo/dynamo/pull/7234) 通过在流开始前填充 `input_tokens` 修复了这条 Anthropic 路径。这些计数在长会话中也属于控制平面数据：harness 用上下文长度判断何时在下次请求超出模型窗口前压缩对话。更广泛的 tokenizer 服务工作单独落在 [PR #7699](https://github.com/ai-dynamo/dynamo/pull/7699)，它新增了 `/v1/tokenize` 与 `/v1/detokenize` endpoint，以便在请求被引擎处理之前精确地获得 token 数。

## Responses API 与 Codex 的兼容性

同一问题在 Codex 这一侧位于 `v1/responses` 路径。仅仅通过合规测试并不足以提供用户体验上的对等。
我们发现 Responses API 的请求在内部往返时会丢失把它定义为 Responses 请求（而不是 chat completions 请求）的字段。要保留这些字段，需要在 Dynamo 的 `ResponseParams` 路径上做架构性改动，配合 [PR #6089](https://github.com/ai-dynamo/dynamo/pull/6089) 的上游类型对齐工作。

Codex 应通过启用了请求压缩的 OpenAI 兼容 Responses API 指向 Dynamo：

```bash
OPENAI_API_KEY=local-dev-token \
codex exec \
  -c 'openai_base_url="http://localhost:8000/v1"' \
  -c 'features.enable_request_compression=true' \
  -m nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4 \
  "Say ok"
```

### Codex 的模型元数据塑造请求

Codex 的对等性始于第一个 `POST /v1/responses` 之前。CLI 会把配置的模型字符串解析为本地模型目录中的一条记录，得到的 `ModelInfo` 控制了 harness 围绕该模型搭建的状态：基础指令、历史格式化、工具注册表、推理参数、verbosity 控制、图像支持、上下文统计、工具输出截断、`parallel_tool_calls`，以及最终的 Responses payload。

两个 endpoint 即使服务于同一个底层模型，也可能因 Codex 附加了不同的目录元数据而驱动出不同的 agent 行为。请求可能在 schema 上通过校验，但围绕它的 harness 已经发生变化。

工具输出截断是一个有用的例子。Codex 不会把无限的命令输出回放到下一轮模型上下文中。shell 与工具的观察会按选定模型的目录策略截断后再回到上下文。在我们测试的目录快照中，`gpt-5.5` 使用：

```json
{ "mode": "tokens", "limit": 10000 }
```

相比之下，自定义 endpoint 上的 `openai/openai/gpt-5.5` 使用了回退元数据：

```json
{ "mode": "bytes", "limit": 10000 }
```

这两个预算并不等价。对于 ASCII 密集的代码输出来说，`10,000` 字节限制会比 `10,000` token 限制更早截断结构化日志、traceback、JSON 或测试输出。对一个 coding agent 来说，这改变了模型在测试失败、搜索命令或编译器错误之后能够检视到的内容。模型可能需要额外的工具调用来恢复本应由原本的目录配置保留的上下文。

推理设置同样源自目录。当选定的模型元数据声明支持 reasoning summary 时，Codex 会发送 Responses 中的 `reasoning` 对象。在该路径下，Codex 还会请求 `reasoning.encrypted_content`，以便跨轮回放推理状态。回退元数据会移除这条路径。

prompt 也会变化。在 Codex 中，从 fallback/默认 profile 切换到 `gpt-5.5` 目录 profile 会改变 system prompt。fallback prompt 围绕通用 Codex 操作组织（`# How you work`、`# AGENTS.md spec`、`# Tool Guidelines`），强调 `AGENTS.md` 优先、计划、验证以及 shell 搜索习惯。`gpt-5.5` prompt 是一份不同的指令文档（`# Personality`、`# General`、`# Working with the user`），把 agent 框定为务实的软件工程师，并对阅读代码库、复用本地模式、限定范围的修改、脏 worktree、`apply_patch`、协作更新与最终回答的格式提供更强的指引。也就是说，目录别名（aliasing）不仅影响 truncation 与 reasoning 这类请求字段，也影响基础行为策略。

我们在 SWE-Bench Verified 的 50 任务子集中直接看到这一点。在该设置下，两条路径最终都到达 OpenAI 提供的 GPT-5.5；区别在于 endpoint 以及 Codex 给它附加的模型目录记录。当自定义 endpoint 使用 model ID `openai/openai/gpt-5.5` 而未关联到 `gpt-5.5` 目录 profile 时，Codex 走通用的 fallback 行为。在一次实验中，fallback profile 大约只发起了一半的工具调用：

| 目录 profile | 总工具调用数 | 每任务 |
|-----------------|------------------|----------|
| `gpt-5.5` profile | 2,087 | 41.7 |
| Fallback profile | 1,048 | 21.0 |
| Delta | -1,039 | -20.8 |

逐任务的成对比较在每一项上都指向同一方向：`gpt-5.5` profile 在 `50 / 50` 任务中使用了更多工具，而 fallback profile 在 `0 / 50` 任务中胜出。置换检验（permutation test）显示差异 `p < 0.001`。

在加入一个目录别名让 `openai/openai/gpt-5.5` 继承期望的 `gpt-5.5` profile 之后，同样的 50 任务设置变得非常接近：

| 目录 profile | 总工具调用数 | 每任务 |
|-----------------|------------------|----------|
| `gpt-5.5` profile | 2,081 | 41.6 |
| 由别名支撑的 custom profile | 2,205 | 44.1 |
| Delta | +124 | +2.5 |

剩余差异在该次运行中并不显著：置换检验约为 `p = 0.22`，成对方向也呈现混合（`20 / 50` 任务偏好原生 profile，`28 / 50` 偏好别名支撑的 profile，`2 / 50` 持平）。

对 Dynamo 的启示是：Codex 的兼容性需要在目录与请求整形层评估，而不仅仅是 HTTP schema 层。如果 Codex 无法把 model ID 解析到期望的 profile，回退默认值可能在 Dynamo 收到请求之前就改变了截断方式、搜索工具可用性、verbosity 控制、reasoning summary 支持以及 parallel tool call 支持。

## 接下来

Dynamo 现在有了 `nvext.agent_hints`：`latency_sensitivity`、`priority`、`osl` 与 `speculative_prefill`。这些字段让 harness 能在 prompt 之外向后端传达更多关于这一轮的信息。一个等待用户回复的会话与一个正在跑长背景工具序列的会话并不一样，API 现在能把这种差别传递出来。

在 v1.1.0 这条线上，Dynamo 还把 agent 栈的更多部分作为可复用片段开放出来。protocol、parser 与 tokenizer 层以独立 crate 的形式版本化发布，包括 `dynamo-protocols`、`dynamo-parsers` 与 `dynamo-tokenizers`。这让团队能够构建或定制面向 harness 的服务路径，而无需把 Dynamo 内部实现拷贝到一个独立项目。

这也是通往诸如 AutoResearch 这类长时运行系统的桥梁。第一篇文章解释了为什么 agentic 负载会给服务栈带来压力。本文则展示了正确运行这类负载所需的 harness 端契约，并为以 Dynamo endpoint 支撑高效长时运行 agent 奠定了基础。
