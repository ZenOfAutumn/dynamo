# 在 AKS 上部署 Dynamo

本指南介绍如何在 Azure Kubernetes Service（AKS）上部署 Dynamo 并运行 LLM 推理。你将了解如何搭建带 GPU 节点的 AKS 集群、安装所需组件，并部署你的第一个模型。

## 前置条件

开始之前，请确保你具备：

- 一个有效的 Azure 订阅
- 足够的 Azure GPU VM 配额
- 已安装 [kubectl](https://kubernetes.io/docs/tasks/tools/)
- 已安装 [Helm](https://helm.sh/docs/intro/install/)

## 步骤 1：创建带 GPU 节点的 AKS 集群

如果还没有 AKS 集群，可使用 [Azure CLI](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli)、[Azure PowerShell](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-powershell) 或 [Azure 门户](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-portal) 创建一个。

确保 AKS 集群中存在带 GPU 节点的 node pool。请按 [在 Azure Kubernetes Service（AKS）上为计算密集型工作负载使用 GPU](https://learn.microsoft.com/en-us/azure/aks/use-nvidia-gpu?tabs=add-ubuntu-gpu-node-pool#skip-gpu-driver-installation) 指南创建启用 GPU 的 node pool。

**重要：** 建议在创建 node pool 时**跳过 GPU 驱动安装**，因为下一步的 NVIDIA GPU Operator 会负责完成。

## 步骤 2：安装 NVIDIA GPU Operator

AKS 集群配置好启用了 GPU 的 node pool 后，请安装 NVIDIA GPU Operator。该 operator 自动化部署并管理 Kubernetes 集群中提供 GPU 所需的所有 NVIDIA 软件组件的生命周期，包括驱动、容器工具集、设备插件和监控工具。

请按 [安装 NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html) 指南在 AKS 集群上安装 GPU Operator。

你应能看到类似下面的输出。注意这并非完整输出；还应有更多 pod 在运行。最重要的是确认 GPU Operator 的 pod 都处于 `Running` 状态。

```bash
NAMESPACE     NAME                                                          READY   STATUS    RESTARTS   AGE
gpu-operator  gpu-feature-discovery-xxxxx                                   1/1     Running   0          2m
gpu-operator  gpu-operator-xxxxx                                            1/1     Running   0          2m
gpu-operator  nvidia-container-toolkit-daemonset-xxxxx                      1/1     Running   0          2m
gpu-operator  nvidia-cuda-validator-xxxxx                                   0/1     Completed 0          1m
gpu-operator  nvidia-device-plugin-daemonset-xxxxx                          1/1     Running   0          2m
gpu-operator  nvidia-driver-daemonset-xxxxx                                 1/1     Running   0          2m
```

## 步骤 3：部署 Dynamo Kubernetes Operator

请按 [将推理图部署到 Kubernetes](../../../docs/kubernetes/README.md) 指南在 AKS 集群上安装 Dynamo。

确认 Dynamo 的 pod 已运行：

```bash
kubectl get pods -n dynamo-system

# 期望输出：
# NAME                                                              READY   STATUS    RESTARTS   AGE
# dynamo-platform-dynamo-operator-controller-manager-xxxxxxxxxx     2/2     Running   0          2m50s
# dynamo-platform-etcd-0                                            1/1     Running   0          2m50s
# dynamo-platform-nats-0                                            2/2     Running   0          2m50s
# dynamo-platform-nats-box-xxxxxxxxxx                               1/1     Running   0          2m51s
```

## 步骤 4：部署并测试模型

请按 [部署模型/工作流](../../../docs/kubernetes/installation-guide.md#next-steps) 指南在 AKS 集群上部署并测试模型。

## AKS 上模型缓存与运行时数据的存储选项

要实现分层存储，可以利用 Azure 提供的多种存储选项，例如：

| 存储选项 | 性能 | 最适用于 |
|----------------|-------------|----------|
| 本地 CSI（临时盘） | 极高 | 快速模型缓存、热重启 |
| [Azure Managed Lustre](https://learn.microsoft.com/en-us/azure/azure-managed-lustre/use-csi-driver-kubernetes) | 极高 | 大型多节点模型，共享缓存 |
| [Azure Disk（托管磁盘）](https://learn.microsoft.com/en-us/azure/aks/azure-csi-driver-volume-provisioning?tabs=dynamic-volume-blob%2Cnfs%2Ckubernetes-secret%2Cnfs-3%2Cgeneral%2Cgeneral2%2Cdynamic-volume-disk%2Cgeneral-disk%2Cdynamic-volume-files%2Cgeneral-files%2Cgeneral-files2%2Cdynamic-volume-files-mid%2Coptimize%2Csmb-share&pivots=csi-disk#create-azure-disk-pvs-using-built-in-storage-classes) | 高 | 持久化的单写入者模型缓存 |
| [Azure Files](https://learn.microsoft.com/en-us/azure/aks/azure-csi-driver-volume-provisioning?tabs=dynamic-volume-blob%2Cnfs%2Ckubernetes-secret%2Cnfs-3%2Cgeneral%2Cgeneral2%2Cdynamic-volume-disk%2Cgeneral-disk%2Cdynamic-volume-files%2Cgeneral-files%2Cgeneral-files2%2Cdynamic-volume-files-mid%2Coptimize%2Csmb-share&pivots=csi-files#use-a-persistent-volume-for-storage) | 中 | 共享的小/中型模型 |
| [Azure Blob（通过 Fuse 或 init）](https://learn.microsoft.com/en-us/azure/aks/azure-csi-driver-volume-provisioning?tabs=dynamic-volume-blob%2Cnfs%2Ckubernetes-secret%2Cnfs-3%2Cgeneral%2Cgeneral2%2Cdynamic-volume-disk%2Cgeneral-disk%2Cdynamic-volume-files%2Cgeneral-files%2Cgeneral-files2%2Cdynamic-volume-files-mid%2Coptimize%2Csmb-share&pivots=csi-blob#create-a-pvc-using-built-in-storage-class) | 低–中 | 冷模型存储、引导下载 |

注：Azure Managed Lustre 与本地 CSI（临时盘）在 AKS 中默认未安装，使用前需额外配置。Azure Disk、Azure Files 与 Azure Blob CSI 驱动开箱可用。详情参见 [AKS CSI 存储选项文档](https://learn.microsoft.com/azure/aks/csi-storage-drivers)。

在不同 [recipes](https://github.com/ai-dynamo/dynamo/tree/main/recipes) 的 cache.yaml 中，可以将 storageClassName 设置为你 AKS 集群中可用的预定义存储选项：

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
对 Dynamo 缓存的存储选项推荐如下：

- 模型缓存（Model Cache），存储原始模型工件、配置文件、tokenizer 等。<br>
  - 持久化：必需，避免重复下载并降低冷启动延迟。<br>
  - 推荐存储：Azure Managed Lustre（共享、高吞吐）或 Azure Disk（单副本、持久化）。

- 编译缓存（Compilation Cache），存储后端特定的编译产物（例如 TensorRT 引擎）。<br>
  - 持久化：可选<br>
  - 推荐存储：本地 CSI（快速、节点本地）或 Azure Disk（GPU 配置固定时持久化）。

- 性能缓存（Performance Cache），存储运行时调优与剖析数据。<br>
  - 持久化：不必需<br>
  - 推荐存储：本地 CSI（或其他临时存储）。

cache.yaml 示例：
```bash
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

## 在 AKS Spot VM GPU node pool 上运行

当你在 AKS 上使用启用了 GPU 的 [Spot VM](https://azure.microsoft.com/en-us/products/virtual-machines/spot) node pool 部署 Dynamo 时，AKS 会自动给这些 Spot 节点打上以下 taint，以默认避免普通工作负载被调度到上面。
```bash
kubernetes.azure.com/scalesetpriority=spot:NoSchedule
```
由于这些 taint，工作负载（包括 Dynamo CRD controller、Platform 组件以及任何 GPU 工作负载）都必须在其 Helm chart 中包含下方的 toleration。否则 Kubernetes 不会把 pod 调度到 Spot VM node pool，GPU 资源也将闲置。
```bash
tolerations:
  - key: kubernetes.azure.com/scalesetpriority
    operator: Equal
    value: spot
    effect: NoSchedule
```
要把 Dynamo 平台组件与 job 调度到这些节点上，请使用提供的 dynamo/examples/deployments/AKS/values-aks-spot.yaml，它包含以下组件所需的所有 toleration：
- Dynamo operator controller manager
- Webhook CA inject 与证书生成 job
- etcd
- NATS
- MPI SSH 密钥生成 job
- 其他核心 Dynamo 平台 pod

使用以下命令通过 AKS Spot values 文件安装或升级 Dynamo：
```bash
helm install dynamo-platform dynamo-platform-${RELEASE_VERSION}.tgz \
  --namespace dynamo-system \
  --create-namespace \
  -f ./values-aks-spot.yaml
```
或
```bash
helm upgrade dynamo-platform dynamo-platform-${RELEASE_VERSION}.tgz \
  --namespace dynamo-system \
  -f ./values-aks-spot.yaml
```

## 清理资源

如果你想清理本指南中创建的 Dynamo 资源，可以执行：

```bash
# 删除所有 Dynamo Graph Deployment
kubectl delete dynamographdeployments.nvidia.com --all --all-namespaces

# 卸载 Dynamo Platform 与 CRD
helm uninstall dynamo-platform -n dynamo-kubernetes
helm uninstall dynamo-crds -n default
```

这会下线 Dynamo 部署及所有相关资源。

如果你想删除 GPU Operator，请参考 [卸载 NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/uninstall.html) 指南。

如果你想删除整个 AKS 集群，请参考 [删除 AKS 集群](https://learn.microsoft.com/en-us/azure/aks/delete-cluster) 指南。
