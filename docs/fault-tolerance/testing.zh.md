---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Testing
---

本文档介绍用于验证 Dynamo 容错机制的测试基础设施。该测试框架支持请求取消、请求迁移、etcd HA 以及硬件故障注入等场景。

## 概览

Dynamo 的容错测试套件位于 `tests/fault_tolerance/`，包含：

| 测试类别 | 位置 | 用途 |
|---------------|----------|---------|
| 取消（Cancellation） | `cancellation/` | 在请求进行中取消请求 |
| 迁移（Migration） | `migration/` | worker 失败时的请求迁移 |
| etcd HA | `etcd_ha/` | etcd 故障切换与恢复 |
| 硬件（Hardware） | `hardware/` | GPU 与网络故障注入 |
| 部署（Deployment） | `deploy/` | 端到端部署测试 |

## 测试目录结构

```
tests/fault_tolerance/
├── cancellation/
│   ├── test_vllm.py
│   ├── test_trtllm.py
│   ├── test_sglang.py
│   └── utils.py
├── migration/
│   ├── test_vllm.py
│   ├── test_trtllm.py
│   ├── test_sglang.py
│   └── utils.py
├── etcd_ha/
│   ├── test_vllm.py
│   ├── test_trtllm.py
│   ├── test_sglang.py
│   └── utils.py
├── hardware/
│   └── fault_injection_service/
│       ├── api_service/
│       └── agents/
├── deploy/
│   ├── test_deployment.py
│   ├── scenarios.py
│   ├── base_checker.py
│   └── ...
└── client.py
```

## 请求取消测试

测试进行中的请求是否能被正确取消。

### 运行取消测试

```bash
# Run all cancellation tests
pytest tests/fault_tolerance/cancellation/ -v

# Run for specific backend
pytest tests/fault_tolerance/cancellation/test_vllm.py -v
```

### 取消测试工具

`cancellation/utils.py` 模块提供：

#### CancellableRequest

通过 TCP socket 操作实现线程安全的请求取消：

```python
from tests.fault_tolerance.cancellation.utils import CancellableRequest

request = CancellableRequest()

# Send request in separate thread
thread = Thread(target=send_request, args=(request,))
thread.start()

# Cancel after some time
time.sleep(1)
request.cancel()  # Closes underlying socket
```

#### send_completion_request / send_chat_completion_request

发送可取消的 completion 请求：

```python
from tests.fault_tolerance.cancellation.utils import (
    send_completion_request,
    send_chat_completion_request
)

# Non-streaming
response = send_completion_request(
    base_url="http://localhost:8000",
    model="Qwen/Qwen3-0.6B",
    prompt="Hello, world!",
    max_tokens=100
)

# Streaming with cancellation
responses = send_chat_completion_request(
    base_url="http://localhost:8000",
    model="Qwen/Qwen3-0.6B",
    messages=[{"role": "user", "content": "Hello!"}],
    stream=True,
    cancellable_request=request
)
```

#### poll_for_pattern

等待日志中出现特定模式：

```python
from tests.fault_tolerance.cancellation.utils import poll_for_pattern

# Wait for cancellation confirmation
found = poll_for_pattern(
    log_file="/var/log/dynamo/worker.log",
    pattern="Request cancelled",
    timeout=30,
    interval=0.5
)
```

## 迁移测试

测试在发生失败时，请求能否迁移到健康的 worker 上。

### 运行迁移测试

```bash
# Run all migration tests
pytest tests/fault_tolerance/migration/ -v

# Run for specific backend
pytest tests/fault_tolerance/migration/test_vllm.py -v
```

### 迁移测试工具

`migration/utils.py` 模块提供：

- 可配置 request plane 的 frontend 封装器
- 用于迁移场景的长时请求生成
- 用于受控测试的健康检查关闭能力

### 迁移测试示例

```python
def test_migration_on_worker_failure():
    # Start deployment with 2 workers
    deployment = start_deployment(workers=2)

    # Send long-running request
    request_thread = spawn_long_request(max_tokens=1000)

    # Kill one worker mid-generation
    kill_worker(deployment.workers[0])

    # Verify request completes on remaining worker
    response = request_thread.join()
    assert response.status_code == 200
    assert len(response.tokens) > 0
```

## etcd HA 测试

测试 etcd 故障与恢复期间系统的行为。

### 运行 etcd HA 测试

```bash
pytest tests/fault_tolerance/etcd_ha/ -v
```

### 测试场景

- **Leader 故障切换**：etcd leader 节点失败，集群选举新 leader
- **网络分区**：etcd 节点不可达
- **恢复**：etcd 恢复可用后，系统自我恢复

## 硬件故障注入

故障注入服务支持在模拟的硬件故障下进行测试。

### 故障注入服务

位于 `tests/fault_tolerance/hardware/fault_injection_service/`，这是一个用于编排故障注入的 FastAPI 服务：

```bash
# Start the fault injection service
cd tests/fault_tolerance/hardware/fault_injection_service
python -m api_service.main
```

### 支持的故障类型

#### GPU 故障

| 故障类型 | 描述 |
|------------|-------------|
| `XID_ERROR` | 模拟 GPU XID 错误（多种代码） |
| `THROTTLE` | GPU 热降频 |
| `MEMORY_PRESSURE` | GPU 显存耗尽 |
| `OVERHEAT` | GPU 过热 |
| `COMPUTE_OVERLOAD` | GPU 计算饱和 |

#### 网络故障

| 故障类型 | 描述 |
|------------|-------------|
| `FRONTEND_WORKER` | frontend 与 worker 之间的分区 |
| `WORKER_NATS` | worker 与 NATS 之间的分区 |
| `WORKER_WORKER` | worker 之间的分区 |
| `CUSTOM` | 自定义网络分区 |

### 故障注入 API

#### 注入 GPU 故障

```bash
curl -X POST http://localhost:8080/api/v1/faults/gpu/inject \
  -H "Content-Type: application/json" \
  -d '{
    "target_pod": "vllm-worker-0",
    "fault_type": "XID_ERROR",
    "severity": "HIGH"
  }'
```

#### 注入特定 XID 错误

```bash
# Inject XID 79 (GPU memory page fault)
curl -X POST http://localhost:8080/api/v1/faults/gpu/inject/xid-79 \
  -H "Content-Type: application/json" \
  -d '{"target_pod": "vllm-worker-0"}'
```

支持的 XID 代码：43、48、74、79、94、95、119、120

#### 注入网络分区

```bash
curl -X POST http://localhost:8080/api/v1/faults/network/inject \
  -H "Content-Type: application/json" \
  -d '{
    "partition_type": "FRONTEND_WORKER",
    "duration_seconds": 30
  }'
```

#### 从故障中恢复

```bash
curl -X POST http://localhost:8080/api/v1/faults/{fault_id}/recover
```

#### 列出当前故障

```bash
curl http://localhost:8080/api/v1/faults
```

### GPU 故障注入器 Agent

GPU 故障注入器作为 DaemonSet 在 worker 节点上运行：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-fault-injector
spec:
  selector:
    matchLabels:
      app: gpu-fault-injector
  template:
    spec:
      containers:
      - name: agent
        image: dynamo/gpu-fault-injector:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: dev
          mountPath: /dev
```

agent 通过向 `/dev/kmsg` 注入伪造的 XID 消息，触发 NVSentinel 检测。

## 部署测试框架

`deploy/` 目录包含一个端到端的测试框架。

### 测试阶段

测试分三个阶段进行：

| 阶段 | 描述 |
|-------|-------------|
| `STANDARD` | 正常情况下的基线性能 |
| `OVERFLOW` | 故障/过载下的系统行为 |
| `RECOVERY` | 故障解除后的系统恢复 |

### 场景配置

在 `scenarios.py` 中定义测试场景：

```python
from tests.fault_tolerance.deploy.scenarios import Scenario, Load, Failure

scenario = Scenario(
    name="worker_failure_migration",
    backend="vllm",
    load=Load(
        clients=10,
        requests_per_client=100,
        max_tokens=256
    ),
    failure=Failure(
        type="pod_kill",
        target="vllm-worker-0",
        trigger_after_requests=50
    )
)
```

### 运行部署测试

```bash
# Run all deployment tests
pytest tests/fault_tolerance/deploy/test_deployment.py -v

# Run specific scenario
pytest tests/fault_tolerance/deploy/test_deployment.py::test_worker_failure -v
```

### 验证 Checker

框架包含可插拔的验证器：

```python
from tests.fault_tolerance.deploy.base_checker import BaseChecker, ValidationContext

class MigrationChecker(BaseChecker):
    def check(self, context: ValidationContext) -> bool:
        # Verify migrations occurred
        migrations = context.metrics.get("migrations_total", 0)
        return migrations > 0
```

### 结果解析

解析测试结果以便分析：

```python
from tests.fault_tolerance.deploy.parse_results import process_overflow_recovery_test

results = process_overflow_recovery_test(log_dir="/path/to/logs")
print(f"Success rate: {results['success_rate']}")
print(f"P99 latency: {results['p99_latency_ms']}ms")
```

## 客户端工具

`client.py` 模块提供共享的客户端功能：

### 多线程负载生成

```python
from tests.fault_tolerance.client import client

# Generate load with multiple clients
results = client(
    base_url="http://localhost:8000",
    num_clients=10,
    requests_per_client=100,
    model="Qwen/Qwen3-0.6B",
    max_tokens=256,
    log_dir="/tmp/test_logs"
)
```

### 请求选项

| 参数 | 描述 |
|-----------|-------------|
| `base_url` | frontend URL |
| `num_clients` | 并发客户端数 |
| `requests_per_client` | 每个客户端发送的请求数 |
| `model` | 模型名 |
| `max_tokens` | 每个请求的最大 token 数 |
| `log_dir` | 客户端日志目录 |
| `endpoint` | `completions` 或 `chat/completions` |

## 运行完整测试套件

### 前置条件

1. 带 GPU 节点的 Kubernetes 集群
2. Dynamo 部署
3. etcd 集群（用于 HA 测试）
4. 故障注入服务（用于硬件测试）

### 环境设置

```bash
export KUBECONFIG=/path/to/kubeconfig
export DYNAMO_NAMESPACE=dynamo-test
export FRONTEND_URL=http://localhost:8000
```

### 运行所有测试

```bash
# Install test dependencies
pip install pytest pytest-asyncio

# Run all fault tolerance tests
pytest tests/fault_tolerance/ -v --tb=short

# Run with specific markers
pytest tests/fault_tolerance/ -v -m "not slow"
```

### 测试标记

| 标记 | 描述 |
|--------|-------------|
| `slow` | 长时测试（> 5 分钟） |
| `gpu` | 需要 GPU 资源 |
| `k8s` | 需要 Kubernetes 集群 |
| `etcd_ha` | 需要多节点 etcd |

## 最佳实践

### 1. 隔离测试环境

将容错测试运行在专用命名空间中：

```bash
kubectl create namespace dynamo-fault-test
```

### 2. 测试结束后清理

确保已注入的故障都恢复：

```bash
# List and recover all active faults
curl http://localhost:8080/api/v1/faults | jq -r '.[].id' | \
  xargs -I {} curl -X POST http://localhost:8080/api/v1/faults/{}/recover
```

### 3. 收集日志

保留日志便于调试：

```bash
pytest tests/fault_tolerance/ -v \
  --log-dir=/tmp/fault_test_logs \
  --capture=no
```

### 4. 测试期间监控

在测试期间观察系统状态：

```bash
# Terminal 1: Watch pods
watch kubectl get pods -n dynamo-test

# Terminal 2: Watch metrics
watch 'curl -s localhost:8000/metrics | grep -E "(migration|rejection)"'
```

## 相关文档

- [请求迁移](request-migration.md) - 迁移实现细节
- [请求取消](request-cancellation.md) - 取消实现
- [健康检查](../observability/health-checks.md) - 健康监控
- [指标](../observability/metrics.md) - 可用于监控的指标
