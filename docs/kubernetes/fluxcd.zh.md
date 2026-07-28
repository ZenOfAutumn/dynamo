---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: FluxCD
---

本节介绍如何使用 FluxCD 进行基于 GitOps 的 Dynamo 推理图部署。GitOps 使你能够以 Git 作为唯一可信源，声明式地管理 Dynamo 部署。我们将以[聚合式 vLLM 示例](../backends/vllm/README.md)演示工作流程。

## 前置条件

- 一个已安装 [Dynamo Kubernetes Platform](./installation-guide.md) 的 Kubernetes 集群
- 集群中已安装 [FluxCD](https://fluxcd.io/flux/installation/)
- 一个 Git 仓库，用于保存部署配置

## 工作流概述

Dynamo 部署的 GitOps 工作流程包含三个主要步骤：

1. 构建并推送 Dynamo Operator
2. 创建并提交一个 DynamoGraphDeployment 自定义资源以进行初次部署
3. 在后续更新时，构建新版本并更新该 CR

## 步骤 1：构建并推送 Dynamo Operator

首先，请按照[安装 Dynamo Kubernetes Platform](./installation-guide.md) 操作。

## 步骤 2：创建初始部署

在你的 Git 仓库中创建一个新文件（例如 `deployments/llm-agg.yaml`），内容如下：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: llm-agg
spec:
  pvcs:
    - name: vllm-model-storage
      size: 100Gi
  services:
    Frontend:
      replicas: 1
      envs:
      - name: SPECIFIC_ENV_VAR
        value: some_specific_value
    Processor:
      replicas: 1
      envs:
      - name: SPECIFIC_ENV_VAR
        value: some_specific_value
    VllmWorker:
      replicas: 1
      envs:
      - name: SPECIFIC_ENV_VAR
        value: some_specific_value
      # Add PVC for model storage
      volumeMounts:
        - name: vllm-model-storage
          mountPoint: /models
```

将该文件提交并推送到 Git 仓库。FluxCD 将检测到新的 CR，并在集群中创建初始的 Dynamo 部署。

## 步骤 3：更新现有部署

要更新流水线，只需更新对应的 DynamoGraphDeployment CRD。

Dynamo operator 会自动协调它。

## 监控部署

可使用以下命令监控部署状态：

```bash

export NAMESPACE=<namespace-with-the-dynamo-operator>

# Check the DynamoGraphDeployment status
kubectl get dynamographdeployment llm-agg -n $NAMESPACE
```
