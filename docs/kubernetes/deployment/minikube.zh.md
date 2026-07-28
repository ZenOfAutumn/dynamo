---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Minikube 安装
---

没有 Kubernetes 集群？没问题！你可以使用 Minikube 搭建一个本地开发环境。本指南将带你完成在本地运行 Dynamo Kubernetes Platform 所需的全部设置。

## 1. 安装 Minikube
首先，开始安装 Minikube。请按照适合你操作系统的官方[Minikube 安装指南](https://minikube.sigs.k8s.io/docs/start/)进行安装。

## 2. 配置 GPU 支持（可选）
计划使用 GPU 加速的工作负载？你需要在 Minikube 中配置 GPU 支持。继续之前请按照 [Minikube GPU 指南](https://minikube.sigs.k8s.io/docs/tutorials/nvidia/) 来设置 NVIDIA GPU 支持。

> [!TIP]
> 如果计划使用 GPU 工作负载，请务必在启动 Minikube 之前完成 GPU 支持的配置！


## 3. 启动 Minikube
是时候启动你的本地集群了！

```bash
# Start Minikube with GPU support (if configured)
minikube start --driver docker --container-runtime docker --gpus all --memory=16000mb --cpus=8

# Enable required addons
minikube addons enable istio-provisioner
minikube addons enable istio
minikube addons enable storage-provisioner-rancher
```

## 4. 验证安装
让我们确认一切运行正常！

```bash
# Check Minikube status
minikube status

# Verify Istio installation
kubectl get pods -n istio-system

# Verify storage class
kubectl get storageclass
```

## 后续步骤

本地环境搭建好之后，你可以继续按照 [Dynamo Kubernetes Platform 安装指南](../installation-guide.md) 将平台部署到本地集群。

