---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Metrics
---

## 概述

本指南介绍如何使用 kube-prometheus-stack 采集并可视化 Dynamo 组件的指标（metrics）。kube-prometheus-stack 通过 PodMonitor 等自定义资源（CRD）为 Kubernetes 应用提供了强大且灵活的监控配置方式，可方便地自动发现并抓取 Dynamo 组件的指标。

## 前置条件

### 安装 kube-prometheus-stack
如果你尚无 Prometheus 环境，建议安装 kube-prometheus-stack。它是一组 Kubernetes 清单，包含 Prometheus Operator、Prometheus、Grafana 以及其他监控组件，开箱即用。该 stack 引入了若干自定义资源，便于在 Kubernetes 中部署与管理监控：

- `PodMonitor`：基于 label selector 自动发现并抓取 Pod 指标
- `ServiceMonitor`：与 PodMonitor 类似，但作用于 Service
- `PrometheusRule`：定义告警与记录规则

基础安装：
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
# 这些 values 允许 Prometheus 拾取 kube-prometheus-stack helm release 之外的 PodMonitor
helm install prometheus -n monitoring --create-namespace prometheus-community/kube-prometheus-stack \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorNamespaceSelector.matchLabels=null \
  --set prometheus.prometheusSpec.probeNamespaceSelector.matchLabels=null
```

> [!Note]
> 下文所列命令默认你按上述方式安装了 kube-prometheus-stack。若你的监控栈安装方式不同，可能需要相应修改后续的 `kubectl` 命令（如调整 Namespace 或 Service 名）。

### 安装 Dynamo Operator
配置指标采集前，集群中必须已部署 Dynamo Operator。详细步骤请参考[安装指南](../installation-guide.md)。
请将 `dynamo-operator.dynamo.metrics.prometheusEndpoint` 设置为上一步部署的 Prometheus 端点。

```bash
helm install dynamo-platform ...
  --set dynamo-operator.dynamo.metrics.prometheusEndpoint=http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
```


### 用于 CPU/内存指标的 Node Exporter

Dynamo 的 Grafana 仪表板包含节点级 CPU 利用率、系统负载与容器资源使用面板。这些指标由 [node-exporter](https://github.com/prometheus/node_exporter) 采集并导出至 Prometheus，node-exporter 暴露 Linux 系统的硬件与操作系统指标。

> [!Note]
> 上述安装的 kube-prometheus-stack 默认包含 node-exporter。若你使用自定义 Prometheus 部署，请确保以 DaemonSet 方式在集群节点上部署 node-exporter。

验证 node-exporter 运行情况：

```bash
kubectl get daemonset -A | grep node-exporter
```

如果 node-exporter 没有运行，可通过 kube-prometheus-stack 安装，或单独部署。详见 [node-exporter 文档](https://github.com/prometheus/node_exporter)。

### DCGM 指标采集（可选）

GPU 利用率指标由 dcgm-exporter 采集并导出至 Prometheus。Dynamo Grafana 仪表板包含与你的 Dynamo 部署相关的 GPU 利用率面板。要使其生效，需要确保集群内运行了 dcgm-exporter。检查方式如下：

```bash
kubectl get daemonset -A | grep dcgm-exporter
```

若输出为空，请安装 dcgm-exporter。详见官方 [dcgm-exporter 文档](https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/latest/dcgm-exporter.html)。


## 部署一个 DynamoGraphDeployment

首先部署一个简单的 vLLM 聚合（aggregated）部署：

```bash
export NAMESPACE=dynamo-system # 已安装 dynamo operator 的命名空间
pushd examples/backends/vllm/deploy
kubectl apply -f agg.yaml -n $NAMESPACE
popd
```

这将创建两个组件：
- 一个前端（Frontend）组件，在其 HTTP 端口暴露指标
- 一个 Worker 组件，在其 system port 暴露指标

两者都暴露符合 OpenMetrics 格式的 `/metrics` 端点，但根据各自角色暴露的指标不同。详细信息：
- 部署配置：见 [vLLM README](../../backends/vllm/README.md)
- 可用指标：见[指标指南](../../observability/metrics.md)

### 验证部署

发送一些测试请求以填充指标：

```bash
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [
    {
        "role": "user",
        "content": "In the heart of Eldoria, an ancient land of boundless magic and mysterious creatures, lies the long-forgotten city of Aeloria. Once a beacon of knowledge and power, Aeloria was buried beneath the shifting sands of time, lost to the world for centuries. You are an intrepid explorer, known for your unparalleled curiosity and courage, who has stumbled upon an ancient map hinting at ests that Aeloria holds a secret so profound that it has the potential to reshape the very fabric of reality. Your journey will take you through treacherous deserts, enchanted forests, and across perilous mountain ranges. Your Task: Character Background: Develop a detailed background for your character. Describe their motivations for seeking out Aeloria, their skills and weaknesses, and any personal connections to the ancient city or its legends. Are they driven by a quest for knowledge, a search for lost familt clue is hidden."
    }
    ],
    "stream": true,
    "max_tokens": 30
  }'
```

更多关于部署验证的内容，请参见 [vLLM README](../../backends/vllm/README.md)。

## 配置指标采集

### 启用 NIXL 遥测（可选）

如需在 Dynamo 指标之外另外启用 NIXL 遥测指标，请在 worker 组件中设置以下环境变量：

spec:
  services:
    YourWorker:
      envs:
        - name: NIXL_TELEMETRY_ENABLE
          value: "y"

NIXL 遥测默认关闭。启用后，NIXL 指标会暴露在 `NIXL_TELEMETRY_PROMETHEUS_PORT` 指定的端口（默认 19090）。

### 创建 PodMonitor

Prometheus Operator 通过 PodMonitor 资源自动发现并抓取 Pod 指标。为了启用这一发现机制，Dynamo Operator 会自动创建 PodMonitor 资源，并为所有 Pod 添加以下 label：
- `nvidia.com/metrics-enabled: "true"` —— 启用指标采集
- `nvidia.com/dynamo-component-type: "frontend|worker"` —— 标识组件类型

<Note>
你可以为某个具体部署关闭指标采集，方法是在 DynamoGraphDeployment 上添加以下 annotation：
</Note>
```yaml
apiVersion: nvidia.com/v1
kind: DynamoGraphDeployment
metadata:
  name: my-deployment
  annotations:
    nvidia.com/enable-metrics: "false"
spec:
  # …
```

### 配置 Grafana 仪表板

应用 Dynamo 仪表板配置以让 Grafana 加载该仪表板：
```bash
kubectl apply -n monitoring -f deploy/observability/k8s/grafana-dynamo-dashboard-configmap.yaml
```

仪表板内嵌于 ConfigMap 中。由于其带有 label `grafana_dashboard: "1"`，Grafana 会自动发现并将其加入可用仪表板列表。该仪表板包含以下面板：
- 前端请求速率
- 首 token 时间（time to first token）
- token 间延迟（inter-token latency）
- 请求时长
- 输入/输出序列长度
- 通过 DCGM 获取的 GPU 利用率
- 节点 CPU 利用率与系统负载
- 各 Pod 的容器 CPU 使用
- 各 Pod 的内存使用

## 查看指标

### 在 Prometheus 中
```bash
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring
```

访问 http://localhost:9090，可尝试以下示例查询：
- `dynamo_frontend_requests_total`
- `dynamo_frontend_time_to_first_token_seconds_bucket`

![Prometheus UI 显示 Dynamo 指标](../../assets/img/prometheus-k8s.png)

### 在 Grafana 中
```bash
# 获取 Grafana 凭据
export GRAFANA_USER=$(kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-user}" | base64 --decode)
export GRAFANA_PASSWORD=$(kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode)
echo "Grafana user: $GRAFANA_USER"
echo "Grafana password: $GRAFANA_PASSWORD"

# 端口转发 Grafana service
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```

访问 http://localhost:3000，使用上面获取的凭据登录。

登录后，可在 General 下找到 Dynamo 仪表板。

![Grafana 仪表板显示 Dynamo 指标](../../assets/img/grafana-k8s.png)

## Operator 指标

> **说明：** 上文描述的指标针对 Dynamo **应用**（前端、Worker）。Dynamo **Operator** 自身也暴露用于监控控制器调谐（reconciliation）、webhook 校验与资源清单的指标。
>
> 详情请见 **[Operator 指标指南](operator-metrics.md)**，了解 Operator 专属指标与 Operator 仪表板。
