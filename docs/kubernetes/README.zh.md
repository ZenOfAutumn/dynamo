---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 快速入门
---

在几分钟内让一个模型运行在 Kubernetes 上。

## 前置条件

- 带 GPU 节点的 Kubernetes 集群（v1.24+）
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)（v1.24+）
- 已安装 [Helm](https://helm.sh/docs/intro/install/)（v3.0+）
- 集群上已安装 [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html)
- 集群上有 HuggingFace token secret

### HuggingFace token secret

为模型下载创建 HuggingFace token secret。如果没有 token，请参阅 HuggingFace [token 指南](https://huggingface.co/docs/hub/en/security-tokens)。

```bash
export HF_TOKEN=<your-hf-token>

kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="$HF_TOKEN"
```

### GPU Operator 快速安装

如果尚未安装 GPU Operator：

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia --force-update
helm repo update nvidia
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator --create-namespace \
  --wait --timeout=600s
```

> [!TIP]
> 如果集群已自带 GPU 驱动（例如 GKE 配 `gpu-driver-version=latest`，或 AKS），添加：
> ```bash
> --set driver.enabled=false --set toolkit.enabled=false
> ```

### 详细安装

GPU Operator 是基础部署的唯一前置条件。如需 RDMA、Prometheus 或带 Grove/KAI Scheduler 的多节点调度等附加特性，请参阅[安装指南](installation-guide.md)。

> [!TIP]
> 如果你的 GPU SKU 与云提供商受支持，可以使用 [AICR](https://github.com/NVIDIA/aicr) 快速安装前置条件与 Dynamo Helm chart。

### 验证集群就绪

可选：验证集群已就绪：

```bash
./deploy/pre-deployment/pre-deployment-check.sh
```

## 安装 Dynamo

```bash
export NAMESPACE=dynamo-system
helm install dynamo-platform \
  oci://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/dynamo-platform \
  --version "1.0.2" \
  --namespace "$NAMESPACE" \
  --create-namespace
```

等待平台 Pod：

```bash
kubectl get pods -n $NAMESPACE
# Expected: dynamo-operator-*, etcd-*, nats-* pods all Running
```

## 部署你的第一个模型

使用 DynamoGraphDeploymentRequest（DGDR）部署 `Qwen/Qwen3-0.6B`。

DGDR 是部署模型的入口。它会为你的模型 / 硬件运行自动剖析，并创建一个自动配置的 DynamoGraphDeployment（DGD）。之后 DGDR 进入终态完成（类似 K8s Job），可以被清理。DGD 是持久存在并提供模型服务的资源。

```yaml
# qwen3-quickstart.yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: qwen3-quickstart
spec:
  model: Qwen/Qwen3-0.6B
  backend: auto
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-planner:1.0.2"
```

```bash
kubectl apply -f qwen3-quickstart.yaml -n $NAMESPACE
```

观察 DGDR 从 `Pending` → `Profiling` → `Deploying` → `Deployed` 的进度：

```bash
kubectl get dgdr qwen3-quickstart -n $NAMESPACE -w
```

> [!NOTE]
> Dynamo 支持 vLLM、TensorRT-LLM 与 SGLang 后端。设置 `backend: auto` 让 profiler 为你的模型与硬件选择最佳后端。详情见[后端指南](../backends/)。


## 发送请求

DGDR 显示为 `Deployed` 后：

```bash
# Find and port-forward the frontend
FRONTEND_SVC=$(kubectl get svc -n $NAMESPACE -o name | grep frontend | head -1)
kubectl port-forward "$FRONTEND_SVC" 8000:8000 -n $NAMESPACE &

# Send a request
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [{"role": "user", "content": "What is NVIDIA Dynamo?"}],
    "max_tokens": 200
  }' | python3 -m json.tool
```

## 清理

```bash
kubectl delete dgdr qwen3-quickstart -n $NAMESPACE
```

## 后续步骤

- **[安装指南](installation-guide.md)** — 云提供商配置、GPU Operator 细节、可选组件（Grove、RDMA、模型缓存、Prometheus）
- **[模型部署指南](model-deployment-guide.md)** — 策略选择、模型缓存、planner、多节点、常见陷阱
- **[DGDR 参考](dgdr.md)** — Spec 参考、生命周期阶段、监控命令、DGDR 与 DGD 对比
- **[创建部署](deployment/create-deployment.md)** — 手工编写 DGD spec 以实现完全控制
