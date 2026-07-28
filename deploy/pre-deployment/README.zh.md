<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# 部署前检查脚本

本目录包含一个部署前检查脚本，用于校验你的 Kubernetes 集群是否满足部署 Dynamo
的要求。

- 关于 NCCL 测试，请参考 [NCCL 测试](https://docs.nebius.com/kubernetes/gpu/nccl-test#run-tests)
  获取更多细节。

要查看最新的部署前检查说明，请参见
[main 分支版本的本 README](https://github.com/ai-dynamo/dynamo/blob/main/deploy/pre-deployment/README.md)。

## 使用方法

部署 Dynamo 前先运行本检查脚本：

```bash
./pre-deployment-check.sh
```

## 它会检查什么

脚本会执行若干项检查并给出详细汇总：

### 1. kubectl 连通性
- 校验是否安装了 `kubectl` 并且能够连上你的 Kubernetes 集群

### 2. 默认 StorageClass
- 校验集群中是否已配置默认 StorageClass
- 如果没有找到默认 StorageClass：
  - 列出集群上所有可用的 StorageClass 及其完整信息
  - 给出将某个 StorageClass 设为默认的示例命令
  - 给出 Kubernetes 官方文档链接以供详细参考

### 3. 集群 GPU 资源
- 通过标签 `nvidia.com/gpu.present=true` 检查集群中是否存在启用 GPU 的节点

## 示例输出

### 完整脚本输出示例：
```
========================================
  Dynamo Pre-Deployment Check Script
========================================

--- Checking kubectl connectivity ---
✅ kubectl is available and cluster is accessible

--- Checking for default StorageClass ---
❌ No default StorageClass found

Dynamo requires a default StorageClass for persistent volume provisioning.
Please configure a default StorageClass before proceeding with deployment.

Available StorageClasses in your cluster:
NAME                                 PROVISIONER                     RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
my-default-storage-class (default)   compute.csi.mock                Delete          WaitForFirstConsumer   true                   65d
fast-ssd-storage                     kubernetes.io/gce-pd            Delete          Immediate              true                   30d

To set a StorageClass as default, use the following command:
kubectl patch storageclass <storage-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

Example with your first available StorageClass:
kubectl patch storageclass my-default-storage-class -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

For more information on managing default StorageClasses, visit:
https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/

--- Checking cluster gpu resources ---
✅ Found 17 gpu node(s) in the cluster
Node information:

--- Pre-Deployment Check Summary ---
✅ kubectl Connectivity: PASSED
❌ Default StorageClass: FAILED
✅ Cluster Resources: PASSED

Summary: 2 passed, 1 failed
❌ 1 pre-deployment check(s) failed.
Please address the issues above before proceeding with deployment.
```

### 当所有检查通过时：
```
========================================
  Dynamo Pre-Deployment Check Script
========================================


--- Checking kubectl connectivity ---
✅ kubectl is available and cluster is accessible

--- Checking for default StorageClass ---
✅ Default StorageClass found
  - NAME                               PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
my-default-storage-class (default)   compute.csi.mock   Delete          WaitForFirstConsumer   true                   65d

--- Checking cluster gpu resources ---
✅ Found 17 gpu node(s) in the cluster
Node information:


--- Pre-Deployment Check Summary ---
✅ kubectl Connectivity: PASSED
✅ Default StorageClass: PASSED
✅ Cluster Resources: PASSED

Summary: 3 passed, 0 failed
🎉 All pre-deployment checks passed!
Your cluster is ready for Dynamo deployment.
```

## 检查状态汇总

脚本会给出综合性汇总，显示每一项检查的状态：

| 检查名称 | 描述 | 通过 / 失败状态 |
|------------|-------------|------------------|
| **kubectl 连通性** | 校验 kubectl 安装情况以及对集群的访问 | ✅ PASSED / ❌ FAILED |
| **默认 StorageClass** | 检查默认 StorageClass 注解 | ✅ PASSED / ❌ FAILED |
| **集群资源** | 校验 GPU 节点的可用性 | ✅ PASSED / ❌ FAILED |

## 设置默认 StorageClass

如需将某个 StorageClass 设为默认，可使用以下命令：

```bash
kubectl patch storageclass <storage-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

将 `<storage-class-name>` 替换为目标 StorageClass 的名称。

## 故障排查

### 多个默认 StorageClass
如果有多个 StorageClass 被标记为默认，脚本会发出警告：
```
⚠️  Warning: Multiple default StorageClasses detected
   This may cause unpredictable behavior. Consider having only one default StorageClass.
```

要从某个 StorageClass 上去掉默认注解：
```bash
kubectl patch storageclass <storage-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### 未找到 GPU 节点
如果未找到 GPU 节点，请确认你的集群中存在带有 `nvidia.com/gpu.present=true`
标签的节点。

### 没有可用的 StorageClass
如果集群中完全没有 StorageClass，你需要：
1. 安装一个存储 provisioner（例如：云厂商存储、本地存储等）
2. 创建合适的 StorageClass 资源
3. 将其中一个标记为默认

## 参考

关于管理默认 StorageClass 的更多信息，请参见：
[Kubernetes 文档 —— 修改默认 StorageClass](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/)
