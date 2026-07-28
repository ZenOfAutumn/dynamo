---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Azure Kubernetes Service (AKS)
---

# 在 AKS 上运行 Dynamo

本指南介绍如何搭建带 GPU 节点的 AKS 集群，并在其上部署 Dynamo。

## 前置条件

- 一个具备足够 GPU 虚拟机配额的有效 Azure 订阅
- 已安装并登录 [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)（`az`）
- 已安装 [kubectl](https://kubernetes.io/docs/tasks/tools/)
- 已安装 [Helm](https://helm.sh/docs/intro/install/) v3.0+

## 步骤 1：创建资源组与集群

```bash
az group create \
  --name <RESOURCE_GROUP> \
  --location <REGION>
```

```bash
az aks create \
  --resource-group <RESOURCE_GROUP> \
  --name <CLUSTER_NAME> \
  --node-count 1 \
  --generate-ssh-keys
```

然后获取访问凭据：

```bash
az aks get-credentials \
  --resource-group <RESOURCE_GROUP> \
  --name <CLUSTER_NAME>
```

## 步骤 2：添加 GPU 节点池

添加一个 GPU 节点池，并跳过驱动安装。`--skip-gpu-driver-install` 参数让 AKS 不再自行管理 GPU 驱动——驱动将由步骤 3 中的 NVIDIA GPU Operator 来管理。

```bash
az aks nodepool add \
  --resource-group <RESOURCE_GROUP> \
  --cluster-name <CLUSTER_NAME> \
  --name gpunp \
  --node-count 2 \
  --node-vm-size Standard_NC24ads_A100_v4 \
  --skip-gpu-driver-install
```

对于需要 RDMA 的工作负载（解耦推理），请使用 ND 系列虚拟机，例如 `Standard_ND96asr_v4` 或 `Standard_ND96isr_H100_v5`。在这类节点上需要的额外配置请参见 [RDMA / InfiniBand 指南](rdma-infiniband.md)。

完整的 GPU 虚拟机规格列表可参见 [GPU-optimized VM sizes](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes-gpu)。

## 步骤 3：安装 NVIDIA GPU Operator

GPU Operator 负责管理 GPU 节点上的 NVIDIA 驱动、容器工具集（container toolkit）、设备插件以及监控组件。

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
```

```bash
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator --create-namespace
```

确认 Pod 正常运行：

```bash
kubectl get pods -n gpu-operator
```

预期输出（节选）：

```text
NAMESPACE      NAME                                       READY   STATUS      RESTARTS   AGE
gpu-operator   gpu-feature-discovery-xxxxx                1/1     Running     0          2m
gpu-operator   gpu-operator-xxxxx                         1/1     Running     0          2m
gpu-operator   nvidia-container-toolkit-daemonset-xxxxx   1/1     Running     0          2m
gpu-operator   nvidia-cuda-validator-xxxxx                0/1     Completed   0          1m
gpu-operator   nvidia-device-plugin-daemonset-xxxxx       1/1     Running     0          2m
gpu-operator   nvidia-driver-daemonset-xxxxx              1/1     Running     0          2m
```

> [!NOTE]
> 如果你的解耦推理需要 RDMA / InfiniBand，**此时不要安装 GPU Operator**——RDMA 配置需要不同的 Helm values。完整步骤请参见 [RDMA / InfiniBand](rdma-infiniband.md)，其中包含正确的 GPU Operator 安装命令。

## 步骤 4：安装 Dynamo

按照 [安装指南](../../installation-guide.md) 安装 Dynamo Platform 并部署你的第一个模型。

## 进一步阅读

### [RDMA / InfiniBand](rdma-infiniband.md)

生产环境进行解耦推理时所必需。如果没有 RDMA，预填充与解码 worker 之间的 KV 缓存（KV cache）传输会回退到 TCP，并带来严重的延迟下降（首 token 时延约 98 秒，而使用 RDMA 时约 200–500 毫秒）。ND 系列虚拟机（例如 `Standard_ND96asr_v4`、`Standard_ND96isr_H100_v5`）自带 Mellanox ConnectX InfiniBand 网卡，但除了 GPU Operator 之外还需要额外配置：包括 NVIDIA Network Operator、用于 MOFED 驱动的 NicClusterPolicy、用于配置内核模块和 memlock 限制的 `ib-node-config` DaemonSet，以及用于把网卡暴露给 Pod 的 RDMA Shared Device Plugin。

### [模型缓存的存储](storage.md)

避免每个 Pod 启动时都独立下载模型权重。如果没有共享存储，加载大模型可能耗时数小时，并且在大规模启动时会触发 HuggingFace 限流。该指南覆盖 Azure Managed Lustre、Azure Files、Azure Disk 和 Local CSI 等方案，并针对不同缓存类型（模型缓存、编译缓存、性能缓存）给出了具体建议。

### [Azure Lustre CSI 驱动](azure-lustre-csi.md)

对于需要高吞吐共享访问的大规模多节点模型，这是推荐的存储方案。Azure Managed Lustre 默认未安装——本指南介绍如何安装并配置 Lustre CSI 驱动，让你之后可以将其作为 PVC 的 storage class 使用。

### [Spot 虚拟机](spot-vms.md)

通过运行在可抢占的 Spot 节点池上来显著降低 GPU 计算成本。AKS 会自动给 Spot 节点打上 `kubernetes.azure.com/scalesetpriority=spot:NoSchedule` 这个 taint，因此 Dynamo 组件需要显式声明对应的 toleration。Dynamo Helm chart 中已经内置了能够处理这一点的 `values-aks-spot.yaml`。

## 资源清理

```bash
# 删除所有 Dynamo Graph Deployment
kubectl delete dynamographdeployments.nvidia.com --all --all-namespaces

# 卸载 Dynamo Platform
export NAMESPACE="dynamo-system"
helm uninstall dynamo-platform -n $NAMESPACE

# 如果运行的是 Dynamo < 1.0 且 CRDs 在独立 chart 中：
# helm uninstall dynamo-crds -n $NAMESPACE
```

如果你想删除 GPU Operator，请参见 [Uninstalling the NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/uninstall.html)。

如果你想删除整个 AKS 集群，请参见 [Delete an AKS cluster](https://learn.microsoft.com/en-us/azure/aks/delete-cluster)。
