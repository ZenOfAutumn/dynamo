---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Azure Lustre CSI Driver for AKS
---

# AKS 上的 Azure Lustre CSI 驱动

本指南介绍如何在 AKS 集群上安装并配置 [Azure Lustre CSI 驱动](https://github.com/kubernetes-sigs/azurelustre-csi-driver)，从而使 Dynamo 工作负载能够使用 Azure Managed Lustre（AMLFS）文件系统作为高性能模型存储。

## 前置条件

**AKS 集群要求**
- Kubernetes 1.21 或更高版本
- 节点池必须使用 **Ubuntu** 操作系统镜像——不支持 Windows 与 Azure Linux（CBL Mariner）节点
- 仅支持 AKS 这一种 Kubernetes 发行版（不支持自管集群）

**工具**
- Azure CLI（`az`）
- `kubectl`

**网络连通性**

AKS 与 AMLFS 文件系统之间必须能够互相访问。支持以下两种网络拓扑：

- **VNet 对等连接（peering）**：在独立的 VNet 中部署 AKS，并将其与 AMLFS 所在的 VNet 建立对等连接。AKS 自身的基础设施 VNet 位于自动创建的资源组 `MC_<aks-rg>_<aks-name>_<region>` 中。
- **共享 VNet**：使用 AKS 的 "Bring your own VNet" 特性，把 AKS 部署到 AMLFS VNet 内部一个专用子网中。注意不要与 AMLFS 共用同一个子网。

> [!WARNING]
> 即使共用同一个 VNet，也不要让 AKS 节点和 AMLFS 文件系统位于同一个子网。

## 步骤 1：连接到你的 AKS 集群

```bash
az login

az aks get-credentials \
  --subscription <SUBSCRIPTION_ID> \
  --resource-group <AKS_RESOURCE_GROUP> \
  --name <AKS_CLUSTER_NAME>

kubectl config current-context
```

## 步骤 2：安装 CSI 驱动

目前没有 Helm chart，请通过官方提供的 shell 脚本安装：

```bash
# 安装最新版本
curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azurelustre-csi-driver/main/deploy/install-driver.sh | bash -s main

# 或安装指定版本
curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azurelustre-csi-driver/main/deploy/install-driver.sh | bash -s v0.3.1
```

该脚本会把 CSI 控制器（2 副本的 Deployment）以及节点插件（DaemonSet）部署到 `kube-system` 命名空间，并等待它们就绪。

**验证安装：**

```bash
# 控制器 Pod —— 期望状态为 2/2 或 3/3 Running
kubectl get -n kube-system pod -l app=csi-azurelustre-controller

# 节点插件 Pod —— 期望每个节点上都为 3/3 Running
kubectl get -n kube-system pod -l app=csi-azurelustre-node -o wide
```

## 步骤 3：配置存储

根据你是否已经存在 AMLFS 文件系统，有两种供应（provisioning）模式可选。

### 方案 A：静态供应（已有 AMLFS 文件系统）

如果你想使用已有的 Azure Managed Lustre 文件系统，请选择该方式。如果还没有，请先创建一个，然后再配置 CSI 驱动来使用它。

#### 创建一个 Azure Managed Lustre 文件系统

**1. 注册资源提供方（仅首次需要）：**

```bash
az provider register --namespace Microsoft.StorageCache
# 等待状态变为 "Registered"
az provider show --namespace Microsoft.StorageCache --query "registrationState"
```

**2. 在创建文件系统之前先校验子网：**

该子网必须专供 AMLFS 使用（不要与 AKS 节点或其他资源共用），且容量需要足够承载该文件系统。先检查需求：

```bash
# 根据计划使用的 SKU 与容量获取所需的子网大小
az amlfs get-subnets-size \
  --sku AMLFS-Durable-Premium-250 \
  --storage-capacity 16

# 校验你选定的子网是否满足要求
az amlfs check-amlfs-subnet \
  --filesystem-subnet /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RG>/providers/Microsoft.Network/virtualNetworks/<VNET>/subnets/<SUBNET> \
  --sku AMLFS-Durable-Premium-250 \
  --location <REGION> \
  --storage-capacity 16
```

**3. 为 AMLFS 创建一个独立子网：**

AMLFS 必须有自己的子网——它不能与 AKS 节点共用。可以在 AKS VNet 中（或在其对等连接的 VNet 中）新建子网：

```bash
# 获取节点资源组，并检查是否使用了自定义 VNet 子网
az aks show \
  --name <AKS_CLUSTER_NAME> \
  --resource-group <AKS_RESOURCE_GROUP> \
  --query "{vnet: agentPoolProfiles[0].vnetSubnetId, nodeRG: nodeResourceGroup}"
```

如果 `vnet` 字段非空，说明你的集群使用 Azure CNI 配合自定义 VNet——下面请使用该 VNet 的名称和资源组。

如果 `vnet` 为 `null`，说明 AKS 在节点资源组中自管 VNet。可以通过下列命令找到它：

```bash
az network vnet list \
  --resource-group <NODE_RESOURCE_GROUP> \
  --query "[].{name:name, addressPrefixes:addressSpace.addressPrefixes}"
```

列出已有子网，从中找出还未占用的 CIDR 段：

```bash
az network vnet subnet list \
  --resource-group <VNET_RESOURCE_GROUP> \
  --vnet-name <AKS_VNET_NAME> \
  --query "[].{name:name, prefix:addressPrefix}"
```

在 VNet 的地址空间内挑选一个不重叠的 CIDR 段。`get-subnets-size` 返回的 `filesystemSubnetSize` 表示所需的 IP 数量；Azure 还会在每个子网保留 5 个 IP，请把这部分加进来再确定前缀长度（例如 `filesystemSubnetSize: 8` → 共需 13 个 IP → 至少使用 `/28`，即 16 个地址）。

然后创建专门的 AMLFS 子网：

```bash
az network vnet subnet create \
  --name amlfs-subnet \
  --resource-group <VNET_RESOURCE_GROUP> \
  --vnet-name <AKS_VNET_NAME> \
  --address-prefix <CIDR>
```

下一步中需要使用完整的子网资源 ID：
`/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<VNET_RESOURCE_GROUP>/providers/Microsoft.Network/virtualNetworks/<AKS_VNET_NAME>/subnets/amlfs-subnet`

**4. 创建文件系统：**

```bash
az amlfs create \
  --name <AMLFS_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --location <REGION> \
  --sku AMLFS-Durable-Premium-250 \
  --storage-capacity 16 \
  --zones "[1]" \
  --filesystem-subnet /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RG>/providers/Microsoft.Network/virtualNetworks/<VNET>/subnets/<SUBNET> \
  --maintenance-window "{dayOfWeek:Sunday,timeOfDayUtc:'22:00'}"
```

整个过程通常需要 **10–20 分钟**。可以加 `--no-wait` 让命令立即返回，并使用 `az amlfs show` 来轮询状态。

**可选 SKU：**

| SKU | 最小容量 | 吞吐 |
|-----|----------|------------|
| `AMLFS-Durable-Premium-40` | 48 TiB | 每 TiB 40 MB/s |
| `AMLFS-Durable-Premium-125` | 16 TiB | 每 TiB 125 MB/s |
| `AMLFS-Durable-Premium-250` | 8 TiB | 每 TiB 250 MB/s |
| `AMLFS-Durable-Premium-500` | 4 TiB | 每 TiB 500 MB/s |

**5. 获取 MGS IP 地址：**

```bash
az amlfs show \
  --name <AMLFS_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --query "{mgsAddress: clientInfo.mgsAddress, mountCommand: clientInfo.mountCommand}"
```

把 `mgsAddress` 的值填入下面的 StorageClass。也可以在 Azure 门户中你的文件系统页面的 **Client connection** 选项卡找到该地址。

**StorageClass：**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurelustre-static
provisioner: azurelustre.csi.azure.com
parameters:
  mgs-ip-address: <MGS_IP_ADDRESS>  # From portal > Client connection
reclaimPolicy: Retain
volumeBindingMode: Immediate
mountOptions:
  - noatime
  - flock
```

**PersistentVolumeClaim：**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-lustre
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: <AMLFS_STORAGE_CAPACITY>  # Match your filesystem size, e.g. 16Ti
  storageClassName: azurelustre-static
```

```bash
kubectl apply -f storageclass.yaml
kubectl apply -f pvc.yaml
```

### 方案 B：动态供应（自动创建 AMLFS 文件系统）

需要驱动 v0.3.0 及以上版本。当 PVC 创建时，驱动会自动创建一个 AMLFS 集群——耗时 **10 分钟以上**。

**额外的 IAM 权限**（需要授予 kubelet 的托管标识，且必须在创建 PVC 之前授予）：

```
Microsoft.StorageCache/amlFilesystems/read
Microsoft.StorageCache/amlFilesystems/write
Microsoft.StorageCache/amlFilesystems/delete
Microsoft.StorageCache/checkAmlFSSubnets/action
Microsoft.StorageCache/getRequiredAmlFSSubnetsSize/*
Microsoft.Network/virtualNetworks/subnets/read
Microsoft.Network/virtualNetworks/subnets/join/action
Microsoft.ManagedIdentity/userAssignedIdentities/assign/action
```

或者直接授予更宽泛的角色：订阅范围的 **Reader**、目标资源组的 **Contributor** 以及 VNet 范围的 **Network Contributor**。

**可选 SKU：**

| SKU | 吞吐 |
|-----|------------|
| `AMLFS-Durable-Premium-40` | 每 TiB 40 MB/s |
| `AMLFS-Durable-Premium-125` | 每 TiB 125 MB/s（最小 48 TiB） |
| `AMLFS-Durable-Premium-250` | 每 TiB 250 MB/s |
| `AMLFS-Durable-Premium-500` | 每 TiB 500 MB/s |

**StorageClass：**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurelustre-dynamic
provisioner: azurelustre.csi.azure.com
parameters:
  sku-name: "AMLFS-Durable-Premium-125"
  zone: "1"                          # Availability zone: "1", "2", or "3"
  maintenance-day-of-week: "Sunday"
  maintenance-time-of-day-utc: "22:00"
  # Optional overrides (defaults to AKS cluster values):
  # location: "eastus"
  # resource-group-name: "my-rg"
  # vnet-name: "my-vnet"
  # subnet-name: "my-subnet"
reclaimPolicy: Delete   # WARNING: deletes the AMLFS cluster when PVC is deleted — use Retain in production
volumeBindingMode: Immediate
mountOptions:
  - noatime
  - flock
```

**PersistentVolumeClaim：**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-lustre-dynamic
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 48Ti   # Minimum for AMLFS-Durable-Premium-125
  storageClassName: azurelustre-dynamic
```

```bash
kubectl apply -f storageclass-dynamic.yaml
kubectl apply -f pvc-dynamic.yaml

# 监控供应进度（耗时 10 分钟以上）
kubectl describe pvc pvc-lustre-dynamic
```

## 故障排查

**Pod 卡在 `ContainerCreating`**

```bash
kubectl describe pod <pod-name>
# Look for volume mount errors in Events

kubectl logs -n kube-system -l app=csi-azurelustre-node -c azurelustre --tail=50
```

**PVC 卡在 `Pending`（动态供应）**

```bash
kubectl describe pvc <pvc-name>
# Check Events for authorization errors — kubelet identity may lack IAM permissions
```

**节点无法挂载** —— 请确认操作系统镜像为 Ubuntu：

```bash
kubectl get nodes -o custom-columns="NAME:.metadata.name,OS:.status.nodeInfo.osImage"
```

## 相关链接

- [Azure Managed Lustre CSI Driver — GitHub](https://github.com/kubernetes-sigs/azurelustre-csi-driver)
- [Use Azure Managed Lustre with AKS — Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-managed-lustre/use-csi-driver-kubernetes)
