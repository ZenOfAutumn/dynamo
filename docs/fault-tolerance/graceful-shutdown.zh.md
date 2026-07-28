---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 优雅关停
---

本文档描述 Dynamo 各组件如何处理关停信号，以确保进行中的请求能够顺利完成、资源被正确清理。

## 概述

Dynamo 的优雅关停（graceful shutdown）确保：

1. **路由迅速停止** - 端点首先从发现服务（discovery）中注销
2. **进行中的请求可以完成** - worker 在短暂的宽限期内继续提供服务
3. **端点排空（drain）** - 宽限期结束后，端点被失效，并可选择等待进行中的工作完成
4. **资源被清理** - 释放引擎、连接和临时文件
5. **Pod 干净重启** - 退出码为 Kubernetes 提供合适的重启信号

## 信号处理

所有 Dynamo 组件均处理 Unix 信号以实现优雅关停：

| 信号 | 触发 | 行为 |
|--------|---------|----------|
| `SIGTERM` | Kubernetes pod 终止 | 启动优雅关停 |
| `SIGINT` | Ctrl+C / 手动中断 | 启动优雅关停 |

### 实现

每个组件在启动时注册信号处理器：

```python
def signal_handler():
    asyncio.create_task(graceful_shutdown(runtime, endpoints))

for sig in (signal.SIGTERM, signal.SIGINT):
    loop.add_signal_handler(sig, signal_handler)
```

`graceful_shutdown()` 函数：
1. 记录关停信号
2. 从发现服务中注销所有端点
3. 等待可配置的宽限期（`DYN_GRACEFUL_SHUTDOWN_GRACE_PERIOD_SECS`，默认 5s）
4. 调用 `runtime.shutdown()` 使端点失效，停止接受新请求
5. 等待进行中的请求（基于每个端点的 `graceful_shutdown` 设置）
6. 返回以便后续清理

## 端点排空

宽限期结束后，`runtime.shutdown()` 使端点失效，不再接受新请求。进行中请求的处理方式取决于端点注册时的 `graceful_shutdown` 参数。

### 配置

注册端点时，`graceful_shutdown` 参数控制排空行为：

```python
generate_endpoint.serve_endpoint(
    handler.generate,
    graceful_shutdown=True,  # 等待所有请求完成
    metrics_labels=[("model", model_name)],
    health_check_payload=health_check_payload,
)
```

| `graceful_shutdown` | 行为 |
|---------------------|----------|
| `True` | 在返回前等待所有进行中的请求完成 |
| `False` | 立即返回，不等待请求 |

### 各组件的具体行为

| 组件 | 默认行为 | 理由 |
|-----------|------------------|-----------|
| **前端（Frontend）** | 不适用（HTTP 服务器） | HTTP 服务器自行处理关停 |
| **预填充（Prefill）Worker** | `graceful_shutdown=True` | prefill 必须完成以避免计算浪费 |
| **解码（Decode）Worker** | `graceful_shutdown=True` | decode 应当完成以避免计算浪费 |
| **路由（Router）** | `graceful_shutdown=True` | 确保路由决策完成 |

### 与请求迁移的集成

后端 worker 始终使用 `graceful_shutdown=True`，即在引擎被停止前等待进行中的请求完成。请求迁移在**前端**层通过 `--migration-limit` 配置：

- 当前端启用迁移后，来自失败 worker 的断开流会自动在健康 worker 上重试
- worker 不需要了解迁移配置 - 它们只需完成自己的工作或对未完成的流进行信号通知
- 关于迁移的工作机制详见 [Request Migration Architecture](./request-migration.md)

## 资源清理

端点排空后，组件在 `finally` 块中清理资源：

### vLLM Worker 清理

```python
finally:
    logger.debug("Cleaning up worker")
    handler.cleanup()
```

handler 的 `cleanup()` 方法：
- 删除临时目录（LoRA 适配器等）
- 释放引擎资源

### SGLang Worker 清理

```python
def cleanup(self) -> None:
    # 取消未完成的消费任务
    for task in self._consume_tasks:
        if not task.done():
            task.cancel()
    self._consume_tasks.clear()

    # 关停引擎
    self.engine.shutdown()
```

### TensorRT-LLM Worker 清理

```python
async def cleanup(self):
    if self._llm:
        try:
            self._llm.shutdown()
        except Exception as e:
            logging.error(f"Error during cleanup: {e}")
        finally:
            self._llm = None
```

## 由错误触发的关停

当出现致命错误时，worker 可主动启动优雅关停：

### 引擎健康监控（vLLM）

`VllmEngineMonitor` 持续检查引擎健康：

```python
async def _check_engine_health(self):
    while True:
        try:
            await self.engine_client.check_health()
            await asyncio.sleep(HEALTH_CHECK_INTERVAL)  # 2 秒
        except EngineDeadError as e:
            logger.error(f"Health check failed: {e}")
            self._shutdown_engine()
            self.runtime.shutdown()
            os._exit(1)
```

配置：
- `HEALTH_CHECK_INTERVAL`：每次检查之间 2 秒
- `ENGINE_SHUTDOWN_TIMEOUT`：引擎关停最长 30 秒

### 致命错误处理（TensorRT-LLM）

```python
async def _initiate_shutdown(self, error: Exception):
    logging.warning(f"Initiating graceful shutdown due to: {error}")

    try:
        if self.runtime:
            self.runtime.shutdown()
        if self.engine:
            await self.engine.cleanup()
    except Exception as cleanup_error:
        logging.error(f"Error during graceful shutdown: {cleanup_error}")
    finally:
        logging.critical("Forcing process exit for restart")
        os._exit(1)
```

## 与 Kubernetes 的集成

### Pod 终止流程

1. Kubernetes 向 pod 发送 `SIGTERM`
2. Dynamo 启动优雅关停
3. Pod 拥有 `terminationGracePeriodSeconds` 来完成（默认：30s）
4. 如果未终止，Kubernetes 发送 `SIGKILL`

### 推荐配置

对于生产部署，配置足够的终止宽限期：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
spec:
  services:
    VllmWorker:
      extraPodSpec:
        terminationGracePeriodSeconds: 60  # 为请求排空预留时间
```

### 健康检查集成

Kubernetes 通过健康端点判断 pod 是否就绪：

- **关停期间**：端点变为不可用
- **就绪探针失败**：流量停止路由到该 pod
- **优雅排空**：已有请求完成

## 最佳实践

### 1. 设置合适的宽限期

把 `terminationGracePeriodSeconds` 与预期的请求完成时间相匹配：
- 短请求（\< 10s）：30s 宽限期
- 长生成（> 30s）：120s+ 宽限期

### 2. 启用请求迁移

在前端启用迁移以便在 worker 关停时恢复请求：

```bash
python3 -m dynamo.frontend ... --migration-limit 3  # 最多允许 3 次迁移
```

这样前端可以自动在健康 worker 上重试断开的流。

### 3. 监控关停指标

通过日志跟踪关停行为：

```
INFO  Received shutdown signal, shutting down DistributedRuntime
INFO  DistributedRuntime shutdown complete
DEBUG Cleaning up worker
```

### 4. 处理清理错误

确保清理方法优雅地处理错误：

```python
def cleanup(self):
    for resource in self.resources:
        try:
            resource.cleanup()
        except Exception as e:
            logger.warning(f"Cleanup failed: {e}")
            # 继续清理其他资源
```

## 相关文档

- [Request Migration](request-migration.md) - 关停期间请求如何迁移
- [Request Cancellation](request-cancellation.md) - 取消进行中的请求
- [Health Checks](../observability/health-checks.md) - 存活与就绪探针
