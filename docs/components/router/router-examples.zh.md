---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Router Examples
---

如需快速上手，请参阅 [Router README](README.md)。本文档提供了使用 Dynamo Router 的更多示例，包括 Python API 用法、Kubernetes 部署以及自定义路由（router）模式。

## 使用 KvRouter Python API

除了通过命令行启动 KV Router，你还可以直接在 Python 中创建 `KvRouter` 对象。这样可以为单次请求覆盖路由配置。

>[!Warning]
> **同一 runtime 上的多个 Router**：不要从同一个 `DistributedRuntime` 创建多个独立管理的 `KvRouter` 实例。从同一个 runtime 拥有的 endpoint 创建出的 router 会共享该 runtime 的主 cancellation token，所以销毁其中一个 router 可能会取消其他 router 仍在使用的后台任务。对于一个进程内的前端（frontend），请使用单一 `KvRouter`；如果需要彼此独立的 router 生命周期，请使用独立的前端进程，或为每个 router 使用独立的 `DistributedRuntime`。

在事件循环（event loop）以 `loop` 提供的情况下，进程内独立的 router 生命周期需要各自独立的 runtime：

```python
router_a = KvRouter(DistributedRuntime(loop, "etcd", "tcp").endpoint("dynamo.backend.generate"), 16, KvRouterConfig())
router_b = KvRouter(DistributedRuntime(loop, "etcd", "tcp").endpoint("dynamo.backend.generate"), 16, KvRouterConfig())
```

### 方法

`KvRouter` 提供以下方法：

- **`generate(token_ids, model, ...)`**：路由并执行一次请求，返回响应的异步流。会自动处理 worker 选择、状态跟踪与生命周期管理。

- **`best_worker(token_ids, router_config_override=None, request_id=None, update_indexer=False)`**：查询若给定 token 应选择哪个 worker，返回 `(worker_id, dp_rank, overlap_blocks)`。
  - 不传 `request_id`：仅查询，不更新 router 状态
  - 传 `request_id`：更新 router 的生命周期状态以跟踪该请求。**注意**：使用 `request_id` 时，必须在合适的生命周期点调用 `mark_prefill_complete()` 与 `free()`，以维持准确的负载跟踪
  - 设置 `update_indexer=True`：把所选 worker 记录到近似 indexer 中，以便后续做重叠预测。仅在 `use_kv_events=False` 时才有意义

- **`get_potential_loads(token_ids)`**：获取所有 worker 的详细负载信息，包括潜在的 prefill token 数与活跃 decode block 数。返回一个负载字典列表。

- **`mark_prefill_complete(request_id)`**：通知 router 某请求的 prefill 阶段已完成。仅在使用 `best_worker()` 进行手动路由（而非 `generate()`）时的[手动生命周期管理](#2-manual-state-management-advanced)中使用。

- **`free(request_id)`**：通知 router 某请求已结束、应释放其资源。仅在使用 `best_worker()` 进行手动路由（而非 `generate()`）时的[手动生命周期管理](#2-manual-state-management-advanced)中使用。

- **`dump_events()`**：将 router indexer 中所有 KV 缓存（KV cache）事件以 JSON 字符串形式导出。便于调试与分析。

### 准备工作

首先启动后端引擎：
```bash
python -m dynamo.vllm --model meta-llama/Llama-2-7b-hf
```

### 示例脚本

```python
import asyncio
from dynamo.runtime import DistributedRuntime
from dynamo.llm import KvRouter, KvRouterConfig

async def main():
    # Get runtime and create endpoint
    loop = asyncio.get_running_loop()
    runtime = DistributedRuntime(loop, "etcd", "nats")
    endpoint = runtime.endpoint("dynamo.backend.generate")

    # Create KV router
    kv_router_config = KvRouterConfig()
    router = KvRouter(
        endpoint=endpoint,
        block_size=16,
        kv_router_config=kv_router_config
    )

    # Optional startup gate shared with the frontend and standalone indexer:
    # os.environ["DYN_ROUTER_MIN_INITIAL_WORKERS"] = "2"

    # Your input tokens
    token_ids = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

    # Generate with per-request routing override
    stream = await router.generate(
        token_ids=token_ids,
        model="meta-llama/Llama-2-7b-hf",
        stop_conditions={
            "max_tokens": 20,        # Generate exactly 20 tokens
            "ignore_eos": True,      # Don't stop at EOS token
        },
        sampling_options={
            "temperature": 0.7,
            "top_p": 0.9,
        },
        router_config_override={
            "overlap_score_weight": 2.0,    # Prioritize cache hits for this request
            "router_temperature": 0.5,       # Add routing randomness
        }
    )

    # Collect generated tokens
    generated_tokens = []
    async for response in stream:
        if isinstance(response, dict) and "token_ids" in response:
            generated_tokens.extend(response["token_ids"])

    print(f"Generated {len(generated_tokens)} tokens: {generated_tokens}")

if __name__ == "__main__":
    asyncio.run(main())
```

## K8s 示例

KV Router 的基本 Kubernetes 部署，请参见快速上手指南中的 [Kubernetes Deployment 章节](README.md#kubernetes-deployment)。

### 完整 K8s 示例

- [TRT-LLM 聚合路由示例](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/trtllm/deploy/agg_router.yaml)
- [vLLM 聚合路由示例](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/deploy/agg_router.yaml)
- [SGLang 聚合路由示例](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/sglang/deploy/agg_router.yaml)
- [Kubernetes 部署指南](../../kubernetes/README.md)

**A/B 测试与高级 K8s 部署：**
关于在 Kubernetes 中部署、配置与压测 KV router 的完整步骤，请参阅 [KV Router A/B 基准测试指南](../../benchmarks/kv-router-ab-testing.md)。

### 配合高级配置的示例

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-deployment
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      envs:
        - name: DYN_ROUTER_MODE
          value: kv
        - name: DYN_ROUTER_TEMPERATURE
          value: "0.5"  # Add some randomness to prevent worker saturation
        - name: DYN_ROUTER_KV_OVERLAP_SCORE_WEIGHT
          value: "1.5"  # Prioritize TTFT over ITL
        - name: DYN_KV_CACHE_BLOCK_SIZE
          value: "16"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
```

### 备选方案：在 K8s 中使用命令行参数

也可以直接在容器命令中传入 CLI 参数：

```yaml
extraPodSpec:
  mainContainer:
    image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
    command:
      - /bin/sh
      - -c
    args:
      - "python3 -m dynamo.frontend --router-mode kv --router-temperature 0.5 --http-port 8000"
```

**建议：** 使用环境变量来管理配置更便于维护，也更符合 Dynamo 在 K8s 上的常用模式。

## 路由模式

`KvRouter` 提供多种使用模式，可根据控制力需求选择：

### 1. 自动路由（推荐）
直接调用 `generate()` 让 router 处理一切：
```python
stream = await router.generate(token_ids=tokens, model="model-name")
```
- **适用场景**：绝大多数情况
- **Router 自动完成**：选择最优 worker、更新状态、路由请求、跟踪生命周期

### 2. 手动状态管理（进阶）
使用 `best_worker(request_id=...)` 选择并跟踪 worker，自行管理请求：
```python
worker_id, _dp_rank, overlap = await router.best_worker(
    tokens,
    request_id="req-123",
    update_indexer=True,  # needed for approximate mode (use_kv_events=False)
)
response = await client.generate(tokens, request_id="req-123")
# await anext(response)  # Get first token
await router.mark_prefill_complete("req-123")  # After first token
# async for _ in response:  # Continue generating
#     ...
await router.free("req-123")  # After completion
```
- **适用场景**：需要自定义请求处理同时保留 router 状态跟踪
- **要求**：在正确的生命周期点调用 `mark_prefill_complete()` 与 `free()`
- **近似模式**：当 `use_kv_events=False` 时传入 `update_indexer=True`，让 router 从手动 worker 选择中学习
- **注意**：错误的生命周期管理会降低负载均衡的准确性

### 3. 分层 Router 探测
不更新状态地查询，再通过选定的 router 路由：
```python
# Probe multiple routers without updating state
worker_id_1, dp_rank, overlap_1 = await router_1.best_worker(tokens)  # No request_id
worker_id_2, dp_rank, overlap_2 = await router_2.best_worker(tokens)

# Pick the best router and corresponding worker based on results
if overlap_1 > overlap_2:
    chosen_router, chosen_worker = router_1, worker_id_1
else:
    chosen_router, chosen_worker = router_2, worker_id_2
stream = await chosen_router.generate(tokens, model="model-name", worker_id=chosen_worker)
```
- **适用场景**：多层部署（例如 Envoy Gateway 路由到多组 router）
- **优势**：在最终选定 router 之前可以查询多个 router

### 4. 自定义负载路由
使用 `get_potential_loads()` 实现自定义路由逻辑：
```python
loads = await router.get_potential_loads(tokens)
# Apply custom logic (e.g., weighted scoring, constraints)
best_worker = min(loads, key=lambda x: custom_cost_fn(x))
stream = await router.generate(tokens, model="model-name", worker_id=best_worker['worker_id'])
```
- **适用场景**：超越内置代价函数的自定义优化策略
- **优势**：完全掌控 worker 选择逻辑
- **另见**：下文 “Custom Routing Example: Minimizing TTFT” 中的详细示例

所有模式都支持 `router_config_override`，无需重建 router 即可在请求级别调整路由行为。

## 自定义路由示例：最小化 TTFT

下面这个示例使用 `get_potential_loads()` 来实现自定义路由：通过选择 prefill 工作量最小的 worker 来最小化首 token 时间（TTFT）：

```python
import asyncio
from dynamo.runtime import DistributedRuntime
from dynamo.llm import KvRouter, KvRouterConfig

async def minimize_ttft_routing():
    # Setup router
    loop = asyncio.get_running_loop()
    runtime = DistributedRuntime(loop, "etcd", "nats")
    endpoint = runtime.endpoint("dynamo.backend.generate")

    router = KvRouter(
        endpoint=endpoint,
        block_size=16,
        kv_router_config=KvRouterConfig()
    )

    # Your input tokens
    token_ids = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

    # Get potential loads for all workers
    potential_loads = await router.get_potential_loads(token_ids)

    # Find worker with minimum prefill tokens (best for TTFT)
    best_worker = min(potential_loads, key=lambda x: x['potential_prefill_tokens'])

    print(f"Worker loads: {potential_loads}")
    print(f"Selected worker {best_worker['worker_id']} with {best_worker['potential_prefill_tokens']} prefill tokens")

    # Route directly to the selected worker
    stream = await router.generate(
        token_ids=token_ids,
        model="meta-llama/Llama-2-7b-hf",
        worker_id=best_worker['worker_id'],  # Force routing to optimal worker
        stop_conditions={"max_tokens": 20}
    )

    # Process response
    async for response in stream:
        if isinstance(response, dict) and "token_ids" in response:
            print(f"Generated tokens: {response['token_ids']}")

if __name__ == "__main__":
    asyncio.run(minimize_ttft_routing())
```

这种方式让你完全掌控路由决策，可以根据自己的需求针对不同指标做优化。例如：

- **最小化 TTFT**：选择 `potential_prefill_tokens` 最小的 worker
- **最大化缓存复用**：使用 `best_worker()`，它同时考虑 prefill 与 decode 负载
- **均衡负载**：把 `potential_prefill_tokens` 与 `potential_decode_blocks` 一起纳入考虑

架构细节与代价函数算法，请参见 [Router 设计](../../design-docs/router-design.md)。

## 自定义引擎的 KV 事件发布

关于如何为自定义推理引擎实现 KV 事件发布的完整文档，请参阅独立的 [自定义引擎 KV 事件发布](../../integrations/kv-events-custom-engines.md) 指南。其中涵盖：

- **直接发布**：调用 `publish_stored()` / `publish_removed()` 通过 Dynamo 事件平面推送事件
- **ZMQ 中继**：对于通过 ZMQ 发出原始 KV 事件的引擎（例如 SGLang 与 vLLM），可使用同一个 `KvEventPublisher` 订阅 ZMQ socket 并自动转发事件
- API 参考、事件结构、ZMQ 通信格式与最佳实践

## 全局 Router（分层路由）

对于具有多个 worker 池的部署，**全局 Router**（Global Router）通过位于前端与本地 router 之间，实现分层路由。它根据可配置策略为每个请求选择合适的池，支持那些不同池针对不同负载特性进行调优的解耦（disaggregated）拓扑。

- **组件细节**：[`components/src/dynamo/global_router/`](https://github.com/ai-dynamo/dynamo/tree/main/components/src/dynamo/global_router/)
- **示例**：[`examples/global_planner/`](https://github.com/ai-dynamo/dynamo/tree/main/examples/global_planner/)

## 参见

- **[Router README](README.md)**：KV Router 快速上手指南
- **[配置与调优](router-configuration.md)**：Router 启动参数与生产部署
- **[Router 设计](../../design-docs/router-design.md)**：架构细节与事件传输模式
