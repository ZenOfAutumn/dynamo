---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Dynamo Operator
---

## 概述

Dynamo Operator 是一个 Kubernetes Operator，用于简化 DynamoGraph 的部署、配置和生命周期管理。它会自动调谐自定义资源，使集群始终保持你期望的状态。该 Operator 非常适合那些希望通过声明式 YAML 和 Kubernetes 原生工具来管理复杂部署的用户。

## 架构

- **Operator 部署：**
  以 Kubernetes `Deployment` 的形式部署在指定命名空间中。

- **控制器（Controllers）：**
  - `DynamoGraphDeploymentController`：监听 `DynamoGraphDeployment` 自定义资源（CR），编排整图部署。
  - `DynamoComponentDeploymentController`：监听 `DynamoComponentDeployment` CR，处理单个组件的部署。
  - `DynamoModelController`：监听 `DynamoModel` CR，管理模型生命周期（例如加载 LoRA adapter）。

- **工作流：**
  1. 用户或 API server 创建一个自定义资源。
  2. 对应的控制器检测到变更，触发一次调谐（reconciliation）。
  3. 创建或更新相应的 Kubernetes 资源（Deployment、Service 等）以匹配 CR spec。
  4. 状态字段被更新，反映当前实际状态。

## 部署模式

Dynamo Operator 支持三种部署模式，以适应不同的集群环境和使用场景：

### 1. 集群级模式（默认，推荐）

Operator 会监控并管理集群中**所有命名空间**下的 DynamoGraph 资源。

**适用场景：**
- 你拥有完整的集群管理员权限
- 你希望集中管理所有 Dynamo 工作负载
- 在专属集群上的标准生产部署

---

### 2. 命名空间限定模式（已废弃）

> **已废弃：** 命名空间限定模式（`namespaceRestriction.enabled=true`）已被废弃，未来版本将移除。请改用集群级模式，新部署不要使用该模式。

Operator 只监控并管理**特定命名空间**内的 DynamoGraph 资源。同时它会创建一个 lease 标记，以告知集群级 Operator 这里已存在一个本地 Operator。

**适用场景：**
- 你处于共享或多租户集群
- 你只拥有命名空间级别的权限
- 你希望以隔离方式测试新版本的 Operator
- 你需要避免与其他 Operator 冲突

**安装：**
```bash
helm install dynamo-platform dynamo-platform-${RELEASE_VERSION}.tgz \
  --namespace my-namespace \
  --create-namespace \
  --set dynamo-operator.namespaceRestriction.enabled=true
```

---

### 3. 混合模式（已废弃）

> **已废弃：** 混合模式依赖于命名空间限定的 Operator，这种 Operator 已被废弃，未来版本将移除。请改为只使用一个集群级 Operator。

**集群级 Operator** 管理大多数命名空间，同时在某些特定命名空间中运行**一个或多个命名空间限定的 Operator**（例如用于测试新版本）。集群级 Operator 会通过 lease 标记自动发现并跳过这些命名空间。

**适用场景：**
- 在生产中运行稳定版本的 Operator 工作负载
- 在隔离命名空间中测试新版本 Operator，且不影响生产
- 渐进式发布 Operator 升级
- 在生产集群上运行开发/预发环境

**工作原理：**
1. 命名空间限定的 Operator 在自己所在的命名空间创建一个名为 `dynamo-operator-namespace-scope` 的 lease。
2. 集群级 Operator 监控所有命名空间中的此类 lease 标记。
3. 集群级 Operator 自动跳过任何带有 lease 标记的命名空间。
4. 如果命名空间限定的 Operator 停止运行，其 lease 会过期（默认 TTL 为 30 秒）。
5. 之后集群级 Operator 会自动接管对该命名空间的管理。

**配置示例：**

```bash
# 1. Install cluster-wide operator (production, v1.0.0)
helm install dynamo-platform dynamo-platform-${RELEASE_VERSION}.tgz \
  --namespace dynamo-system \
  --create-namespace

# 2. Install namespace-scoped operator (testing, v2.0.0-beta)
helm install dynamo-test dynamo-platform-${RELEASE_VERSION}.tgz \
  --namespace test-namespace \
  --create-namespace \
  --set dynamo-operator.namespaceRestriction.enabled=true \
  --set dynamo-operator.controllerManager.manager.image.tag=v2.0.0-beta
```

**可观测性：**

```bash
# List all namespaces with local operators
kubectl get lease -A --field-selector metadata.name=dynamo-operator-namespace-scope

# Check which operator version is running in a namespace
kubectl get lease -n my-namespace dynamo-operator-namespace-scope \
  -o jsonpath='{.spec.holderIdentity}'
```


## 自定义资源定义（CRDs）

Dynamo 提供以下自定义资源：

- **DynamoGraphDeployment（DGD）**：部署完整的推理流水线
- **DynamoComponentDeployment（DCD）**：部署单个组件
- **DynamoModel**：管理模型生命周期（例如加载 LoRA adapter）

关于 Dynamo 自定义资源定义的完整技术 API 参考，请见：

**📖 [Dynamo CRD API Reference](./api-reference.md)**

针对用户的 DynamoModel 部署与管理使用指南，请见：

**📖 [Managing Models with DynamoModel Guide](./deployment/dynamomodel-guide.md)**

## Webhooks

Dynamo Operator 使用 **Kubernetes 准入 webhook（admission webhook）** 在自定义资源被持久化到集群之前进行实时校验和变更。Webhook 是 Operator 必备的组件，可在 API server 层面立即拒绝无效配置。

**主要特性：**
- ✅ 所有 webhook 类型共享同一套证书基础设施
- ✅ 默认在所有环境下自动生成并轮换证书
- ✅ 可选地集成 cert-manager（适合自定义 PKI 场景）
- ✅ 对关键字段强制不可变约束

完整的 webhook 文档、证书管理与故障排查请见：

**📖 [Webhooks Guide](./webhooks.md)**

## 可观测性

Dynamo Operator 通过 Prometheus 指标和 Grafana 仪表盘提供完善的可观测性。你可以监控以下方面：

- **控制器性能**：调谐循环耗时、按资源类型划分的成功率与错误率
- **Webhook 活动**：校验性能、准入率、拒绝模式
- **资源清单**：按状态和命名空间统计当前所管理的资源数量
- **运行健康度**：控制器和 webhook 的成功率与健康指标

### 指标采集

指标会自动暴露在 Operator 的 `/metrics` 端点（默认端口 8443），并由 Prometheus 通过 ServiceMonitor 进行抓取。当你通过 Helm 安装 Operator 时，ServiceMonitor 会自动创建（由 `metricsService.enabled` 控制，默认为 `true`）。

### Grafana 仪表盘

我们提供了预制的 Grafana 仪表盘用于可视化 Operator 指标。该仪表盘包含：

- **调谐指标**：按资源类型划分的速率、耗时（P95）和错误数
- **Webhook 指标**：按资源类型与操作类型划分的请求速率、耗时（P95）和拒绝数
- **资源清单**：按状态和命名空间统计的 DynamoGraphDeployment 数量
- **运行健康度**：控制器与 webhook 的成功率指示

完整的部署步骤和指标参考请见：

**📖 [Operator Metrics Guide](./observability/operator-metrics.md)**

## 安装

### 使用 Helm 快速安装

```bash
# Set environment
export NAMESPACE=dynamo-system
export RELEASE_VERSION=0.x.x # any version of Dynamo 0.3.2+ listed at https://github.com/ai-dynamo/dynamo/releases

# Install Platform (includes operator)
helm fetch https://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/dynamo-platform-${RELEASE_VERSION}.tgz
helm install dynamo-platform dynamo-platform-${RELEASE_VERSION}.tgz --namespace ${NAMESPACE} --create-namespace
```

> **说明：** 命名空间限定模式与混合模式均已废弃。所有新部署请使用集群级模式。如需向后兼容的配置，请参见上文的 [部署模式](#部署模式)。

### 从源码构建

```bash
# Set environment
export NAMESPACE=dynamo-system
export DOCKER_SERVER=your-registry.com/  # your container registry
export IMAGE_TAG=latest

# Build operator image
cd deploy/operator
docker build -t $DOCKER_SERVER/kubernetes-operator:$IMAGE_TAG \
  --build-context snapshot=../snapshot \
  --build-arg DOCKER_PROXY="" \
  .
docker push $DOCKER_SERVER/kubernetes-operator:$IMAGE_TAG
cd -

# Install platform with custom operator image (CRDs are automatically installed by the chart)
cd deploy/helm/charts
helm install dynamo-platform ./platform/ \
  --namespace ${NAMESPACE} \
  --create-namespace \
  --set "dynamo-operator.controllerManager.manager.image.repository=${DOCKER_SERVER}/kubernetes-operator" \
  --set "dynamo-operator.controllerManager.manager.image.tag=${IMAGE_TAG}" \
  --set dynamo-operator.imagePullSecrets[0].name=docker-imagepullsecret
```

详细的安装选项请见 [安装指南](./installation-guide.md)。


## 开发

- **代码结构：**

该 Operator 基于 Kubebuilder 与 operator-sdk 构建，目录结构如下：

- `controllers/`：调谐逻辑
- `api/v1alpha1/`：CRD 类型
- `config/`：清单与 Helm chart


## 参考资料

- [Kubernetes Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Operator SDK](https://sdk.operatorframework.io/)
- [Helm Best Practices for CRDs](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/)
