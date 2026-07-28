---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: AKS 上的模型缓存存储
---

# AKS 上的模型缓存存储

要在 AKS 上实现分层存储，可以利用 Azure 上多种可用的存储选项。本指南介绍如何为每种 Dynamo 缓存类型选择合适的存储，以及如何配置 PVC。

## 可用存储选项

| 存储选项 | 性能 | 适用场景 |
|----------------|-------------|----------|
| Local CSI（临时盘） | 非常高 | 快速模型缓存、热重启 |
| [Azure Managed Lustre](azure-lustre-csi.md) | 极高 | 大型多节点模型、共享缓存 |
| [Azure Disk（托管磁盘）](https://learn.microsoft.com/en-us/azure/aks/azure-csi-driver-volume-provisioning?tabs=dynamic-volume-blob%2Cnfs%2Ckubernetes-secret%2Cnfs-3%2Cgeneral%2Cgeneral2%2Cdynamic-volume-disk%2Cgeneral-disk%2Cdynamic-volume-files%2Cgeneral-files%2Cgeneral-files2%2Cdynamic-volume-files-mid%2Coptimize%2Csmb-share&pivots=csi-disk#create-azure-disk-pvs-using-built-in-storage-classes) | 高 | 持久化的单写入者模型缓存 |
| [Azure Files](https://learn.microsoft.com/en-us/azure/aks/azure-csi-driver-volume-provisioning?tabs=dynamic-volume-blob%2Cnfs%2Ckubernetes-secret%2Cnfs-3%2Cgeneral%2Cgeneral2%2Cdynamic-volume-disk%2Cgeneral-disk%2Cdynamic-volume-files%2Cgeneral-files%2Cgeneral-files2%2Cdynamic-volume-files-mid%2Coptimize%2Csmb-share&pivots=csi-files#use-a-persistent-volume-for-storage) | 中等 | 共享的中小型模型 |
| [Azure Blob（通过 Fuse 或 init）](https://learn.microsoft.com/en-us/azure/aks/azure-csi-driver-volume-provisioning?tabs=dynamic-volume-blob%2Cnfs%2Ckubernetes-secret%2Cnfs-3%2Cgeneral%2Cgeneral2%2Cdynamic-volume-disk%2Cgeneral-disk%2Cdynamic-volume-files%2Cgeneral-files%2Cgeneral-files2%2Cdynamic-volume-files-mid%2Coptimize%2Csmb-share&pivots=csi-blob#create-a-pvc-using-built-in-storage-class) | 低-中等 | 冷模型存储、bootstrap 下载 |

> [!NOTE]
> Azure Managed Lustre 与 Local CSI（临时盘）默认未在 AKS 中安装，使用前需要额外设置。Azure Disk、Azure Files 与 Azure Blob CSI 驱动开箱即用。Lustre 的安装见 [Azure Lustre CSI Driver](azure-lustre-csi.md) 指南，或参考 [AKS CSI 存储选项文档](https://learn.microsoft.com/azure/aks/csi-storage-drivers) 获取所有内置驱动的概览。

Azure Managed Lustre 的安装见 [Azure Lustre CSI Driver](azure-lustre-csi.md) 指南。

## 按缓存类型推荐

- **Model Cache** — 原始模型构件、配置文件、tokenizer 等。
  - 持久化：必需，避免重复下载并降低冷启动延迟。
  - 推荐存储：Azure Managed Lustre（共享、高吞吐）或 Azure Disk（单副本、持久）。

- **Compilation Cache** — 后端特定的编译产物（例如 TensorRT 引擎）。
  - 持久化：可选。
  - 推荐存储：Local CSI（快速、节点本地）或 Azure Disk（GPU 配置固定时持久化）。

- **Performance Cache** — 运行时调优与剖析数据。
  - 持久化：不需要。
  - 推荐存储：Local CSI（或其他临时存储）。

## 检查可用 StorageClass

列出 AKS 集群中可用的 StorageClass：

```bash
kubectl get storageclass

NAME                           PROVISIONER                 RECLAIMPOLICY
azureblob-csi                  blob.csi.azure.com          Delete
azurefile                      file.csi.azure.com          Delete
azurefile-csi                  file.csi.azure.com          Delete
azurefile-csi-premium          file.csi.azure.com          Delete
azurefile-premium              file.csi.azure.com          Delete
default                        disk.csi.azure.com          Delete
managed                        disk.csi.azure.com          Delete
managed-csi                    disk.csi.azure.com          Delete
managed-csi-premium            disk.csi.azure.com          Delete
managed-premium                disk.csi.azure.com          Delete
sc.azurelustre.csi.azure.com   azurelustre.csi.azure.com   Retain
```

## 示例 PVC 配置

在不同 [recipe](https://github.com/ai-dynamo/dynamo/tree/main/recipes) 的 `cache.yaml` 中，你可以将 `storageClassName` 设为 AKS 集群中可用的存储选项：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-cache
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: "sc.azurelustre.csi.azure.com"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: compilation-cache
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: "azurefile-csi"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: perf-cache
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: "local-ephemeral"
```

## 另请参阅

- [Azure Lustre CSI Driver](azure-lustre-csi.md) — Azure Managed Lustre 的完整安装指南
- [模型缓存](../../model-caching.md) — 使用 Dynamo 配置模型缓存的完整说明，包括下载 Job 与挂载配置
- [AKS CSI 存储驱动](https://learn.microsoft.com/azure/aks/csi-storage-drivers) — Microsoft 关于所有内置 CSI 驱动的文档
