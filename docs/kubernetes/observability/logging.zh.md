---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Logging
---

本指南演示如何在 Kubernetes 中使用 Grafana Loki 与 Alloy 为 Dynamo 配置日志采集。该方案提供一个简洁的参考日志栈，可在包括 Minikube 与 MicroK8s 在内的 Kubernetes 集群中沿用。

> [!Note]
> 该方案面向开发与测试。生产环境的高可用配置请参考官方文档。

## 组件概览

- **[Grafana Loki](https://grafana.com/oss/loki/)**：快速且经济的 Kubernetes 原生日志聚合系统。

- **[Grafana Alloy](https://grafana.com/oss/alloy/)**：取代 Promtail 的 OpenTelemetry collector，从 Kubernetes pod 收集日志、指标与 traces。

- **[Grafana](https://grafana.com/grafana/)**：用于查询与浏览日志的可视化平台。

## 先决条件

### 1. Dynamo Kubernetes Platform

本指南假定你已安装 Dynamo Kubernetes Platform。详情请参考 [Dynamo Kubernetes Platform](../README.md)。

### 2. Kube-prometheus

虽然本指南不使用 Prometheus，但假定 Grafana 已通过 kube-prometheus 安装。详情参见 [kube-prometheus](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)。

### 3. 环境变量

#### Kubernetes 部署变量

会用到以下环境变量：
- `MONITORING_NAMESPACE`：Loki 安装所在的 namespace
- `DYN_NAMESPACE`：Dynamo Kubernetes Platform 安装所在的 namespace

```bash
export MONITORING_NAMESPACE=monitoring
export DYN_NAMESPACE=dynamo-system
```

#### Dynamo 日志变量

| 变量 | 说明 | 示例 |
|----------|-------------|---------|
| `DYN_LOGGING_JSONL` | 启用 JSONL 日志格式（Loki 必需） | `true` |
| `DYN_LOG` | 按 target 设置的日志级别 `<default_level>,<module_path>=<level>,<module_path>=<level>` | `DYN_LOG=info,dynamo_runtime::system_status_server:trace` |
| `DYN_LOG_USE_LOCAL_TZ` | 时间戳使用本地时区 | `true` |

## 安装步骤

### 1. 安装 Loki

首先以 single binary 模式安装 Loki，适合测试与开发：

```bash
# Add the Grafana Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki
helm install --values deploy/observability/k8s/logging/values/loki-values.yaml loki grafana/loki -n $MONITORING_NAMESPACE
```

我们的配置（`loki-values.yaml`）以一个适合测试与开发的简洁形态部署 Loki，使用本地的 MinIO 作为存储。可以用以下命令查看安装的 pod：
```bash
kubectl get pods -n $MONITORING_NAMESPACE -l app=loki
```

### 2. 安装 Grafana Alloy

接下来安装 Grafana Alloy collector，从 Kubernetes 集群收集日志并转发给 Loki。这里使用 Grafana 提供的 Helm chart `k8s-monitoring` 进行安装：

```bash
# Generate a custom values file with the namespace information
envsubst < deploy/observability/k8s/logging/values/alloy-values.yaml > alloy-custom-values.yaml

# Install the collector
helm install --values alloy-custom-values.yaml alloy grafana/k8s-monitoring -n $MONITORING_NAMESPACE
```

values 文件（`alloy-values.yaml`）为 collector 包含以下配置：
- 日志转发目的地（Loki）
- 收集日志的 namespace
- 映射到 Loki label 的 pod label
- 收集方式（kubernetesApi 或者 tail `/var/log/containers/`）

```yaml
destinations:
- name: loki
  type: loki
  url: http://loki-gateway.$MONITORING_NAMESPACE.svc.cluster.local/loki/api/v1/push
podLogs:
  enabled: true
  gatherMethod: kubernetesApi # collect logs from the kubernetes api, rather than /var/log/containers/; friendly for testing and development
  collector: alloy-logs
  labels:
    app_kubernetes_io_name: app.kubernetes.io/name
    nvidia_com_dynamo_component_type: nvidia.com/dynamo-component-type
    nvidia_com_dynamo_graph_deployment_name: nvidia.com/dynamo-graph-deployment-name
  labelsToKeep:
  - "app_kubernetes_io_name"
  - "container"
  - "instance"
  - "job"
  - "level"
  - "namespace"
  - "service_name"
  - "service_namespace"
  - "deployment_environment"
  - "deployment_environment_name"
  - "nvidia_com_dynamo_component_type" # extract this label from the dynamo graph deployment
  - "nvidia_com_dynamo_graph_deployment_name" # extract this label from the dynamo graph deployment
  namespaces:
  - $DYN_NAMESPACE
```

### 3. 在 Grafana 中配置 Loki 数据源与 Dynamo Logs 仪表盘

我们将在 Grafana 中查看与 DynamoGraphDeployment 关联的日志。为此需要在 Grafana 中配置 Loki 数据源与 Dynamo Logs 仪表盘。

由于我们使用的是搭配 Prometheus Operator 的 Grafana，可以通过 apply 以下 ConfigMap 快速完成：

```bash
# Configure Grafana with the Loki datasource
envsubst < deploy/observability/k8s/logging/grafana/loki-datasource.yaml | kubectl apply -n $MONITORING_NAMESPACE -f -

# Configure Grafana with the Dynamo Logs dashboard
kubectl apply -f deploy/observability/k8s/logging/grafana/logging-dashboard.yaml -n $MONITORING_NAMESPACE
```

> [!Note]
> 如果 Grafana 没有通过 Prometheus Operator 安装，可以在 Grafana UI 中手动导入 Loki 数据源与 Dynamo Logs 仪表盘。

### 4. 部署一个启用 JSONL 日志的 DynamoGraphDeployment

到此应当已经具备在 Grafana 实例中收集与查看日志的所有条件。剩下的就是部署一个 DynamoGraphDeployment 用以产生日志。

要在 DynamoGraphDeployment 中启用结构化日志，需要把 `DYN_LOGGING_JSONL` 环境变量设为 `1`。Sglang 后端的 `agg_logging.yaml` 已经为你设置好了。可以这样部署 DynamoGraphDeployment：

```bash
kubectl apply -n $DYN_NAMESPACE -f examples/backends/sglang/deploy/agg_logging.yaml
```

向其发送几次 chat completions 请求，让 frontend 与 worker pod 在 DynamoGraphDeployment 中产生结构化日志。现在就可以在 Grafana 中查看日志了。

## 在 Grafana 中查看日志

将 Grafana service 端口转发以访问 UI：

```bash
kubectl port-forward svc/prometheus-grafana 3000:80 -n $MONITORING_NAMESPACE
```

如果一切正常，在 Home > Dashboards > Dynamo Logs 下，你应该能看到一个用于查看 DynamoGraphDeployments 关联日志的仪表盘。

该仪表盘支持按 DynamoGraphDeployment、namespace 与组件类型（例如 frontend、worker 等）进行过滤。
