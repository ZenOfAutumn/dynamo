<!-- # SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0 -->

# 独立路由（Standalone Router）

一个与后端（backend）无关的、面向 Dynamo 部署的独立 KV 感知路由（router）服务。关于 KV 感知路由如何工作的细节，参见 [Routing Concepts](/docs/components/router/router-concepts.md)。

## 概述

独立路由可以为 Dynamo 部署中任意一组 worker 提供可配置的 KV 感知路由。它可用于解耦（disaggregated）服务（例如，将请求路由到 prefill worker）、多层架构，或任何需要根据 KV 缓存（KV cache）做出智能路由决策的场景。

该组件 **完全可配置**，可与任意 Dynamo 后端（vLLM、TensorRT-LLM、SGLang 等）以及任意 worker 端点一起使用。

## 使用方式

### 命令行

```bash
python -m dynamo.router \
    --endpoint dynamo.prefill.generate \
    --router-block-size 64 \
    --router-reset-states \
    --no-router-track-active-blocks
```

### 参数

**必填：**
- `--endpoint`：worker 的完整端点路径，格式为 `namespace.component.endpoint`（例如 `dynamo.prefill.generate`）

**路由配置：**
所有路由选项都使用 `--router-*` 前缀（例如 `--router-block-size`、`--router-kv-overlap-score-weight`、`--router-temperature`、`--router-kv-events` / `--no-router-kv-events`、`--router-replica-sync`、`--router-snapshot-threshold`、`--router-reset-states`、`--router-track-active-blocks` / `--no-router-track-active-blocks`、`--router-track-prefill-tokens` / `--no-router-track-prefill-tokens`）。无前缀的旧名称（如 `--block-size`、`--kv-events`）仍然可用，但已废弃。详细说明参见 [Configuration and Tuning](/docs/components/router/router-configuration.md)。

## 架构

独立路由通过 Dynamo 运行时（runtime）暴露两个端点：

1. **`generate`**：将请求路由到最佳 worker 并以流式方式返回生成结果（KV 感知路由）。
2. **`best_worker_id`**：给定 token id，返回该请求的最佳 worker id，但不进行路由；可用于调试或自定义路由逻辑。

客户端可调用 `generate` 端点流式获取补全结果，或调用 `best_worker_id` 决定使用哪个 worker，然后直接联系该 worker。

## 示例：手动解耦服务（替代方案）

> [!Note]
> **这是一种替代的高级搭建方式。** 解耦服务的推荐做法是使用 frontend 的自动 prefill 路由，当你以 `ModelType.Prefill` 注册 worker 时即会激活。默认搭建方式参见 [Disaggregated Serving](/docs/components/router/router-disaggregated-serving.md)。
>
> 当你需要对 prefill 路由配置进行显式控制，或希望分别管理 prefill 路由与 decode 路由时，再使用这种手动方式。

完整示例参见 [`examples/backends/vllm/launch/disagg_router.sh`](/examples/backends/vllm/launch/disagg_router.sh)。

```bash
# 为 decode worker 启动 frontend 路由
python -m dynamo.frontend \
    --router-mode kv \
    --http-port 8000 \
    --kv-overlap-score-weight 0  # decode 走纯负载均衡

# 为 prefill worker 启动独立路由
python -m dynamo.router \
    --endpoint dynamo.prefill.generate \
    --router-block-size 64 \
    --router-reset-states \
    --no-router-track-active-blocks

# 启动 decode worker
python -m dynamo.vllm --model MODEL_NAME --block-size 64 &

# 启动 prefill worker
python -m dynamo.vllm --model MODEL_NAME --block-size 64 --disaggregation-mode prefill &
```

>[!Note]
> **prefill 路由为何使用 `--no-router-track-active-blocks`？**
> 活跃块跟踪用于跨 decode（生成）阶段做负载均衡。对于仅 prefill 路由而言，decode 负载并不相关，因此关闭它可降低开销并简化路由状态。
>
> **何时使用 `--no-router-track-prefill-tokens`？**
> 用于仅 decode 路由，使其忽略已经完成的 prompt 工作。这样在 prefill 到 decode 的交接之后，`active_prefill_tokens`、队列压力与负载估算就能聚焦于 decode 端的工作。
>
> **为何独立路由必须指定 `--router-block-size`？**
> 与 frontend 路由不同——它能在 worker 注册时通过 ModelDeploymentCard（MDC）推断 block size，而独立路由无法访问 MDC，因此必须显式指定 block size。让其能自动推断的工作正在进行中。

## 配置最佳实践

>[!Note]
> **block size 一致性：**
> block size 必须在以下位置保持一致：
> - 独立路由（`--router-block-size`）
> - 所有 worker 实例（与后端相关，例如 vLLM 的 `--block-size`）
>
> **endpoint 一致性：**
> `--endpoint` 参数必须与目标 worker 注册的位置一致。例如：
> - vLLM prefill worker：`dynamo.prefill.generate`
> - vLLM decode worker：`dynamo.backend.generate`
> - 自定义 worker：`<your_namespace>.<your_component>.<your_endpoint>`

## 与后端集成

将独立路由与一个后端集成的步骤：

1. worker 应在 `--endpoint` 参数指定的端点上注册
2. 客户端调用 `router.generate` 端点以流式获取补全结果（路由选择最佳 worker），或调用 `router.best_worker_id` 获取最佳 worker id 后直接向该 worker 发送请求
3. 路由状态会在请求被路由时自动更新；不需要单独的 "free" 调用

参考实现参见 [`components/src/dynamo/vllm/handlers.py`](../vllm/handlers.py)（搜索 `prefill_router_client`）。

## 另请参阅

- [Router Guide](/docs/components/router/router-guide.md) - 部署模式与快速开始
- [Configuration and Tuning](/docs/components/router/router-configuration.md) - CLI flag、传输模式与指标（metrics）
- [Disaggregated Serving](/docs/components/router/router-disaggregated-serving.md) - prefill 与 decode 路由搭建
- [Router Design](/docs/design-docs/router-design.md) - 架构细节与事件传输模式
- [Frontend Router](../frontend/README.md) - 集成路由的主 HTTP frontend
- [Router Benchmarking](/benchmarks/router/README.md) - 性能测试与调优
