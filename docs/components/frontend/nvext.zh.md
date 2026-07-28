---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: NVIDIA 请求扩展（nvext）
---

`nvext` 是请求体顶层的一个 JSON 对象，为 OpenAI 兼容 API 提供 NVIDIA 特有的扩展。`nvext` 字段会被 Dynamo 的 frontend、preprocessor、router 与后端 worker 消费，用于控制路由、预处理、响应元数据、调度以及 engine 级优先级。

## 用法

把 `nvext` 与标准 OpenAI 字段并列地放在请求体顶层：

```json
{
    "model": "my-model",
    "messages": [{"role": "user", "content": "Hello"}],
    "nvext": {
        "greed_sampling": true,
        "extra_fields": ["worker_id", "timing"],
        "agent_hints": {
            "osl": 1024,
            "priority": 5
        }
    }
}
```

## 字段说明

| 字段 | 类型 | 默认值 | 由谁消费 | 说明 |
|-------|------|---------|-------------|-------------|
| `greed_sampling` | `bool` | `None` | Preprocessor | 强制使用贪心采样，忽略其他采样参数 |
| `use_raw_prompt` | `bool` | `None` | Preprocessor | 跳过 prompt template，直接把 prompt 传给 tokenizer |
| `annotations` | `string[]` | `None` | Preprocessor | 通过 SSE 流的 `event:` 字段触发带外信息 |
| `backend_instance_id` | `u64` | `None` | Router | 把请求路由到指定的 backend instance |
| `token_data` | `u32[]` | `None` | Preprocessor | 已分词的 prompt token。当与 `backend_instance_id` 一起提供时，将跳过分词 |
| `max_thinking_tokens` | `u32` | `None` | Backend | 允许的最大 thinking token 数（透传给 backend） |
| `extra_fields` | `string[]` | `None` | Response builder | 要包含在响应 `nvext` 中的字段。可选：`"worker_id"`、`"timing"`、`"routed_experts"`、`"engine_data"`、`"stop_reason"` |
| `prefill_worker_id` | `u64` | `None` | Router | 把请求路由到指定的 prefill worker（解耦服务场景） |
| `decode_worker_id` | `u64` | `None` | Router | 把请求路由到指定的 decode worker（解耦服务场景） |
| `agent_context` | object | `None` | Preprocessor | 用于 agent trace 的被动 session 与 trajectory 标识。详见下文 [Agent Context](#agent-context) 与 [Agent Tracing](../../agents/agent-tracing.md) |
| `agent_hints` | object | `None` | Router | 单请求级别的调度与负载均衡 hint。详见 [Agent Hints](#agent-hints) |
| `session_control` | object | `None` | Router | subagent KV 隔离用的 session 生命周期与粘性路由。详见 [Session Control](#session-control) |

相关的 Dynamo 顶层输出选项：

| 字段 | 类型 | 默认值 | 由谁消费 | 说明 |
|-------|------|---------|-------------|-------------|
| `return_tokens_as_token_ids` | `bool` | `false` | Response builder | 把 logprob 的 token 字符串格式化为 `token_id:<id>`，而不是解码后的文本 |

`return_tokens_as_token_ids` 仅改变返回的 logprob token 显示形式。如果要按 token ID 触发 stop，请在标准的 `stop` 数组中传整数 ID，例如 `"stop": [576]`。形如 `"token_id:576"` 的字符串仍会被当作字面 stop 序列，**不会**被解析为 token ID。

### Header 覆盖

路由相关的字段还可以通过 HTTP header 设置，header 优先级高于 `nvext` 的值：

| Header | 覆盖的字段 |
|--------|-----------|
| `x-worker-instance-id` | `backend_instance_id` 与 `decode_worker_id` |
| `x-prefill-instance-id` | `prefill_worker_id` |

## Agent Context

`agent_context` 子对象承载 agent 类请求的被动 session / trajectory 标识。在启用 agent trace sink 时，Dynamo 会用它发送请求 trace。它**不会**改变路由、调度或 cache 行为。

| 字段 | 类型 | 必填 | 说明 |
|-------|------|:--------:|-------------|
| `session_type_id` | `string` | 是 | 可复用的 profile 或 agent class 标签 |
| `session_id` | `string` | 是 | 顶层 agent run / session 标识 |
| `trajectory_id` | `string` | 是 | 一个可调度的推理 / 工具 trajectory |
| `parent_trajectory_id` | `string` | 否 | 父 trajectory，通常用于 subagent |

```json
{
    "nvext": {
        "agent_context": {
            "session_type_id": "deep_research",
            "session_id": "research-run-42",
            "trajectory_id": "research-run-42:researcher",
            "parent_trajectory_id": "research-run-42:planner"
        }
    }
}
```

关于身份语义、trace sink 配置与 JSONL schema 的细节，请参阅 [Agent Tracing](../../agents/agent-tracing.md)。

## Agent Hints

`agent_hints` 子对象承载单请求级别的 hint，由 router 用于调度、负载均衡与 KV cache 优化。

| 字段 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------------|
| `priority` | `i32` | `None` | 统一的请求优先级。在 Dynamo API 层数值越大优先级越高。用于 router 排队顺序与后端调度 / 驱逐 |
| `osl` | `u32` | `None` | 预期输出序列长度（token 数），用于 output block 追踪与资源估算 |
| `speculative_prefill` | `bool` | `false` | `true` 时在当前轮结束后投机性预填充预测的下轮 prompt，以预热 KV cache |

### `priority`

`priority` 是面向用户的唯一调度 hint。在 Dynamo 内部，"数值越大 = 越重要"。

当设置了 `--router-queue-threshold` 且队列处于活跃状态时，高优先级请求会在 router 队列中前移。被分发后，Dynamo 把同一份语义优先级转发给后端 engine，用于排队、抢占与 KV cache 驱逐。Dynamo 内部会自动处理后端不同的极性（包括 vLLM "数值越小越高"的约定）。

```json
{
    "nvext": {
        "agent_hints": {
            "priority": 5
        }
    }
}
```

### `osl`

预期输出序列长度——请求估计将生成的 output token 数。router 通过该 hint 做两件事：

1. **Output block 追踪**：启用 `--router-track-output-blocks` 后，router 会在生成期间添加占位 block，并依据相对 `osl` 的进度做分数衰减。
2. **资源估算**：在做路由决策时帮助 router 估算总资源需求。

```json
{
    "nvext": {
        "agent_hints": {
            "osl": 1024
        }
    }
}
```

### `speculative_prefill`

当置为 `true` 时，系统会在当前 assistant 轮结束后投机性地预填充预测的下轮 prompt。此特性面向多轮 agent 类负载——下次请求的前缀是可预测的。

工作方式：

1. assistant 响应流式输出过程中，系统累积完整响应文本。
2. 响应结束后，后台任务把 assistant 响应追加到对话历史，构造下一轮 prompt（非最后一轮会剥离 thinking 内容）。
3. 构造的 prompt 被分词，并以 `max_tokens=1` 的请求发出，用以在某个 worker 上预热 KV cache。
4. 真正的下一次请求到来时，能直接受益于已预热的 KV cache，从而降低 TTFT。

```json
{
    "nvext": {
        "agent_hints": {
            "speculative_prefill": true
        }
    }
}
```

后端配置说明：

- **SGLang**：需开启 `--enable-priority-scheduling`（用于排队顺序）以及 `--radix-eviction-policy priority`（用于基于优先级的驱逐）。
- **vLLM**：需开启 `--scheduling-policy priority`。
- **TensorRT-LLM**：当前不支持单请求优先级。

```json
{
    "nvext": {
        "agent_hints": {
            "priority": 5
        }
    }
}
```

## Session Control

`session_control` 启用 subagent KV 隔离与粘性路由。router 用 `session_id` 把同一 session 锁定在同一个 worker 上，并能在流式 session 起止时下发 `open` / `close` 生命周期 RPC。

| 字段 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------------|
| `session_control.session_id` | `string` | — | 唯一 session 标识，每一轮都需携带 |
| `session_control.action` | `string` | 省略 | 可选生命周期动作：`"open"` 或 `"close"` |
| `session_control.timeout` | `integer` | `300` | 不活动超时（秒）。仅与 `action: "open"` 一同使用 |

```json
{
    "nvext": {
        "session_control": {
            "session_id": "subagent-1",
            "action": "open",
            "timeout": 300
        }
    }
}
```

需要 frontend 启动时设置 `--router-mode=kv`。当请求携带 `nvext.session_control` 时，session control 自动激活。后端配置详见 [SGLang for Agentic Workloads](../../backends/sglang/agents.md)。


## 响应扩展

当客户端通过 `extra_fields` 请求响应元数据时，响应里会带上一个 `nvext` 对象，包含被请求的字段：

| 字段 | 通过什么请求 | 说明 |
|-------|---------------|-------------|
| `worker_id` | `extra_fields: ["worker_id"]` | 处理该请求的 prefill / decode worker ID 与 data-parallel rank |
| `timing` | `extra_fields: ["timing"]` | 单请求时序信息（TTFT、ITL、排队时间等） |
| `routed_experts` | `extra_fields: ["routed_experts"]` | SGLang 后端请求返回的 routed expert capture payload |
| `engine_data` | `extra_fields: ["engine_data"]` | 由后端提供的不透明 engine 元数据 |
| `stop_reason` | `extra_fields: ["stop_reason"]` | 后端命中的具体停止条件，由于不属于 OpenAI completions schema，故放在 `nvext` 中。Dynamo 当前以"响应级"字段返回，仅适用于单 choice 请求；支持 `n > 1` 还需要按 choice 索引化的结构 |
| `token_ids` | 自动（GAIE Stage 1） | 已分词的 prompt，用于 Stage 2 query-only 模式复用 |

### 响应 `nvext` 示例

```json
{
    "nvext": {
        "worker_id": {
            "prefill_worker_id": 1,
            "prefill_dp_rank": 0,
            "decode_worker_id": 2,
            "decode_dp_rank": 0
        },
        "timing": {
            "ttft_ms": 45.2,
            "itl_ms": 12.1
        }
    }
}
```

## 另请参阅

| 文档 | 说明 |
|----------|-------------|
| [Frontend Guide](frontend-guide.md) | KServe gRPC 配置与集成 |
| [Configuration and Tuning](../router/router-configuration.md) | 完整的 router 配置与 CLI 参数 |
| [Agent Tracing](../../agents/agent-tracing.md) | 被动 session/trajectory 标识、JSONL 请求 trace 与 harness 工具事件采集 |
| [Agent Hints](../../agents/agent-hints.md) | 路由、调度与 cache 行为的单请求 hint |
| [SGLang for Agentic Workloads](../../backends/sglang/agents.md) | SGLang engine 的优先级调度、驱逐策略与 session 控制相关参数 |

