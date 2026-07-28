---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: AKS Spot VMs
---

# 在 AKS Spot VM 上运行 Dynamo

[Azure Spot VMs](https://azure.microsoft.com/en-us/products/virtual-machines/spot) 为 GPU 工作负载提供显著的成本节省，但可能在任意时刻被 Azure 回收。本指南介绍在 Spot VM 节点池上调度 Dynamo 所需的配置。

## AKS 如何为 Spot 节点打污点

当某个节点池使用 Spot VM 时，AKS 会自动为该池中的所有节点打上以下污点：

```yaml
kubernetes.azure.com/scalesetpriority=spot:NoSchedule
```

这可以防止常规工作负载默认落到 Spot 节点上。任何应当在 Spot 节点上运行的 Pod 必须显式容忍此污点。

## 必需的 Toleration

在所有应当运行于 Spot 节点上的工作负载中加入以下 toleration：

```yaml
tolerations:
  - key: kubernetes.azure.com/scalesetpriority
    operator: Equal
    value: spot
    effect: NoSchedule
```

## 在 Spot 节点上部署 Dynamo

Dynamo 平台 Helm chart 包含一个面向 Spot VM 部署的预构建 values 文件——[`examples/deployments/AKS/values-aks-spot.yaml`](https://github.com/ai-dynamo/dynamo/blob/main/examples/deployments/AKS/values-aks-spot.yaml)——它为所有 Dynamo 组件添加了所需的 toleration：

- Dynamo operator 控制器管理器
- Webhook CA inject 与证书生成 Job
- etcd
- NATS
- MPI SSH 密钥生成 Job
- 其他核心 Dynamo 平台 Pod

使用 Spot values 文件安装 Dynamo：

```bash
helm install dynamo-platform dynamo-platform-${RELEASE_VERSION}.tgz \
  --namespace dynamo-system \
  --create-namespace \
  -f ./values-aks-spot.yaml
```

升级现有安装：

```bash
helm upgrade dynamo-platform dynamo-platform-${RELEASE_VERSION}.tgz \
  --namespace dynamo-system \
  -f ./values-aks-spot.yaml
```

## 创建 Spot GPU 节点池

向现有 AKS 集群添加 Spot GPU 节点池：

```bash
az aks nodepool add \
  --resource-group <RESOURCE_GROUP> \
  --cluster-name <CLUSTER_NAME> \
  --name spotgpunp \
  --node-count 2 \
  --node-vm-size Standard_NC24ads_A100_v4 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --skip-gpu-driver-install
```

`--spot-max-price -1` 表示愿意支付最高至 on-demand 价格（推荐）。`--eviction-policy Delete` 会从节点池中删除被回收的节点；如果希望在被回收时保留节点状态，请使用 `Deallocate`。

## 另请参阅

- [Azure Spot VMs 概览](https://learn.microsoft.com/en-us/azure/virtual-machines/spot-vms)
- [在 AKS 中使用 Spot VM](https://learn.microsoft.com/en-us/azure/aks/spot-node-pool)
