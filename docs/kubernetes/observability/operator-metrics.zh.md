---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Operator Metrics
---

## 概述

Dynamo Operator 暴露了一组 Prometheus 指标（metrics），用于监控其自身的健康状况与性能。这些指标与应用层指标（前端/worker）相互独立，并提供以下可观测性（observability）：

- **控制器调谐（Reconciliation）**：控制器处理 DynamoGraphDeployments、DynamoComponentDeployments 与 DynamoModels 的效率
- **Webhook 校验**：admission webhook 请求的性能与结果
- **资源清单**：当前被管理资源在不同状态/命名空间下的数量

## 前置条件

Operator 指标功能所需的监控基础设施与应用指标相同。详细的搭建步骤，请参见 [Kubernetes Metrics 指南](./metrics.md#prerequisites)。

**速查清单：**
- ✅ 已安装 kube-prometheus-stack（用于支持 ServiceMonitor）
- ✅ Prometheus 与 Grafana 正在运行
- ✅ 已通过 Helm 安装 Dynamo Operator

## 指标采集

### ServiceMonitor

Operator 指标会通过 ServiceMonitor 自动采集；当 `metricsService.enabled: true`（默认）时，Helm chart 会自动创建该 ServiceMonitor。

**与应用指标不同**（应用指标使用 PodMonitor），operator 使用的是 ServiceMonitor，无需手动配置 RBAC。Operator 的指标端点使用 controller-runtime 内置的 `WithAuthenticationAndAuthorization` filter 来安全地提供服务。

可通过下列命令验证 ServiceMonitor 已被创建：

```bash
kubectl get servicemonitor -n dynamo-system
```

### 关闭指标采集

要关闭 operator 指标采集：

```bash
helm upgrade dynamo-platform dynamo-platform-${RELEASE_VERSION}.tgz \
  --namespace dynamo-system \
  --set dynamo-operator.metricsService.enabled=false
```

## 可用指标

所有指标都使用 `dynamo_operator` 作为命名空间前缀。

### Reconciliation 指标

| 指标 | 类型 | 标签 | 描述 |
|--------|------|--------|-------------|
| `dynamo_operator_reconcile_duration_seconds` | Histogram | `resource_type`、`namespace`、`result` | 调谐循环耗时 |
| `dynamo_operator_reconcile_total` | Counter | `resource_type`、`namespace`、`result` | 调谐总次数 |
| `dynamo_operator_reconcile_errors_total` | Counter | `resource_type`、`namespace`、`error_type` | 按类型分类的调谐错误总数 |

**标签：**
- `resource_type`：`DynamoGraphDeployment`、`DynamoComponentDeployment`、`DynamoModel`、`DynamoGraphDeploymentRequest`、`DynamoGraphDeploymentScalingAdapter`
- `namespace`：资源所在的目标命名空间
- `result`：`success`、`error`、`requeue`
- `error_type`：`not_found`、`already_exists`、`conflict`、`validation`、`bad_request`、`unauthorized`、`forbidden`、`timeout`、`server_timeout`、`unavailable`、`rate_limited`、`internal`

### Webhook 指标

| 指标 | 类型 | 标签 | 描述 |
|--------|------|--------|-------------|
| `dynamo_operator_webhook_duration_seconds` | Histogram | `resource_type`、`operation` | webhook 校验请求耗时 |
| `dynamo_operator_webhook_requests_total` | Counter | `resource_type`、`operation`、`result` | webhook admission 请求总数 |
| `dynamo_operator_webhook_denials_total` | Counter | `resource_type`、`operation`、`reason` | 被拒绝的 webhook 请求总数及原因 |

**标签：**
- `resource_type`：与 reconciliation 指标一致
- `operation`：`CREATE`、`UPDATE`、`DELETE`
- `result`：`allowed`、`denied`
- `reason`：校验失败原因（如 `immutable_field_changed`、`invalid_config`）

### 资源清单指标

| 指标 | 类型 | 标签 | 描述 |
|--------|------|--------|-------------|
| `dynamo_operator_resources_total` | Gauge | `resource_type`、`namespace`、`status` | 按状态统计的当前资源数量 |

**标签：**
- `resource_type`：`DynamoGraphDeployment`、`DynamoComponentDeployment`、`DynamoModel`、`DynamoGraphDeploymentRequest`、`DynamoGraphDeploymentScalingAdapter`
- `namespace`：资源所在命名空间
- `status`：从各 CRD 状态推导出的资源状态。常见取值：
  - `"ready"` - 资源健康且可正常工作（DCD、DM、DGDSA）
  - `"not_ready"` - 资源已存在但不可工作（DCD、DM、DGDSA）
  - `"unknown"` - 状态无法确定（status 为空时的默认值）
  - DGD 使用 `.status.state` 中的：`"pending"`、`"successful"`、`"failed"`
  - DGDR 使用 `.status.phase` 中的：`"Pending"`、`"Profiling"`、`"Ready"`、`"Deploying"`、`"Deployed"`、`"Failed"`

## 查询示例

### 调谐性能

```promql
# P95 reconciliation duration by resource type
histogram_quantile(0.95,
  sum by (resource_type, le) (
    rate(dynamo_operator_reconcile_duration_seconds_bucket[5m])
  )
)

# Reconciliation rate by result
sum by (resource_type, result) (
  rate(dynamo_operator_reconcile_total[5m])
)

# Error rate by type
sum by (resource_type, error_type) (
  rate(dynamo_operator_reconcile_errors_total[5m])
)
```

### Webhook 性能

```promql
# Webhook P95 latency
histogram_quantile(0.95,
  sum by (resource_type, le) (
    rate(dynamo_operator_webhook_duration_seconds_bucket[5m])
  )
)

# Webhook denial rate
sum by (resource_type, operation, reason) (
  rate(dynamo_operator_webhook_denials_total[5m])
)
```

### 资源清单

```promql
# Total resources by type and state
sum by (resource_type, status) (
  dynamo_operator_resources_total
)

# DynamoGraphDeployments by state
sum by (status) (
  dynamo_operator_resources_total{resource_type="DynamoGraphDeployment"}
)

# All resources by namespace and state
sum by (resource_type, namespace, status) (
  dynamo_operator_resources_total
)
```

## Grafana 仪表盘

我们提供了一个预构建的 Grafana 仪表盘，用于可视化 operator 指标。

### 仪表盘分区

1. **Reconciliation 指标**（3 个面板）
   - 按资源类型与结果统计的调谐速率
   - P95 调谐耗时
   - 按类型分类的调谐错误

2. **Webhook 指标**（3 个面板）
   - 按操作类型统计的 webhook 请求速率
   - P95 webhook 耗时
   - 按原因分类的 webhook 拒绝数

3. **资源清单**（2 个面板）
   - 按状态与命名空间随时间变化的资源清单（可按资源类型筛选）
   - 按状态统计的当前资源数量（可按资源类型筛选）

4. **运行健康**（2 个面板）
   - 调谐成功率仪表
   - Webhook admission 成功率仪表

### 部署仪表盘

```bash
kubectl apply -f deploy/observability/k8s/grafana-operator-dashboard-configmap.yaml
```

仪表盘会自动出现在 Grafana 中（前提是已配置好 Grafana dashboard sidecar，kube-prometheus-stack 默认已包含）。

### 找到该仪表盘

1. 端口转发到 Grafana（如有需要）：
   ```bash
   kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
   ```

2. 访问 http://localhost:3000 登录 Grafana

3. 进入 **Dashboards** → 搜索 **"Dynamo Operator"**

### 仪表盘筛选器

仪表盘包含两个筛选变量：

- **Namespace**：可查看所有命名空间的指标，或按指定命名空间筛选（多选）
- **Resource Type**：按资源类型筛选所有面板，或选择 “All” 跨所有 CRD 查看汇总指标（单选）

当 Resource Type 选择 “All” 时，所有面板都会带上 resource_type 标签，展示 5 种被管理 CRD 的数据，便于区分。

## 直接访问指标

访问 Prometheus 与 Grafana 的具体方式，请参见 [Kubernetes Metrics 指南](./metrics.md#viewing-the-metrics)。

进入 Prometheus 后即可直接查询 operator 指标：

```bash
# Port-forward to Prometheus
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring

# Visit http://localhost:9090 and try queries like:
# - dynamo_operator_reconcile_total
# - dynamo_operator_webhook_requests_total
# - dynamo_operator_resources_total
```

## 故障排查

### Prometheus 中看不到指标

1. **检查 ServiceMonitor 是否存在：**
   ```bash
   kubectl get servicemonitor -n dynamo-system | grep operator
   ```

2. **检查 ServiceMonitor 是否被 Prometheus 发现：**
   - 进入 Prometheus UI → Status → Targets
   - 查找 `serviceMonitor/dynamo-system/dynamo-platform-dynamo-operator-operator`
   - 状态应为：`UP`

3. **检查 Prometheus 的 selector 配置：**
   ```bash
   kubectl get prometheus -o yaml | grep serviceMonitorSelector
   ```
   请确保在安装 kube-prometheus-stack 时设置了 `serviceMonitorSelectorNilUsesHelmValues: false`。

### Grafana 中看不到仪表盘

1. **检查 ConfigMap 是否已创建：**
   ```bash
   kubectl get configmap -n monitoring grafana-operator-dashboard
   ```

2. **检查 ConfigMap 是否带有标签：**
   ```bash
   kubectl get configmap -n monitoring grafana-operator-dashboard -o jsonpath='{.metadata.labels.grafana_dashboard}'
   ```
   应返回 `"1"`

3. **检查 Grafana dashboard sidecar 配置：**
   ```bash
   kubectl get deployment -n monitoring prometheus-grafana -o yaml | grep -A 5 sidecar
   ```
   sidecar 应当被配置为监听带有 `grafana_dashboard: "1"` 标签的 ConfigMap。

4. **重启 Grafana pod** 以强制刷新仪表盘：
   ```bash
   kubectl rollout restart deployment/prometheus-grafana -n monitoring
   ```

## 相关文档

- [Kubernetes Metrics 指南](./metrics.md) - 前端与 worker 的应用层指标
- [Dynamo Operator 指南](../dynamo-operator.md) - Operator 架构与部署模式
- [Operator Webhooks](../webhooks.md) - Webhook 校验细节
