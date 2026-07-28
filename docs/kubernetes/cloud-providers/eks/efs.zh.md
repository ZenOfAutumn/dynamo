---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Amazon EFS Setup for EKS
---

# 为 Amazon EKS 创建 Amazon EFS 文件系统

本指南介绍如何创建 Amazon EFS 文件系统并把它接入 EKS 集群（cluster）。EFS CSI Driver 已在创建集群时通过 `eksctl.yaml` 作为 addon 安装。现在我们需要创建实际的文件系统并使其可用于 Kubernetes 工作负载。

该文件系统将被 Dynamo 用于在节点（node）之间存储共享的模型权重与编译缓存。

## 先决条件

- 已按照 [EKS 指南](eks.md)创建 EKS 集群
- 已设置环境变量：

```bash
export AWS_REGION="us-east-1"
export CLUSTER_NAME="ai-dynamo"
export DYNAMO_NAMESPACE="dynamo-system"
```

## 获取 VPC 与子网信息

获取与 EKS 集群关联的 VPC ID：

```bash
export VPC_ID=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text)
```

获取 VPC 的 CIDR 范围（用于 security group 规则）：

```bash
export VPC_CIDR=$(aws ec2 describe-vpcs \
  --vpc-ids $VPC_ID \
  --query "Vpcs[0].CidrBlock" \
  --output text)
```

## 为 EFS 创建一个 Security Group

创建一个允许 VPC 内部 NFS 流量（端口 2049）的 security group：

```bash
export EFS_SG_ID=$(aws ec2 create-security-group \
  --group-name dynamo-efs-sg \
  --description "Security group for EFS access from EKS" \
  --vpc-id $VPC_ID \
  --region $AWS_REGION \
  --query "GroupId" \
  --output text)
```

添加一条来自 VPC CIDR 的入站 NFS 流量规则：

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $EFS_SG_ID \
  --protocol tcp \
  --port 2049 \
  --cidr $VPC_CIDR \
  --region $AWS_REGION
```

## 创建 EFS 文件系统

```bash
export EFS_FS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted \
  --region $AWS_REGION \
  --tags Key=Name,Value=dynamo-efs \
  --query "FileSystemId" \
  --output text)
```

等待文件系统变为可用：

```bash
aws efs describe-file-systems \
  --file-system-id $EFS_FS_ID \
  --region $AWS_REGION \
  --query "FileSystems[0].LifeCycleState" \
  --output text
```

继续之前应看到 `available`。

## 创建挂载目标（Mount Targets）

挂载目标允许 EKS 节点访问 EFS 文件系统。在节点运行的每个子网中都需要一个挂载目标。

获取 EKS 集群使用的子网 ID：

```bash
export SUBNET_IDS=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --query "cluster.resourcesVpcConfig.subnetIds[]" \
  --output text)

echo "Subnet IDs: $SUBNET_IDS"
```

为每个子网创建一个挂载目标：

```bash
for SUBNET_ID in $(echo "$SUBNET_IDS" | tr '\t' '\n'); do
  echo "Creating mount target in subnet: $SUBNET_ID"
  aws efs create-mount-target \
    --file-system-id $EFS_FS_ID \
    --subnet-id $SUBNET_ID \
    --security-groups $EFS_SG_ID \
    --region $AWS_REGION 2>/dev/null || echo "  Mount target already exists or subnet is in a duplicate AZ (this is OK)"
done
```

> [!NOTE]
> EFS 在每个可用区只允许一个挂载目标。如果多个子网在同一可用区，命令会因重复而失败，这是预期且可以安全忽略的。

验证挂载目标已可用：

```bash
aws efs describe-mount-targets \
  --file-system-id $EFS_FS_ID \
  --region $AWS_REGION \
  --query "MountTargets[*].{SubnetId:SubnetId,AZ:AvailabilityZoneName,State:LifeCycleState}" \
  --output table
```

继续之前请等待所有挂载目标在 State 列显示 `available`。

## 创建 Kubernetes StorageClass

创建一个使用 EFS CSI driver 进行动态供给的 StorageClass：

```bash
kubectl apply -f - << EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc-dynamic
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: "${EFS_FS_ID}"
  directoryPerms: "777"
  uid: "1000"
  gid: "1000"
EOF
```

## 创建 PersistentVolumeClaim

我们创建三个独立的 PVC，因为不同的 Dynamo recipe 示例分别引用它们：
* `model-cache` 存储下载的模型权重（如来自 HuggingFace）。
* `compilation-cache` 存储 vLLM/TRT-LLM 的编译产物。
* `perf-cache` 存储基准测试 trace 与性能结果。

```bash
# 如果尚不存在，先创建用于 Dynamo 的命名空间
kubectl create namespace ${DYNAMO_NAMESPACE}

# 创建 PVC
kubectl apply -f - << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-cache
  namespace: ${DYNAMO_NAMESPACE}
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: "efs-sc-dynamic"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: compilation-cache
  namespace: ${DYNAMO_NAMESPACE}
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: "efs-sc-dynamic"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: perf-cache
  namespace: ${DYNAMO_NAMESPACE}
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: "efs-sc-dynamic"
EOF
```

> [!NOTE]
> EFS 是弹性的，PVC 中的 `storage` 值是 Kubernetes 要求的，但并不限制实际存储。EFS 会自动按需扩缩。

## 验证

确认 PVC 已绑定：

```bash
kubectl get pvc -n ${DYNAMO_NAMESPACE}
```

你应当看到类似以下输出：

```text
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     VOLUMEATTRIBUTESCLASS   AGE
compilation-cache   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWX            efs-sc-dynamic   <unset>                 41s
model-cache         Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWX            efs-sc-dynamic   <unset>                 42s
perf-cache          Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWX            efs-sc-dynamic   <unset>                 41s
```

## 清理

不再需要时删除 EFS 资源：

```bash
# 删除 Kubernetes 资源
kubectl delete pvc model-cache compilation-cache perf-cache -n ${DYNAMO_NAMESPACE}
kubectl delete storageclass efs-sc-dynamic

# 删除挂载目标
for MT_ID in $(aws efs describe-mount-targets --file-system-id $EFS_FS_ID --region $AWS_REGION --query "MountTargets[*].MountTargetId" --output text); do
  aws efs delete-mount-target --mount-target-id $MT_ID --region $AWS_REGION
done

# 删除 EFS 文件系统
aws efs delete-file-system --file-system-id $EFS_FS_ID --region $AWS_REGION

# 删除 security group
aws ec2 delete-security-group --group-id $EFS_SG_ID --region $AWS_REGION
```
