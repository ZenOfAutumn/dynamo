---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Autoscaling
---

本指南以 `examples/backends/sglang/deploy/agg.yaml` 中的 `sglang-agg` 示例为例，介绍如何为 DynamoGraphDeployment（DGD）服务配置自动扩缩容。

## 示例 DGD

本指南所有示例使用以下 DGD：

```yaml
# examples/backends/sglang/deploy/agg.yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: sglang-agg
  namespace: default
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1

    decode:
      componentType: worker
      replicas: 1
      resources:
        limits:
          gpu: "1"
```

**关键标识：**
- **DGD 名称**：`sglang-agg`
- **命名空间**：`default`
- **服务**：`Frontend`、`decode`
- **dynamo_namespace 标签**：`default-sglang-agg`（用于指标过滤）

## 概述

Dynamo 通过 `DynamoGraphDeploymentScalingAdapter`（DGDSA）资源提供灵活的自动扩缩容。要让 operator 为某个服务创建 DGDSA，请按下文"为服务启用 DGDSA"一节进行。这些 adapter 实现了 Kubernetes 的 [Scale subresource](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#scale-subresource)，从而可与以下方案集成：

| 自动扩缩容器 | 描述 | 最佳适用 |
|------------|-------------|----------|
| **KEDA** | 事件驱动的自动扩缩容（推荐） | 大多数场景 |
| **Kubernetes HPA** | 原生水平扩缩容 | 简单的 CPU/内存扩缩容 |
| **Dynamo Planner** | 基于 SLA 优化的 LLM 感知扩缩容 | 生产 LLM 工作负载 |
| **自定义控制器** | 任何兼容 scale subresource 的控制器 | 自定义需求 |

> **⚠️ 弃用提示**：DGD 中的 `spec.services[X].autoscaling` 字段 **已废弃且会被忽略**。请改用 DGDSA + HPA、KEDA 或 Planner。如果你已有配置了 `autoscaling` 的 DGD，会看到告警。移除该字段以消除告警。

## 架构

```
┌──────────────────────────────────┐          ┌─────────────────────────────────────┐
│   DynamoGraphDeployment          │          │   Scaling Adapters (auto-created)   │
│   "sglang-agg"                   │          │   (one per service)                 │
├──────────────────────────────────┤          ├─────────────────────────────────────┤
│                                  │          │                                     │
│  spec.services:                  │          │  ┌─────────────────────────────┐    │      ┌──────────────────┐
│                                  │          │  │ sglang-agg-frontend         │◄───┼──────│   Autoscalers    │
│    ┌────────────────────────┐◄───┼──────────┼──│ spec.replicas: 1            │    │      │                  │
│    │ Frontend: 1 replica    │    │          │  └─────────────────────────────┘    │      │  • KEDA          │
│    └────────────────────────┘    │          │                                     │      │  • HPA           │
│                                  │          │  ┌─────────────────────────────┐    │      │  • Planner       │
│    ┌────────────────────────┐◄───┼──────────┼──│ sglang-agg-decode           │◄───┼──────│  • Custom        │
│    │ decode:   1 replica    │    │          │  │ spec.replicas: 1            │    │      │                  │
│    └────────────────────────┘    │          │  └─────────────────────────────┘    │      └──────────────────┘
│                                  │          │                                     │
└──────────────────────────────────┘          └─────────────────────────────────────┘
```

**工作原理：**

1. 你部署一个带服务的 DGD（Frontend、decode）
2. operator 自动为每个服务创建一个 DGDSA
3. 自动扩缩容器（KEDA、HPA、Planner）通过 `/scale` 子资源对 adapter 操作
4. adapter 控制器将副本变化同步到 DGD
5. DGD 控制器协调底层 pod

## 查看扩缩容 Adapter

部署 `sglang-agg` DGD 后，验证自动创建的 adapter：

```bash
kubectl get dgdsa -n default

# Example output:
# NAME                  DGD         SERVICE    REPLICAS   AGE
# sglang-agg-frontend   sglang-agg  Frontend   1          5m
# sglang-agg-decode     sglang-agg  decode     1          5m
```

## 副本所有权模型

启用 DGDSA 时，它成为副本数的 **唯一可信来源**。这与 Kubernetes Deployment 拥有 ReplicaSet 的模式相同。

### 工作原理

1. **DGDSA 拥有副本**：自动扩缩容器（HPA、KEDA、Planner）更新 DGDSA 的 `spec.replicas`
2. **DGDSA 同步到 DGD**：DGDSA 控制器将副本数写入 DGD 的服务
3. **直接编辑 DGD 被阻止**：一个 validating webhook 禁止用户直接编辑 DGD 中的 `spec.services[X].replicas`
4. **允许的控制器**：仅授权控制器（operator、Planner）可修改 DGD 副本

### DGDSA 启用时的手动扩缩容

启用 DGDSA 时，对 adapter（而非 DGD）执行 `kubectl scale`：

```bash
# ✅ Correct - scale via DGDSA
kubectl scale dgdsa sglang-agg-decode --replicas=3

# ❌ Blocked - direct DGD edit rejected by webhook
kubectl patch dgd sglang-agg --type=merge -p '{"spec":{"services":{"decode":{"replicas":3}}}}'
# Error: spec.services[decode].replicas cannot be modified directly when scaling adapter is enabled;
#        use 'kubectl scale dgdsa/sglang-agg-decode --replicas=3' or update the DynamoGraphDeploymentScalingAdapter instead
```

## 为服务启用 DGDSA

默认情况下不会为服务创建 DGDSA，从而允许通过 DGD 直接管理副本。要启用 HPA、KEDA 或 Planner 的自动扩缩容，需显式启用 scaling adapter：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: sglang-agg
spec:
  services:
    Frontend:
      replicas: 2        # ← No DGDSA by default, direct edits allowed

    decode:
      replicas: 1
      scalingAdapter:
        enabled: true    # ← DGDSA created, managed via adapter
```

**何时启用 DGDSA：**
- 想使用 HPA、KEDA 或 Planner 进行自动扩缩容
- 想清晰区分"期望规模"（adapter）与"部署配置"（DGD）
- 想防止意外的直接副本编辑

**何时保持禁用 DGDSA（默认）：**
- 想要简单的手动副本管理
- 该服务不需要自动扩缩容
- 偏好直接编辑 DGD 而非通过 adapter 扩缩容

## 使用 Dynamo Planner 自动扩缩容

Dynamo Planner 是一种 LLM 感知的自动扩缩容器，基于推理特定指标（如 TTFT、ITL、KV 缓存利用率）做出扩缩容决策。

**何时使用 Planner：**
- 想要开箱即用的 LLM 优化扩缩容
- 需要在 prefill/decode 服务之间协调扩缩容
- 想要 SLA 驱动的扩缩容（如目标 TTFT \< 500ms）

**Planner 工作原理：**

Planner 作为 DGD 中的服务组件部署。它：
1. 从 Prometheus 查询前端指标（请求率、延迟等）
2. 利用剖析数据预测最佳副本数
3. 扩缩 prefill/decode worker 以满足 SLA 目标

**部署：**

推荐通过 `DynamoGraphDeploymentRequest`（DGDR）部署 Planner。完整说明请参阅 [SLA Planner 快速入门](../components/planner/planner-guide.md)。

带 Planner 的示例配置：
- `examples/backends/vllm/deploy/disagg_planner.yaml`
- `examples/backends/sglang/deploy/disagg_planner.yaml`
- `examples/backends/trtllm/deploy/disagg_planner.yaml`

更多详情请参阅 [SLA Planner 文档](../components/planner/planner-guide.md)。

## 使用 Kubernetes HPA 自动扩缩容

Horizontal Pod Autoscaler（HPA）是 Kubernetes 原生的扩缩容方案。

**何时使用 HPA：**
- 简单可预测的扩缩容需求
- 想使用标准 Kubernetes 工具
- 需要基于 CPU 或内存的扩缩容

<Note>
对于自定义指标（如 TTFT 或队列深度），建议改用 [KEDA](#autoscaling-with-keda-recommended)——配置更简单。
</Note>

### 基础 HPA（基于 CPU）

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sglang-agg-frontend-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: nvidia.com/v1alpha1
    kind: DynamoGraphDeploymentScalingAdapter
    name: sglang-agg-frontend
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
    scaleUp:
      stabilizationWindowSeconds: 0
```

### 使用 Dynamo 指标的 HPA

Dynamo 导出多个对自动扩缩容有用的指标。它们在每个前端 pod 的 `/metrics` 端点提供。

> **另请参阅**：完整 Dynamo 指标列表请参阅 [Metrics 参考](../observability/metrics.md)。Prometheus 与 Grafana 安装请参阅 [Prometheus 与 Grafana 安装指南](../observability/prometheus-grafana.md)。

#### 可用 Dynamo 指标

| 指标 | 类型 | 说明 | 适合扩缩容 |
|--------|------|-------------|------------------|
| `dynamo_frontend_active_requests` | Gauge | 从 HTTP 入口到响应完成期间的并发请求总数 | ✅ 所有服务 |
| `dynamo_frontend_stage_requests{stage,phase}` | Gauge | 当前位于某个前端流水线阶段的请求数（`preprocess`、`route`、`dispatch`） | ✅ Worker —— 用 `sum(...)` 表示队列深度，或 `stage="dispatch"` 表示后端 prefill 饱和 |
| `dynamo_frontend_time_to_first_token_seconds` | Histogram | TTFT 延迟 | ✅ Worker |
| `dynamo_frontend_inter_token_latency_seconds` | Histogram | ITL 延迟 | ✅ Decode |
| `dynamo_frontend_request_duration_seconds` | Histogram | 请求总时长 | ⚠️ 通用 |
| `dynamo_frontend_inflight_requests` | Gauge | 发往引擎的并发请求数 | ⚠️ **已弃用** —— 请使用 `dynamo_frontend_active_requests` |
| `dynamo_frontend_queued_requests` | Gauge | 在 HTTP 队列中等待的请求 | ⚠️ **已弃用** —— 请使用跨 `preprocess` + `route` + `dispatch` 的 `sum(dynamo_frontend_stage_requests)` |

`stage` 和 `phase` 标签的完整定义以及衍生信号公式，请参阅 Metrics 参考中的 [Stage and phase labels](../observability/metrics.md#stage-and-phase-labels)。

#### 指标标签

Dynamo 指标包含以下用于过滤的标签：

| 标签 | 说明 | 示例 |
|-------|-------------|---------|
| `dynamo_namespace` | 唯一的 DGD 标识（`{k8s-namespace}-{dgd-name}`） | `default-sglang-agg` |
| `model` | 所服务的模型 | `Qwen/Qwen3-0.6B` |

<Note>
当同一命名空间中有多个 DGD 时，使用 `dynamo_namespace` 过滤特定 DGD 的指标。
</Note>

#### 示例：基于 TTFT 扩缩 Decode 服务

通过 HPA + Prometheus Adapter 使用，需要配置外部指标。

**步骤 1：配置 Prometheus Adapter**

将以下内容添加到你的 Helm values 文件（如 `prometheus-adapter-values.yaml`）：

```yaml
# prometheus-adapter-values.yaml
prometheus:
  url: http://prometheus-kube-prometheus-prometheus.monitoring.svc
  port: 9090

rules:
  external:
  # TTFT p95 from frontend - used to scale decode
  - seriesQuery: 'dynamo_frontend_time_to_first_token_seconds_bucket{namespace!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
    name:
      as: "dynamo_ttft_p95_seconds"
    metricsQuery: |
      histogram_quantile(0.95,
        sum(rate(dynamo_frontend_time_to_first_token_seconds_bucket{<<.LabelMatchers>>}[5m]))
        by (le, namespace, dynamo_namespace)
      )
```

**步骤 2：安装 Prometheus Adapter**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter \
  -n monitoring --create-namespace \
  -f prometheus-adapter-values.yaml
```

**步骤 3：验证指标可用**

```bash
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/<your-namespace>/dynamo_ttft_p95_seconds" | jq
```

**步骤 4：创建 HPA**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sglang-agg-decode-hpa
spec:
  scaleTargetRef:
    apiVersion: nvidia.com/v1alpha1
    kind: DynamoGraphDeploymentScalingAdapter
    name: sglang-agg-decode              # ← DGD name + service name (lowercase)
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metric:
        name: dynamo_ttft_p95_seconds
        selector:
          matchLabels:
            dynamo_namespace: "default-sglang-agg"  # ← {namespace}-{dgd-name}
      target:
        type: Value
        value: "500m"  # Scale up when TTFT p95 > 500ms
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60    # Wait 1 min before scaling down
      policies:
      - type: Pods
        value: 1
        periodSeconds: 30
    scaleUp:
      stabilizationWindowSeconds: 0      # Scale up immediately
      policies:
      - type: Pods
        value: 2
        periodSeconds: 30
```

**工作原理：**
1. 前端 pod 导出 `dynamo_frontend_time_to_first_token_seconds` 直方图
2. Prometheus Adapter 按 `dynamo_namespace` 计算 TTFT 的 p95
3. HPA 监控按 `dynamo_namespace: "default-sglang-agg"` 过滤的该指标
4. 当 TTFT p95 > 500ms 时，HPA 扩容 `sglang-agg-decode` adapter
5. adapter 控制器将副本数同步到 DGD 的 `decode` 服务
6. 创建更多 decode worker 以降低 TTFT

#### 示例：基于队列深度扩缩容

这里的"队列深度"指的是已进入前端但尚未收到首 token 的请求数——也就是跨 `preprocess`、`route`、`dispatch` 阶段的 `dynamo_frontend_stage_requests` 之和。它取代了已弃用的 `dynamo_frontend_queued_requests`。

将该规则加入你的 `prometheus-adapter-values.yaml`（与 TTFT 规则并列）：

```yaml
# Add to rules.external in prometheus-adapter-values.yaml
- seriesQuery: 'dynamo_frontend_stage_requests{namespace!="",stage=~"preprocess|route|dispatch"}'
  resources:
    overrides:
      namespace: {resource: "namespace"}
  name:
    as: "dynamo_frontend_pending_requests"
  metricsQuery: |
    sum(<<.Series>>{<<.LabelMatchers>>}) by (namespace, dynamo_namespace)
```

然后创建 HPA：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sglang-agg-decode-queue-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: nvidia.com/v1alpha1
    kind: DynamoGraphDeploymentScalingAdapter
    name: sglang-agg-decode
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metric:
        name: dynamo_frontend_pending_requests
        selector:
          matchLabels:
            dynamo_namespace: "default-sglang-agg"
      target:
        type: Value
        value: "10"  # Scale up when queue > 10 requests
```

## 使用 KEDA 自动扩缩容（推荐）

KEDA（Kubernetes Event-driven Autoscaling）扩展 Kubernetes 提供事件驱动的自动扩缩容，支持包括 Prometheus 在内的 50+ scaler。

**相对 HPA + Prometheus Adapter 的优势：**
- 无需配置 Prometheus Adapter
- PromQL 查询直接定义在 ScaledObject 中（声明式、按部署独立）
- 易于更新——直接 `kubectl apply` 即可
- 可在空闲时缩容到零
- 一个对象支持多个触发器

**何时使用 KEDA：**
- 想要更简单的配置（无需管理 Prometheus Adapter）
- 需要事件驱动扩缩容（如队列深度、Kafka 等）
- 想在空闲时缩容到零

### 安装 KEDA

```bash
# Add KEDA Helm repo
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

# Install KEDA
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace

# Verify installation
kubectl get pods -n keda
```

<Note>
如果已安装 Prometheus Adapter，请先卸载（`helm uninstall prometheus-adapter -n monitoring`），或安装 KEDA 时附加 `--set metricsServer.enabled=false`，避免 API 冲突。
</Note>

### 示例：基于 TTFT 扩缩 Decode

使用 `examples/backends/sglang/deploy/agg.yaml` 中的 `sglang-agg` DGD：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sglang-agg-decode-scaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: nvidia.com/v1alpha1
    kind: DynamoGraphDeploymentScalingAdapter
    name: sglang-agg-decode
  minReplicaCount: 1
  maxReplicaCount: 10
  pollingInterval: 15      # Check metrics every 15 seconds
  cooldownPeriod: 60       # Wait 60s before scaling down
  triggers:
  - type: prometheus
    metadata:
      # Update this URL to match your Prometheus service
      serverAddress: http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090
      metricName: dynamo_ttft_p95
      query: |
        histogram_quantile(0.95,
          sum(rate(dynamo_frontend_time_to_first_token_seconds_bucket{dynamo_namespace="default-sglang-agg"}[5m]))
          by (le)
        )
      threshold: "0.5"              # Scale up when TTFT p95 > 500ms (0.5 seconds)
      activationThreshold: "0.1"    # Start scaling when TTFT > 100ms
```

应用：

```bash
kubectl apply -f sglang-agg-decode-scaler.yaml
```

### 验证 KEDA 扩缩容

```bash
# Check ScaledObject status
kubectl get scaledobject -n default

# KEDA creates an HPA under the hood - you can see it
kubectl get hpa -n default

# Example output:
# NAME                                REFERENCE                                              TARGETS      MINPODS   MAXPODS   REPLICAS
# keda-hpa-sglang-agg-decode-scaler   DynamoGraphDeploymentScalingAdapter/sglang-agg-decode  45m/500m     1         10        1

# Get detailed status
kubectl describe scaledobject sglang-agg-decode-scaler -n default
```

### 示例：基于队列深度扩缩容

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sglang-agg-decode-queue-scaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: nvidia.com/v1alpha1
    kind: DynamoGraphDeploymentScalingAdapter
    name: sglang-agg-decode
  minReplicaCount: 1
  maxReplicaCount: 10
  pollingInterval: 15
  cooldownPeriod: 60
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090
      metricName: dynamo_frontend_pending_requests
      query: |
        sum(dynamo_frontend_stage_requests{dynamo_namespace="default-sglang-agg",stage=~"preprocess|route|dispatch"})
      threshold: "10"    # Scale up when queue > 10 requests
```

### KEDA 工作原理

KEDA 在底层创建并管理一个 HPA：

```
┌──────────────────────────────────────────────────────────────────────┐
│  You create: ScaledObject                                            │
│    - scaleTargetRef: sglang-agg-decode                               │
│    - triggers: prometheus query                                      │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│  KEDA Operator automatically creates: HPA                            │
│    - name: keda-hpa-sglang-agg-decode-scaler                         │
│    - scaleTargetRef: sglang-agg-decode                               │
│    - metrics: External (from KEDA metrics server)                    │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│  DynamoGraphDeploymentScalingAdapter: sglang-agg-decode              │
│    - spec.replicas: updated by HPA                                   │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│  DynamoGraphDeployment: sglang-agg                                   │
│    - spec.services.decode.replicas: synced from adapter              │
└──────────────────────────────────────────────────────────────────────┘
```

## 混合扩缩容

对于解耦部署（prefill + decode），可为不同服务使用不同扩缩容策略：

```yaml
---
# HPA for Frontend (CPU-based)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sglang-agg-frontend-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: nvidia.com/v1alpha1
    kind: DynamoGraphDeploymentScalingAdapter
    name: sglang-agg-frontend
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

---
# KEDA for Decode (TTFT-based)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sglang-agg-decode-scaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: nvidia.com/v1alpha1
    kind: DynamoGraphDeploymentScalingAdapter
    name: sglang-agg-decode
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090
      query: |
        histogram_quantile(0.95,
          sum(rate(dynamo_frontend_time_to_first_token_seconds_bucket{dynamo_namespace="default-sglang-agg"}[5m]))
          by (le)
        )
      threshold: "0.5"
```

## 手动扩缩容

### 启用 DGDSA 时

启用 DGDSA 时，通过 adapter 扩缩容：

```bash
kubectl scale dgdsa sglang-agg-decode -n default --replicas=3
```

验证扩缩容：

```bash
kubectl get dgdsa sglang-agg-decode -n default

# Output:
# NAME                DGD         SERVICE   REPLICAS   AGE
# sglang-agg-decode   sglang-agg  decode    3          10m
```

<Note>
如果有自动扩缩容器（KEDA、HPA、Planner）正在管理 adapter，你的更改会在下一次评估周期被覆盖。
</Note>

### 禁用 DGDSA 时（默认）

如果你已禁用某个服务的 scaling adapter，则直接编辑 DGD：

```bash
kubectl patch dgd sglang-agg --type=merge -p '{"spec":{"services":{"decode":{"replicas":3}}}}'
```

或编辑 YAML（无 `scalingAdapter.enabled: true` 表示允许直接编辑）：

```yaml
spec:
  services:
    decode:
      replicas: 3
      # No scalingAdapter.enabled means replicas can be edited directly
```

## 最佳实践

### 1. 每个服务只选用一种自动扩缩容器

避免对同一服务配置多种自动扩缩容器：

| 配置 | 状态 |
|---------------|--------|
| 前端用 HPA、prefill/decode 用 Planner | ✅ 良好 |
| 全部服务都用 KEDA | ✅ 良好 |
| 仅 Planner（默认） | ✅ 良好 |
| HPA + Planner 同时管理 decode | ❌ 不佳 - 会互相冲突 |

### 2. 使用合适的指标

| 服务类型 | 推荐指标 | Dynamo 指标 |
|--------------|---------------------|---------------|
| Frontend | CPU 利用率、请求率 | `dynamo_frontend_requests_total` |
| Prefill | dispatch 阶段深度（后端 prefill 饱和度）、TTFT | `dynamo_frontend_stage_requests{stage="dispatch"}`、`dynamo_frontend_time_to_first_token_seconds` |
| Decode | ITL、活跃并发 | `dynamo_frontend_inter_token_latency_seconds`、`dynamo_frontend_active_requests` |

### 3. 配置稳定窗口

通过适当的稳定窗口防止抖动：

```yaml
# HPA
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
  scaleUp:
    stabilizationWindowSeconds: 0    # Scale up immediately

# KEDA
spec:
  cooldownPeriod: 300
```

### 4. 设置合理的最小/最大副本数

始终为 HPA/KEDA 配置最小与最大副本以避免：
- 缩容到零（除非有意为之）
- 无界扩容耗尽集群资源

## 故障排查

### Adapter 未创建

```bash
# Check DGD status
kubectl describe dgd sglang-agg -n default

# Check operator logs
kubectl logs -n dynamo-system deployment/dynamo-operator
```

### 扩缩容不生效

```bash
# Check adapter status
kubectl describe dgdsa sglang-agg-decode -n default

# Check HPA/KEDA status
kubectl describe hpa sglang-agg-decode-hpa -n default
kubectl describe scaledobject sglang-agg-decode-scaler -n default

# Verify metrics are available in Kubernetes metrics API
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1
```

### 指标不可用

如果 HPA/KEDA 显示指标为 `<unknown>`：

```bash
# Check if Dynamo metrics are being scraped
kubectl port-forward -n default svc/sglang-agg-frontend 8000:8000
curl http://localhost:8000/metrics | grep dynamo_frontend

# Example output (note: stage_requests has no `model` label — it's per frontend pod):
# dynamo_frontend_active_requests{model="Qwen/Qwen3-0.6B"} 5
# dynamo_frontend_stage_requests{stage="preprocess",phase=""} 0
# dynamo_frontend_stage_requests{stage="route",phase="aggregated"} 0
# dynamo_frontend_stage_requests{stage="dispatch",phase="aggregated"} 2
# dynamo_frontend_queued_requests{model="Qwen/Qwen3-0.6B"} 2        # deprecated
# dynamo_frontend_inflight_requests{model="Qwen/Qwen3-0.6B"} 5      # deprecated

# Verify Prometheus is scraping the metrics
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# Then query: dynamo_frontend_time_to_first_token_seconds_bucket

# Check KEDA operator logs
kubectl logs -n keda deployment/keda-operator
```

### 频繁的扩缩容抖动

如果观察到不稳定的扩缩容：

1. 检查是否有多个自动扩缩容器同时管理同一 adapter
2. 增大 KEDA ScaledObject 的 `cooldownPeriod`
3. 增大 HPA behavior 中的 `stabilizationWindowSeconds`

## 参考

- [Kubernetes HPA 文档](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [KEDA 文档](https://keda.sh/)
- [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter)
- [Planner 文档](../components/planner/planner-guide.md)
- [Dynamo Metrics 参考](../observability/metrics.md)
- [Prometheus 与 Grafana 安装](../observability/prometheus-grafana.md)
