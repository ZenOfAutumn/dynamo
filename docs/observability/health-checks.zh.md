---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Health Checks
---

## 概览

Dynamo 为每个组件提供健康检查（health check）与存活（liveness）的 HTTP 端点，可用于在 Kubernetes 等编排框架中配置 startup、liveness 和 readiness 探针（probe）。

## 环境变量

| 变量 | 说明 | 默认值 | 示例 |
|----------|-------------|---------|---------|
| `DYN_SYSTEM_PORT` | 系统状态服务（system status server）端口 | `-1`（禁用） | `8081` |
| `DYN_SYSTEM_STARTING_HEALTH_STATUS` | 初始健康状态 | `notready` | `ready`、`notready` |
| `DYN_SYSTEM_HEALTH_PATH` | 自定义 health 端点路径 | `/health` | `/custom/health` |
| `DYN_SYSTEM_LIVE_PATH` | 自定义 liveness 端点路径 | `/live` | `/custom/live` |
| `DYN_SYSTEM_USE_ENDPOINT_HEALTH_STATUS` | ready 状态所需的端点 | 无 | `["generate"]` |
| `DYN_HEALTH_CHECK_ENABLED` | 启用 canary 健康检查 | `false`（K8s 中：`true`） | `true`、`false` |
| `DYN_CANARY_WAIT_TIME` | 发送 canary 健康检查前的等待秒数 | `10` | `5`、`30` |
| `DYN_HEALTH_CHECK_REQUEST_TIMEOUT` | 健康检查请求超时（秒） | `3` | `5`、`10` |

## 快速开始

启用健康检查并查询端点：

```bash
# Start your Dynamo components (default port 8000, override with --http-port or DYN_HTTP_PORT env var)
python -m dynamo.frontend &

# Enable system status server on port 8081
DYN_SYSTEM_PORT=8081 python -m dynamo.vllm --model Qwen/Qwen3-0.6B --enforce-eager &
```

查看健康状态：

```bash
# Frontend health (port 8000)
curl -s localhost:8000/health | jq

# Worker health (port 8081)
curl -s localhost:8081/health | jq
```

## 前端（Frontend）存活检查

只要服务在运行，前端 liveness 端点就会报告 `live` 状态。

<Note>
前端存活仅取决于前端服务自身，不依赖 worker 的健康或存活状态。
</Note>

### 请求示例

```
curl -s localhost:8080/live -q | jq
```

### 响应示例

```
{
  "message": "Service is live",
  "status": "live"
}
```

## 前端健康检查

只要服务在运行，前端 health 端点就会报告 `healthy` 状态。一旦有 worker 注册，`health` 端点还会列出已注册的端点与实例。

<Note>
前端存活仅取决于前端服务自身，不依赖 worker 的健康或存活状态。
</Note>

### 请求示例

```
curl -v localhost:8080/health -q | jq
```

### 响应示例

worker 注册前：

```
HTTP/1.1 200 OK
content-type: application/json
content-length: 72
date: Wed, 03 Sep 2025 13:31:44 GMT

{
  "instances": [],
  "message": "No endpoints available",
  "status": "unhealthy"
}
```

worker 注册后：

```
HTTP/1.1 200 OK
content-type: application/json
content-length: 609
date: Wed, 03 Sep 2025 13:32:03 GMT

{
  "endpoints": [
    "dyn://dynamo.backend.generate"
  ],
  "instances": [
    {
      "component": "backend",
      "endpoint": "clear_kv_blocks",
      "instance_id": 7587888160958628000,
      "namespace": "dynamo",
      "transport": {
        "nats_tcp": "dynamo_backend.clear_kv_blocks-694d98147d54be25"
      }
    },
    {
      "component": "backend",
      "endpoint": "generate",
      "instance_id": 7587888160958628000,
      "namespace": "dynamo",
      "transport": {
        "nats_tcp": "dynamo_backend.generate-694d98147d54be25"
      }
    },
    {
      "component": "backend",
      "endpoint": "load_metrics",
      "instance_id": 7587888160958628000,
      "namespace": "dynamo",
      "transport": {
        "nats_tcp": "dynamo_backend.load_metrics-694d98147d54be25"
      }
    }
  ],
  "status": "healthy"
}
```

## Worker 存活与健康检查

非前端组件的健康检查通过环境变量按需启用。启用后，可设置初始状态以及在组件被声明为 `ready` 之前必须已服务的端点集合。

一旦 `DYN_SYSTEM_USE_ENDPOINT_HEALTH_STATUS` 中声明的所有端点都已就绪，组件即转入 `ready` 状态，直至关闭。在初始化阶段，端点返回 `HTTP/1.1 503 Service Unavailable`；ready 后返回 `HTTP/1.1 200 OK`。

<Note>
/live 与 /ready 返回相同信息
</Note>

### 环境变量示例

```
export DYN_SYSTEM_PORT=9090
export DYN_SYSTEM_STARTING_HEALTH_STATUS="notready"
export DYN_SYSTEM_USE_ENDPOINT_HEALTH_STATUS="[\"generate\"]"
```

#### 请求示例

```
curl -v localhost:9090/health | jq
```

#### 响应示例
端点尚未提供服务时：

```
HTTP/1.1 503 Service Unavailable
content-type: text/plain; charset=utf-8
content-length: 96
date: Wed, 03 Sep 2025 13:42:39 GMT

{
  "endpoints": {
    "generate": "notready"
  },
  "status": "notready",
  "uptime": {
    "nanos": 313803539,
    "secs": 12
  }
}
```

端点已提供服务后：

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 139
date: Wed, 03 Sep 2025 13:42:45 GMT

{
  "endpoints": {
    "clear_kv_blocks": "ready",
    "generate": "ready",
    "load_metrics": "ready"
  },
  "status": "ready",
  "uptime": {
    "nanos": 356504530,
    "secs": 18
  }
}
```

## Canary 健康检查（主动监控）

除上述 HTTP 端点外，Dynamo 还内置 **canary 健康检查** 系统，主动监控 worker 端点。

### 概览

canary 健康检查系统：
- 通过定期向 worker 端点发送测试请求来 **监控端点健康**
- 仅在 **空闲期间生效** — 若有持续流量，则跳过健康检查以避免开销
- 在 Kubernetes 部署中由 operator **自动启用**
- 在本地/开发环境中 **默认禁用**

### 工作原理

1. **空闲检测**：当端点在可配置的等待时间（默认 10 秒）内无活动时，触发一次 canary 健康检查
2. **健康检查请求**：向端点发送一个轻量级测试请求，使用最小负载（生成 1 个 token）
3. **活动重置计时器**：若有正常请求到达，则 canary 计时器复位，不发送健康检查
4. **超时处理**：若健康检查在超时时间内（默认 3 秒）未返回，则该端点被标记为不健康

### 配置

#### Kubernetes 中（默认启用）

健康检查由 Dynamo operator 自动启用，无需额外配置。

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-deployment
spec:
  services:
    VllmWorker:
      componentType: worker
      replicas: 2
      # Health checks automatically enabled by operator
```

#### 本地/开发环境（默认禁用）

在本地启用健康检查：

```bash
# Enable health checks
export DYN_HEALTH_CHECK_ENABLED=true

# Optional: Customize timing
export DYN_CANARY_WAIT_TIME=5  # Wait 5 seconds before sending health check
export DYN_HEALTH_CHECK_REQUEST_TIMEOUT=5  # 5 second timeout

# Start worker
python -m dynamo.vllm --model Qwen/Qwen3-0.6B
```

#### 配置选项

| 环境变量 | 说明 | 默认值 | 备注 |
|---------------------|-------------|---------|-------|
| `DYN_HEALTH_CHECK_ENABLED` | 启用/禁用 canary 健康检查 | `false`（K8s：`true`） | K8s 中自动设为 `true` |
| `DYN_CANARY_WAIT_TIME` | 空闲多少秒后发送健康检查 | `10` | 越小检查越频繁 |
| `DYN_HEALTH_CHECK_REQUEST_TIMEOUT` | 等待健康检查响应的最长秒数 | `3` | 越大对慢响应容忍越高 |

### 健康检查负载（Payload）

每个后端定义自己的最小健康检查负载：

- **vLLM**：使用最小化采样选项的单 token 生成
- **TensorRT-LLM**：带 BOS token ID 的单 token
- **SGLang**：单 token 生成请求

这些负载的设计目标：
- 快速完成（通常 \< 100ms）
- 最小化 GPU 开销
- 验证完整推理（inference）栈在工作

### 观察健康检查

启用后，你将看到类似日志：

```
INFO Health check manager started (canary_wait_time: 10s, request_timeout: 3s)
INFO Spawned health check task for endpoint: generate
INFO Canary timer expired for generate, sending health check
INFO Health check successful for generate
```

端点失败时：

```
WARN Health check timeout for generate
ERROR Health check request failed for generate: connection refused
```

### 何时使用 canary 健康检查

**生产（Kubernetes）启用：**
- ✅ 在影响用户流量前发现不健康的 worker
- ✅ 加快故障检测与恢复
- ✅ 持续监控 worker 可用性

**开发禁用：**
- ✅ 减少调试时的日志噪声
- ✅ 在不需要时避免开销
- ✅ 简化本地测试

### 故障排查

**健康检查超时：**
- 增大 `DYN_HEALTH_CHECK_REQUEST_TIMEOUT`
- 检查 worker 日志中的错误
- 验证网络连通性

**健康检查日志过多：**
- 增大 `DYN_CANARY_WAIT_TIME` 以降低频率
- 或在开发中通过 `DYN_HEALTH_CHECK_ENABLED=false` 禁用

**健康检查未运行：**
- 确认设置了 `DYN_HEALTH_CHECK_ENABLED=true`
- 检查 `DYN_SYSTEM_USE_ENDPOINT_HEALTH_STATUS` 是否包含该端点
- 确认 worker 已在提供该端点

## 相关文档

- [分布式运行时架构](../design-docs/distributed-runtime.md)
- [Dynamo 架构概览](../design-docs/architecture.md)
- [后端指南](../development/backend-guide.md)
