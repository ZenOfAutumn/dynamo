---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Installation Guide
---

本指南带你安装在 Kubernetes 上使用 Dynamo 部署模型所需的全部内容。请按顺序执行——每一步都基于上一步。

## 前置要求

开始前，请确认你已具备：

- 一个具备 GPU 节点的 **Kubernetes 集群（v1.24+）**。如需创建，可参考各云厂商指南：
  - [Amazon EKS](cloud-providers/eks/eks.md) | [Azure AKS](cloud-providers/aks/aks.md) | [Google GKE](cloud-providers/gke/gke.md)
  - 本地开发：[Minikube Setup](deployment/minikube.md)
- **kubectl** v1.24+ —— [Install kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- **Helm** v3.0+ —— [Install Helm](https://helm.sh/docs/intro/install/)

> [!IMPORTANT]
> **云厂商 GPU 驱动**：GPU Operator（步骤 1）会为你安装 GPU 驱动。在创建集群的 GPU 节点池时，**不要启用云厂商托管的 GPU 驱动安装**（例如跳过 AKS 的 GPU 驱动安装、不要使用 GKE 的 `--accelerator gpu-driver-version=latest`）。如果你的节点上已经有云厂商托管的驱动，请参阅 GPU Operator 步骤说明的处理方式。

校验你的工具：

```bash
kubectl version --client  # Should show v1.24+
helm version              # Should show v3.0+
```

## 概览

每次 Dynamo 部署都需要两个 Helm chart：**GPU Operator**（步骤 1）和 **Dynamo Platform**（步骤 2）。其余都是可选的。开始前先确定需要哪些可选组件，便于在步骤 3 一并安装。

| 可选组件 | 何时需要 | 用于 |
|-----------|-----------------|--------------|
| Grove + KAI Scheduler | 多节点或解耦推理 | 多节点部署（无 Grove 或 LWS 时 operator 会报错） |
| Network Operator / RDMA | 生产环境的解耦推理 | 可接受的 KV cache 传输性能（TCP 回退会导致约 200-500 倍劣化） |
| kube-prometheus-stack | 自动扩缩容、指标 dashboard 或 Planner | Planner `sla` 模式、KEDA/HPA 自动扩缩容 |
| 共享存储（模型缓存） | 大模型（>70B）或多副本 | 避免逐 pod 下载与 HuggingFace 限频 |

**Grove + KAI Scheduler** —— Grove 是默认的多节点编排器。在多节点部署中如果 Grove 与 [LeaderWorkerSet (LWS)](https://github.com/kubernetes-sigs/lws#installation) 都不可用，operator 会返回硬错误。KAI Scheduler 可选但推荐与 Grove 一同使用以实现 GPU 感知调度。详见 [Grove](grove.md)。

**Network Operator / RDMA** —— 没有 RDMA 时，解耦推理会自动回退到 TCP，但性能会严重下降（TTFT 约 98s vs. 有 RDMA 时的约 200-500ms）。任何生产解耦部署都需要它。配置因云厂商而异——参见 [Disaggregated Communication Guide](disagg-communication-guide.md) 与你的云厂商指南。

**kube-prometheus-stack** —— Planner 的 `sla` 优化模式必需（其会从 Prometheus 读取实时 TTFT/ITL 指标）。基于 KEDA/HPA 的自动扩缩容也必需。Planner 的 `throughput` 模式可使用内部队列深度信号在没有它时工作，但指标驱动的特性将不可用。详见 [Metrics](observability/metrics.md)。

**共享存储** —— 避免每个 pod 各自下载模型权重。否则，大模型（>70B）每个 pod 下载需要数小时，多副本还会触发 HuggingFace 限频。这并非由 operator 强制要求——属于运维考量。完整流程参见 [Model Caching](model-caching.md)。

## 步骤 1：安装 GPU Operator

[NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html) 自动化部署提供 GPU 所需的全部 NVIDIA 软件组件——驱动、容器工具包、device plugin 与监控。

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
```

```bash
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator --create-namespace
  # Uncomment if your nodes already have provider-managed GPU drivers installed:
  # --set driver.enabled=false
```

如果你的 GPU 节点上已经安装了云厂商托管的驱动（例如使用了 GKE 的 `--accelerator gpu-driver-version=latest`），请取消上面 `driver.enabled=false` 的注释，避免 operator 与已有驱动冲突。

> [!NOTE]
> 一些云厂商需要额外的 GPU Operator 配置。详见你所用厂商的指南：
> - [AKS GPU Operator setup](cloud-providers/aks/aks.md) —— 在节点池上跳过 AKS 托管的 GPU 驱动安装
> - [EKS GPU Operator setup](cloud-providers/eks/eks.md)
> - [GKE GPU Operator setup](cloud-providers/gke/gke.md) —— `LD_LIBRARY_PATH` 与 `ldconfig` 初始化要求

确认 GPU Operator 已运行：

```bash
kubectl get pods -n gpu-operator
# Expected: gpu-operator, nvidia-driver-daemonset, nvidia-device-plugin-daemonset, etc. all Running
```

## 步骤 2：安装 Dynamo Platform

设置环境变量：

```bash
export NAMESPACE=dynamo-system
export RELEASE_VERSION=1.0.2  # match a version from https://github.com/ai-dynamo/dynamo/releases
```

```bash
helm fetch https://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/dynamo-platform-$RELEASE_VERSION.tgz
helm install dynamo-platform dynamo-platform-$RELEASE_VERSION.tgz \
  --namespace $NAMESPACE \
  --create-namespace
  # Note: add \ to --create-namespace above when uncommenting any optional flags below
  #
  # Grove + KAI Scheduler — uncomment if using multinode or disaggregated inference.
  # Option A (install=true): Dynamo installs and manages Grove/KAI as bundled subcharts (dev/testing):
  # --set "global.grove.install=true" \
  # --set "global.kai-scheduler.install=true" \
  # Option B (enabled=true): Grove/KAI are already installed externally (production):
  # --set "global.grove.enabled=true" \
  # --set "global.kai-scheduler.enabled=true" \
  #
  # kube-prometheus-stack — uncomment if Prometheus is installed (required for Planner sla mode and autoscaling):
  # --set "dynamo-operator.dynamo.metrics.prometheusEndpoint=http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090"
```

> [!TIP]
> 所有 `helm install` 命令都可以通过你自己的 values 文件定制：`helm install ... -f your-values.yaml`

> [!TIP]
> **共享 / 多租户集群**：如果集群级 Dynamo operator 已在运行，**请勿** 再次安装。可用以下命令检查：
> ```bash
> kubectl get clusterrolebinding -o json | \
>   jq -r '.items[] | select(.metadata.name | contains("dynamo-operator-manager")) |
>   "Cluster-wide operator found in namespace: \(.subjects[0].namespace)"'
> ```

> [!WARNING]
> **命名空间受限模式**（`namespaceRestriction.enabled=true`）已弃用，将在未来版本中移除。所有新部署请使用默认的集群级模式。

确认 Dynamo platform 已运行：

```bash
# Check CRDs
kubectl get crd | grep dynamo
# Expected: dynamographdeployments, dynamocomponentdeployments, dynamographdeploymentrequests, etc.

# Check operator and platform pods
kubectl get pods -n $NAMESPACE
# Expected: dynamo-operator-*, etcd-*, nats-* pods all Running
```

## 步骤 3：安装可选组件

上面的 Dynamo 安装命令为每个可选组件都包含了被注释的 flag。请先安装组件，再取消对应 flag 的注释，然后运行步骤 2 的 `helm install`（或者如果已经安装过 Dynamo，则用 `helm upgrade --reuse-values` 加上对应的 flag）。

### 多节点：

多节点部署需要 Grove + KAI Scheduler，或一套替代的编排方案（LeaderWorkerSet + Volcano）来为跨节点工作负载启用 gang scheduling。编排器选择与配置详见 [Multinode Deployment Guide](./deployment/multinode-deployment.md)。

#### Grove + KAI Scheduler

启用 Grove 与 KAI Scheduler 有两种方式，由你在 Dynamo 安装命令中取消注释的 flag 决定：

- **`install=true`** —— Dynamo 把 Grove/KAI 作为打包子 chart 安装并管理。最简路径；推荐用于开发/测试。
- **`enabled=true`** —— 告诉 Dynamo Grove/KAI 已经被外部安装并管理。当你单独安装 Grove/KAI（例如独立管理其生命期或在 namespace 间共享）时使用。推荐用于生产。

对于 `enabled=true` 路径，请先单独安装 Grove 与 KAI Scheduler。安装步骤参考 [Grove installation guide](https://github.com/NVIDIA/grove/blob/main/docs/installation.md) 与 [KAI Scheduler deployment guide](https://github.com/NVIDIA/KAI-Scheduler)。

> [!NOTE]
> **兼容性矩阵：**
>
> | dynamo-platform | kai-scheduler | Grove |
> |-----------------|---------------|-------|
> | 1.0.x           | >= v0.13.0    | >= v0.1.0-alpha.6 |
> | 1.1.x           | >= v0.13.4    | >= v0.1.0-alpha.8 |

#### LWS + Volcano

如果你不使用 Grove 实现多节点，可以用 [LeaderWorkerSet (LWS)](https://lws.sigs.k8s.io/docs/installation/)（>= v0.7.0）配合 [Volcano](https://volcano.sh/en/docs/installation/) 实现 gang scheduling。两者必须先安装好再部署多节点工作负载。

1. 安装 Volcano：

```bash
helm repo add volcano-sh https://volcano-sh.github.io/helm-charts
helm repo update
helm install volcano volcano-sh/volcano -n volcano-system --create-namespace
```

2. 安装 LWS（>= v0.7.0），并启用 Volcano gang scheduling：

```bash
export LWS_VERSION=0.8.0
helm install lws oci://registry.k8s.io/lws/charts/lws \
  --version=$LWS_VERSION \
  --namespace lws-system \
  --create-namespace \
  --set gangSchedulingManagement.schedulerProvider=volcano \
  --wait --timeout 300s
```

配置项参考 [LWS docs](https://lws.sigs.k8s.io/docs/) 和 [Volcano docs](https://volcano.sh/en/docs/)；编排器选择见 [Multinode Deployment Guide](./deployment/multinode-deployment.md)。

### Network Operator / RDMA

RDMA 配置因云厂商而异。传输选项、UCX 配置与性能预期参见 [Disaggregated Communication Guide](disagg-communication-guide.md)；配置步骤参考你所用厂商的指南：

- [AKS — InfiniBand + Network Operator](cloud-providers/aks/rdma-infiniband.md)
- [EKS — EFA device plugin](cloud-providers/eks/eks.md)（也参见 [EFA configuration guide](disagg-communication-guide.md#aws-efa-configuration)）
- [GKE — GPUDirect-TCPXO](cloud-providers/gke/gke.md)

### kube-prometheus-stack

请在运行 Dynamo 安装命令之前先安装 Prometheus，便于一次性把 endpoint 配置进去：

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set-json 'prometheus.prometheusSpec.podMonitorNamespaceSelector={}' \
  --set-json 'prometheus.prometheusSpec.probeNamespaceSelector={}'
```

然后取消 Dynamo 安装命令中 `prometheusEndpoint` 一行的注释。Dynamo operator 会自动为其组件创建 PodMonitor。dashboard 搭建与可用指标见 [Metrics](observability/metrics.md)，Grafana Loki + Alloy 日志栈见 [Logging](observability/logging.md)。

### 模型缓存的共享存储

配置一个 `ReadWriteMany` 的 PVC，让所有 pod 共享下载好的模型权重，而非各自下载。无需 Dynamo chart flag——存储在你的部署 spec 中配置。配置因云厂商而异：

- [AKS — Azure Files / Managed Lustre](cloud-providers/aks/storage.md)
- [EKS — EFS](cloud-providers/eks/efs.md)
- GKE — Cloud Filestore（见 [GKE guide](cloud-providers/gke/gke.md)）

对于模型频繁更新的大型集群，可考虑使用 [Model Express](model-caching.md#option-2-model-express-p2p-distribution) 进行 P2P 模型分发。下载 Job 与挂载配置等完整流程见 [Model Caching](model-caching.md)。

## 步骤 4：部署前检查

运行部署前检查脚本以校验集群是否就绪：

```bash
./deploy/pre-deployment/pre-deployment-check.sh
```

该脚本检查 kubectl 连通性、默认 StorageClass 配置、GPU 节点可用性以及 GPU Operator 状态。详见 [Pre-Deployment Checks](https://github.com/ai-dynamo/dynamo/tree/main/deploy/pre-deployment/README.md)。

## 后续步骤

集群已就绪。请参考 **[Model Deployment Guide](model-deployment-guide.md)** 使用 DGDR 部署模型。

## 常见问题排查

**"VALIDATION ERROR: Cannot install cluster-wide Dynamo operator"**

```
VALIDATION ERROR: Cannot install cluster-wide Dynamo operator.
Found existing namespace-restricted Dynamo operators in namespaces: ...
```

原因：在已有 namespace 受限 operator 的共享集群上尝试集群级安装。

解决：把已有的 namespace 受限 operator 迁移到集群级模式。namespace 受限模式已弃用。

**CRD 已存在**

原因：在已存在 CRD 的集群上再次安装（共享集群常见）。

解决：CRD 由 Helm chart 自动安装。如遇冲突，可用 `kubectl get crd | grep dynamo` 查看现有 CRD。

**Pod 起不来？**
```bash
kubectl describe pod <pod-name> -n $NAMESPACE
kubectl logs <pod-name> -n $NAMESPACE
```

**Bitnami etcd "unrecognized" 镜像？**

```bash
ERROR: Original containers have been substituted for unrecognized ones.
```

向 helm install 命令追加：
```bash
--set "etcd.image.repository=bitnamilegacy/etcd" --set "etcd.global.security.allowInsecureImages=true"
```

**干净卸载？**

```bash
# Uninstall the platform
helm uninstall dynamo-platform --namespace $NAMESPACE

# List Dynamo CRDs
kubectl get crd | grep "dynamo.*nvidia.com"

# Delete each CRD
kubectl delete crd <crd-name>
```

## 进阶：从源码构建

如果你需要为 Dynamo 贡献代码或使用 main 分支上未发布的最新特性：

```bash
# 1. Set registry environment
export DOCKER_SERVER=nvcr.io/nvidia/ai-dynamo/  # or your registry
export DOCKER_USERNAME='$oauthtoken'
export DOCKER_PASSWORD=<YOUR_NGC_CLI_API_KEY>
export IMAGE_TAG=$RELEASE_VERSION

# 2. Build and push operator image
cd deploy/operator
docker build -t $DOCKER_SERVER/kubernetes-operator:$IMAGE_TAG . && docker push $DOCKER_SERVER/kubernetes-operator:$IMAGE_TAG
cd -

# 3. Create namespace and image pull secret (only if using a private registry)
kubectl create namespace $NAMESPACE
kubectl create secret docker-registry docker-imagepullsecret \
  --docker-server=$DOCKER_SERVER \
  --docker-username=$DOCKER_USERNAME \
  --docker-password=$DOCKER_PASSWORD \
  --namespace=$NAMESPACE

# 4. Install from local chart
cd deploy/helm/charts
helm dep build ./platform/
helm install dynamo-platform ./platform/ \
  --namespace "$NAMESPACE" \
  --set "dynamo-operator.controllerManager.manager.image.repository=$DOCKER_SERVER/kubernetes-operator" \
  --set "dynamo-operator.controllerManager.manager.image.tag=$IMAGE_TAG" \
  --set "dynamo-operator.imagePullSecrets[0].name=docker-imagepullsecret"
```

## 参考

- [Helm Chart Configuration](https://github.com/ai-dynamo/dynamo/tree/main/deploy/helm/charts/platform/README.md)
- [Create Custom Deployments](./deployment/create-deployment.md)
- [Dynamo Operator Details](./dynamo-operator.md)
- [Model Express Server](https://github.com/ai-dynamo/modelexpress)
