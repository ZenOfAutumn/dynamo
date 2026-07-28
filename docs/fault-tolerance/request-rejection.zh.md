---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Request Rejection
---

本文档介绍 Dynamo 如何实现请求拒绝（request rejection），用于在高负载下防止系统过载并维持服务稳定。

## 概览

请求拒绝（也称为 load shedding，负载卸除）是一种容错机制：当 worker 过载时主动拒绝新的请求。这样可以避免：

- 资源耗尽导致的级联失败
- 所有请求的延迟劣化
- GPU worker 上发生 OOM

当所有 worker 都超过其配置的 busy 阈值时，新请求会收到 HTTP 503（Service Unavailable）响应，提示客户端稍后重试。

## 架构

```
                                    ┌─────────────────┐
                                    │  Worker Monitor │
                                    │  (Background)   │
                                    └────────┬────────┘
                                             │ Updates busy list
                                             ▼
┌──────────┐    ┌──────────┐    ┌─────────────────────┐    ┌──────────┐
│  Client  │───▶│ Frontend │───▶│    Push Router      │───▶│  Worker  │
└──────────┘    └──────────┘    │ (checks busy list)  │    └──────────┘
                                └─────────────────────┘
                                         │
                                         │ If all workers busy
                                         ▼
                                ┌─────────────────────┐
                                │   HTTP 503 Error    │
                                │ "All workers busy"  │
                                └─────────────────────┘
```

## 配置

### 前端（frontend）参数

启动 frontend 时配置 busy 阈值：

```bash
python -m dynamo.frontend \
    --active-decode-blocks-threshold 0.85 \
    --active-prefill-tokens-threshold 10000
```

| 参数 | 类型 | 描述 |
|----------|------|-------------|
| `--active-decode-blocks-threshold` | float (0.0-1.0) | KV 缓存（KV cache）块利用率阈值 |
| `--active-prefill-tokens-threshold` | int | 预填充（prefill） token 数阈值 |

### 通过 API 动态配置

阈值可以在运行时通过 `/busy_threshold` endpoint 调整：

#### 设置阈值

```bash
curl -X POST http://localhost:8000/busy_threshold \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "active_decode_blocks_threshold": 0.85,
    "active_prefill_tokens_threshold": 10000
  }'
```

#### 获取当前阈值

```bash
curl http://localhost:8000/busy_threshold
```

响应：
```json
{
  "thresholds": [
    {
      "model": "Qwen/Qwen3-0.6B",
      "active_decode_blocks_threshold": 0.85,
      "active_prefill_tokens_threshold": 10000
    }
  ]
}
```

## Busy 检测逻辑

worker 是否被标记为 "busy" 由双阈值系统决定。**任一**阈值被超过，worker 都会被认为 busy。

### KV 缓存块阈值

监控 KV 缓存块的使用百分比：

```
busy = active_decode_blocks / kv_total_blocks > threshold
```

示例：当 `active_decode_blocks_threshold=0.85` 时，使用了 87% KV 缓存块的 worker 会被标记为 busy。

### Prefill token 阈值

监控当前正在预填充的 token 数：

```
busy = active_prefill_tokens > threshold
```

示例：当 `active_prefill_tokens_threshold=10000` 时，正在预填充 12,000 个 token 的 worker 会被标记为 busy。

### Data-Parallel Rank 聚合

对于带有多个 data-parallel rank（tensor parallelism）的 worker，仅当**所有** rank 都 busy 时该 worker 才被标记为 busy：

```python
def is_busy(worker):
    return all(rank.is_busy() for rank in worker.dp_ranks)
```

这避免了仅有部分 rank 临时高负载时的误判。

## Worker 负载监控

`KvWorkerMonitor` 作为后台任务运行，负责：

1. 订阅来自 worker 的 KV 缓存指标（metrics）事件
2. 维护每个 worker 实例的负载状态
3. 当指标变化时重新计算哪些实例为 busy
4. 更新路由（router）的当前 busy 列表

### 收集的指标

worker 发布以下指标用于监控：

| 指标 | 描述 |
|--------|-------------|
| `active_decode_blocks` | 当前正在使用的 KV 缓存块数 |
| `kv_total_blocks` | 可用的 KV 缓存块总数 |
| `active_prefill_tokens` | 当前正在预填充的 token 数 |

## 拒绝行为

### 请求流

1. 请求到达 frontend
2. push router 检查是否配置了 busy 阈值
3. 若已配置，router 获取空闲（非 busy）实例列表
4. 若没有空闲实例（但确实有注册的实例）：
   - 请求被以 `PipelineError::ServiceOverloaded` 拒绝
   - 向客户端返回 HTTP 503

### 错误响应

请求被拒绝时，客户端收到：

```http
HTTP/1.1 503 Service Unavailable
Content-Type: application/json

{
  "message": "Service temporarily unavailable: All workers are busy, please retry later",
  "type": "service_unavailable",
  "code": 503
}
```

### 客户端重试策略

客户端在收到 503 时应实现指数退避：

```python
import time
import random

def send_with_retry(request, max_retries=5):
    for attempt in range(max_retries):
        response = send_request(request)
        if response.status_code != 503:
            return response

        # Exponential backoff with jitter
        wait_time = min(60, (2 ** attempt) + random.uniform(0, 1))
        time.sleep(wait_time)

    raise Exception("Max retries exceeded")
```

## 监控

### Prometheus 指标

通过以下指标观察拒绝行为：

- `dynamo_frontend_model_rejection_total`：累计因资源耗尽而被拒绝的请求数
  - 标签：
    - `model`：被服务的模型名
    - `endpoint`：收到请求的 API endpoint（如 `chat_completions`、`completions`、`embeddings`）
  - 当 router 因所有 worker 都 busy 而返回 `ResourceExhausted` 错误时，该指标会自增。被拒绝的请求会以 HTTP 503 暴露给客户端。

**示例指标输出：**
```text
dynamo_frontend_model_rejection_total{endpoint="chat_completions",model="Qwen/Qwen3-0.6B"} 32
dynamo_frontend_model_rejection_total{endpoint="completions",model="Qwen/Qwen3-0.6B"} 5
```

**端点：** 在 frontend HTTP 服务的 `/metrics` 上提供。

## 调优阈值

### 保守设置（注重延迟）

适合优先保证低延迟的应用：

```bash
--active-decode-blocks-threshold 0.70
--active-prefill-tokens-threshold 5000
```

- 拒绝得更早，避免 worker 完全打满
- 维持更低的队列深度
- 更好的尾部延迟

### 激进设置（注重吞吐）

适合优先保证吞吐的应用：

```bash
--active-decode-blocks-threshold 0.95
--active-prefill-tokens-threshold 20000
```

- 允许更高的 worker 利用率
- 可能加剧延迟波动
- 更好的整体吞吐

### 关闭（不拒绝）

要完全关闭请求拒绝：

```bash
# Simply don't set the threshold arguments
python -m dynamo.frontend
```

未配置阈值时，所有请求都会被接受，无视 worker 负载。

## 最佳实践

### 1. 从保守开始，再逐步调优

从保守阈值起步，根据观察到的行为再上调：

```bash
# Start here
--active-decode-blocks-threshold 0.75

# Increase if rejection rate is too high
--active-decode-blocks-threshold 0.85
```

### 2. 启用前先观测

设置阈值前先观察 worker 负载模式：

```bash
# Watch KV cache utilization
watch -n 1 'curl -s localhost:8000/metrics | grep kv_blocks'
```

### 3. 解耦（disaggregated）服务时同时使用两个阈值

在解耦部署中：
- 对 prefill worker 使用 `active_prefill_tokens_threshold`
- 对 decode worker 使用 `active_decode_blocks_threshold`

### 4. 与自动扩缩容协同

使用 Kubernetes HPA 时，确保拒绝阈值在自动扩容之前触发：

```yaml
# HPA triggers at 70% utilization
# Rejection at 85% provides buffer
--active-decode-blocks-threshold 0.85
```

## 相关文档

- [健康检查](../observability/health-checks.md) - worker 健康监控
- [指标](../observability/metrics.md) - 可用的 Prometheus 指标
- [请求迁移](request-migration.md) - 处理失败的请求
