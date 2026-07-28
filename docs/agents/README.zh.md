---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Agents
subtitle: Dynamo 中面向 Agent 的服务能力
---

Dynamo 提供了一组面向智能体（agentic）工作负载的小型请求扩展和追踪工具。
Harness 仍然负责整个语义层面的 agent 轨迹。Dynamo 接收轻量级元数据，
并将其用于服务遥测、路由提示，以及后端特定的缓存行为。

## 核心概念

| 概念 | 用途 |
|---------|---------|
| [Agent Tracing](agent-tracing.md) | 被动的 `session_id` / `trajectory_id` 元数据，加上由 Dynamo 自身负责的请求耗时、token、缓存、worker 放置以及 harness 工具事件追踪。 |
| [Agent Hints](agent-hints.md) | 可选的逐请求提示，例如优先级、预期输出长度和推测式预填充。 |
| [Tool Calling](tool-calling.md) | 支持的工具调用解析器及解析器名称。 |
| [Reasoning](reasoning.md) | 面向链式思考（chain-of-thought）模型支持的推理解析器。 |
| [Chat Processors](chat-processor-options.md) | Dynamo、vLLM 和 SGLang 的预处理选项。 |

## 各后端专项指南

Agent 特性通过通用的请求元数据暴露，但各后端运行时的支持情况存在差异。

| 后端指南 | 内容 |
|---------------|----------|
| [SGLang for Agentic Workloads](../backends/sglang/agents.md) | 优先级调度、基于优先级的 radix 驱逐、推测式预填充，以及面向子 agent KV 隔离的流式会话控制。 |

## 请求接口

面向 agent 的请求元数据位于 OpenAI 兼容请求体的 `nvext` 下：

```json
{
    "nvext": {
        "agent_context": {
            "session_type_id": "deep_research",
            "session_id": "research-run-42",
            "trajectory_id": "research-run-42:researcher"
        },
        "agent_hints": {
            "priority": 5,
            "osl": 1024
        }
    }
}
```

当你需要在 LLM 调用、工具调用以及外部轨迹文件之间保持可追踪性时，使用 `agent_context`。
仅在 harness 具有 Dynamo 可据此采取行动的服务相关意图时才使用 `agent_hints`。
