---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Request Cancellation
---

本文档描述 Dynamo 如何在 Dynamo worker 之间实现请求取消（request cancellation），以中止正在执行中的请求。请求取消机制让进行中的请求可以提前终止，从而节省那些不再需要响应所要消耗的计算资源。

## 取消机制如何工作

### 前端（Frontend）取消检测

前端会监测每个客户端连接是否出现非预期的断开。当客户端在响应完整送达之前断开连接时，前端会检测到并发起取消。它涵盖两种场景：

1. **连接非预期关闭** —— 在响应流式发送开始之前，客户端在请求处理过程中断开。
2. **流非预期关闭** —— 当 SSE 流正在向客户端发送响应 token 时，客户端断开了连接。

在这两种情况下，前端都会取消请求对应的 `AsyncEngineContext`，并将取消信号传播到下游 worker 上所有关联的子上下文。

### Worker 取消检测

在 worker 端，runtime 会监测来自前端的 TCP 连接以接收取消信号。Worker 在以下三种场景下检测到取消：

1. **收到控制消息** —— 前端显式发送了取消的控制消息。
2. **TCP 连接断开** —— 前端在没有发送控制消息的情况下断开（例如前端崩溃或网络故障）。

当 worker 收到取消信号时，会在请求对应的 `AsyncEngineContext` 上设置相应状态。之后由 worker 的引擎实现负责观察该取消（例如调用 `is_stopped()`）并相应地终止处理过程。后端 worker 中如何实现取消处理，请参阅 [后端开发指南](../development/backend-guide.md#request-cancellation)。

### 取消的传播

取消通过相互关联的 `AsyncEngineContext` 对象在多层请求链中传播。当父上下文被取消时，所有关联的子上下文也会被自动取消。这样，当客户端在前端取消请求时，下游 worker 上所有相关的子请求都会被自动取消，从而在整个请求流水线中节省计算资源。

## 指标（Metrics）

Dynamo 在前端层与运行时层都暴露了 Prometheus 指标，用于监控请求取消情况。

### 前端指标

| 指标 | 类型 | 描述 |
|--------|------|-------------|
| `dynamo_frontend_model_cancellation_total` | Counter | 前端检测到的请求取消总数 |

#### 标签

| 标签 | 描述 | 取值示例 |
|-------|-------------|----------------|
| `model` | 请求中的模型名 | `Qwen/Qwen3-0.6B` |
| `endpoint` | 接收请求的 API 端点 | `completions`、`chat_completions`、`embeddings`、`images`、`videos`、`audios`、`responses`、`anthropic_messages`、`tensor` |
| `request_type` | 请求是 unary 还是 streaming | `unary`、`stream` |

**端点：** 在前端 HTTP 服务的 `/metrics` 上提供。

### 运行时指标

| 指标 | 类型 | 描述 |
|--------|------|-------------|
| `dynamo_component_cancellation_total` | Counter | 由 work handler 取消的请求总数 |

该指标使用 Dynamo 自动注入的组件标签：

| 标签 | 描述 | 取值示例 |
|-------|-------------|----------------|
| `dynamo_namespace` | Dynamo 命名空间 | `dynamo` |
| `dynamo_component` | 处理该请求的组件 | `backend`、`prefill`、`decode` |
| `dynamo_endpoint` | 组件内的端点 | `generate` |

该计数器使用了去重逻辑，确保即使同一请求同时检测到控制消息和 socket 关闭两种取消信号，也只会被计数一次。

注意：该指标记录的是 worker 收到的取消信号，而非该请求是否真的在引擎层面被中止。是否在引擎层观察到取消（例如调用 `is_stopped()`）并相应终止处理，取决于 worker 的引擎实现。

**端点：** 在 worker system metrics 端口的 `/metrics` 上提供（通常为 9100 端口）。

### 指标输出示例

前端指标（来自前端 HTTP 服务的 `/metrics`）：
```text
dynamo_frontend_model_cancellation_total{endpoint="chat_completions",model="Qwen/Qwen3-0.6B",request_type="stream"} 5
dynamo_frontend_model_cancellation_total{endpoint="chat_completions",model="Qwen/Qwen3-0.6B",request_type="unary"} 1
dynamo_frontend_model_cancellation_total{endpoint="completions",model="Qwen/Qwen3-0.6B",request_type="stream"} 2
```

运行时指标（来自 worker system 端口的 `/metrics`）：
```text
dynamo_component_cancellation_total{dynamo_component="backend",dynamo_endpoint="generate",dynamo_namespace="dynamo"} 8
```

## AsyncEngineContext Trait

Dynamo 请求取消系统的核心是 `AsyncEngineContext` trait。每一个请求流都会关联一个该 trait 的实例，用于管理异步操作的生命周期，包括流的标识、优雅关闭以及立即终止能力。

### 关键方法

#### 标识
- **`id()`**：返回流的唯一标识。该 ID 由用户为请求设置，对应的子请求可以使用相同的 ID 来与原始用户请求建立关联。

#### 状态查询
- **`is_stopped()`**：当通过 `stop_generating()` 请求过优雅取消时返回 `true`。它代表向 worker 发出的信号——该请求已被取消，应尽早返回。
- **`is_killed()`**：当通过 `kill()` 触发了硬终止时返回 `true`。这通常表示客户端与服务器之间的网络连接已断开，或者需要立即终止。

#### 异步状态监听
- **`stopped()`**：异步方法，在上下文进入 stopped 状态时完成。如果已 stopped，则立即返回。
- **`killed()`**：异步方法，在上下文进入 killed 状态时完成。如果已 killed，则立即返回。

#### 取消控制
- **`stop_generating()`**：用于取消请求的推荐方法。它通知引擎对该流优雅地停止产出结果。该方法是幂等的，并且不会使流中已有的结果失效。
- **`stop()`**：`stop_generating()` 的别名。
- **`kill()`**：在 `stop_generating()` 的基础上额外表示希望不必排空流中剩余项就直接终止。具体行为由实现决定，并不是所有引擎都支持。

#### 子请求管理
- **`link_child(child: Arc<dyn AsyncEngineContext>)`**：将一个子 `AsyncEngineContext` 关联到当前上下文。当对父上下文调用 `stop_generating()`、`stop()` 或 `kill()` 时，所有关联的子上下文也会按照关联顺序被自动调用同名方法。这在解耦服务场景中尤其有用：前端收到取消通知后取消发送给 worker 的请求，worker 又可以取消它对应的子请求（例如远程 prefill 操作）。

### 线程安全

`AsyncEngineContext` trait 通过 `Send + Sync` 约束保证了线程安全，可在多个线程与异步任务之间被安全地并发访问。

## Python 绑定

`AsyncEngineContext` 的能力通过 `Context` 类暴露给 Python，Rust 方法与 Python 方法基本是一一对应的。

### Python 的 Context 类

Python 的 `Context` 类对 Rust 的 `AsyncEngineContext` 进行包装，并暴露以下方法：

- **`id()`**：返回上下文的唯一标识
- **`is_stopped()`**：与 Rust 的 `is_stopped()` 等价的同步方法
- **`is_killed()`**：与 Rust 的 `is_killed()` 等价的同步方法
- **`stop_generating()`**：发出 stop generating 信号，等价于 Rust 的同名方法
- **`async_killed_or_stopped()`**：异步方法，当上下文进入 killed 或 stopped 状态（以先发生者为准）时完成。它通过 `tokio::select!` 将 Rust 的 `killed()` 与 `stopped()` 异步方法合并起来。

请求取消的可运行示例，请参见 [取消示例](https://github.com/ai-dynamo/dynamo/tree/main/examples/custom_backend/cancellation/README.md)。

### 在 Python 中使用 Context

无论是入站请求还是出站请求，都可以选择性地使用 context：

#### 入站请求
对入站请求而言，generate 方法可以在 `request` 参数之后可选地接收一个 `context` 参数。如果方法签名中显式声明了 `context`，它就会接收到入站请求的 context 对象。请求处理函数可以：

- 在开始昂贵操作前，使用 `context.is_stopped()` 同步地检查取消
- 通过 `await context.async_killed_or_stopped()` 异步监听取消

示例：
```python
async def generate(self, request, context):
    for i in range(1000):
        # Check for cancellation before expensive work
        if context.is_stopped():
            raise asyncio.CancelledError

        # Perform work...
        await expensive_computation()
        yield result
```

#### 出站请求
对出站请求而言，Python 脚本可以将 context 对象作为关键字参数传给出站的 runtime endpoint client 路由方法（如 `generate`、`round_robin`、`random`、`direct`）。脚本可以通过该 context 对象取消该出站请求。

这在父入站请求被取消时，需要同步取消其子出站请求的场景中尤其有用。在这种情况下，脚本只需把入站 context 对象传给出站请求即可，自动建立取消行为之间的联动。

示例：
```python
async def generate(self, request, context):
    # Forward the incoming context to outgoing request
    # If the incoming request is cancelled, the outgoing request will be too
    stream = await self.client.generate(request, context=context)
    async for response in stream:
        yield response
```
