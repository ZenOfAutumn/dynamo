---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Google Kubernetes Engine (GKE)
---

# 在 GKE 上部署 Dynamo

## 前置条件

### 安装 gcloud CLI
https://cloud.google.com/sdk/docs/install

### 创建 GKE 集群

```bash
export PROJECT_ID=<>
export REGION=<>
export ZONE=<>
export CLUSTER_NAME=<>
export CLUSTER_MACHINE_TYPE=n2-standard-4
export NODE_POOL_MACHINE_TYPE=g2-standard-24
export GPU_TYPE=nvidia-l4
export GPU_COUNT=2
export CPU_NODE=2
export GPU_NODE=2
export DISK_SIZE=200

gcloud container clusters create ${CLUSTER_NAME} \
 	--project=${PROJECT_ID} \
 	--location=${ZONE} \
	--subnetwork=default \
    --disk-size=${DISK_SIZE} \
	--machine-type=${CLUSTER_MACHINE_TYPE} \
 	--num-nodes=${CPU_NODE}
```

#### 创建 GPU 节点池

```bash
gcloud container node-pools create gpu-pool \
 	--accelerator type=${GPU_TYPE},count=${GPU_COUNT},gpu-driver-version=latest \
 	--project=${PROJECT_ID} \
 	--location=${ZONE} \
 	--cluster=${CLUSTER_NAME} \
	--machine-type=${NODE_POOL_MACHINE_TYPE} \
    --disk-size=${DISK_SIZE} \
    --num-nodes=${GPU_NODE} \
    --enable-autoscaling \
    --min-nodes=1 \
    --max-nodes=3
```

### 克隆 Dynamo GitHub 仓库

**注意：** 请确保 GitHub 分支/提交版本与 Dynamo platform 及 VLLM 容器版本一致。

```bash
git clone https://github.com/ai-dynamo/dynamo.git

# Checkout to the desired branch
git checkout release/0.6.0
```

### 为 GKE 设置环境变量

```bash
export NAMESPACE=dynamo-system
kubectl create namespace $NAMESPACE
kubectl config set-context --current --namespace=$NAMESPACE

export HF_TOKEN=<HF_TOKEN>
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n ${NAMESPACE}
```

## 安装 Dynamo Kubernetes Platform

[查看安装步骤](../../installation-guide.md#overview)

安装完成后，验证安装：

**预期输出**

```bash
kubectl get pods
NAME                                                              READY   STATUS             RESTARTS   AGE
dynamo-platform-dynamo-operator-controller-manager-69b9794fpgv9   2/2     Running            0          4m27s
dynamo-platform-etcd-0                                            1/1     Running            0          4m27s
dynamo-platform-nats-0                                            2/2     Running            0          4m27s
```

## 部署推理图

我们将在 Dynamo 平台上部署一个 LLM 模型。这里以 `Qwen/Qwen3-0.6B` 模型，使用 VLLM 与解耦（disaggregated）部署为例。

在部署 yaml 中，可能需要 / 可以做以下调整：

- **（必需）** 在 decoder 容器的 args 中添加修改 `LD_LIBRARY_PATH` 与 `PATH` 的命令，使 GKE 能找到正确的 GPU 驱动
- 将 VLLM 镜像更换为 NGC 上目标版本
- 在 metadata 中添加 namespace
- 调整 GPU/CPU 的 request 与 limits
- 修改要部署的模型

更多配置请参考 https://github.com/ai-dynamo/dynamo/tree/main/examples/deployments/GKE/vllm

### yaml 文件中的关键配置说明
请注意，根据 [在 GKE 中运行 GPU](https://cloud.google.com/kubernetes-engine/docs/how-to/gpus)，需要在 GKE 中正确设置 `LD_LIBRARY_PATH`。

下面这段需要出现在部署 `yaml` 的 `args` 字段中：

```bash
export LD_LIBRARY_PATH=/usr/local/nvidia/lib64:$LD_LIBRARY_PATH
export PATH=$PATH:/usr/local/nvidia/bin:/usr/local/nvidia/lib64
/sbin/ldconfig
```

例如，参考 [`examples/deployments/GKE/vllm/disagg.yaml`](https://github.com/ai-dynamo/dynamo/blob/main/examples/deployments/GKE/vllm/disagg.yaml) 中的：

```yaml
metadata:
  name: vllm-disagg
  namespace: dynamo-system
spec:
  services:
    Frontend:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.0
    VllmDecodeWorker:
​​      resources:
        limits:
          gpu: "3"
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.0
          args:
            - |
            export LD_LIBRARY_PATH=/usr/local/nvidia/lib64:$LD_LIBRARY_PATH
            export PATH=$PATH:/usr/local/nvidia/bin:/usr/local/nvidia/lib64
            /sbin/ldconfig
            python3 -m dynamo.vllm --model Qwen/Qwen3-0.6B
```

## 部署模型

```bash
cd dynamo/examples/deployments/GKE/vllm

kubectl apply -f disagg_gke.yaml -n ${NAMESPACE}
```

**部署成功后的预期输出**

```bash
kubectl get pods
NAME                                                              READY   STATUS    RESTARTS   AGE
dynamo-platform-dynamo-operator-controller-manager-c665684ssqkx   2/2     Running   0          65m
dynamo-platform-etcd-0                                            1/1     Running   0          65m
dynamo-platform-nats-0                                            2/2     Running   0          65m
vllm-disagg-frontend-5954ddc4dd-4w2cb                             1/1     Running   0          11m
vllm-disagg-vllmdecodeworker-77844cfcff-ddn4v                     1/1     Running   0          11m
vllm-disagg-vllmprefillworker-55d5b74b4f-zrskh                    1/1     Running   0          11m
```

## 测试部署

```bash
export DEPLOYMENT_NAME=vllm-disagg

# Find the frontend pod
export FRONTEND_POD=$(kubectl get pods -n ${NAMESPACE} | grep "${DEPLOYMENT_NAME}-frontend" | sort -k1 | tail -n1 | awk '{print $1}')

# Forward the pod's port to localhost
kubectl port-forward deployment/vllm-disagg-frontend  8000:8000 -n ${NAMESPACE}

# disagg
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [
    {
        "role": "user",
        "content": "In the heart of Eldoria, an ancient land of boundless magic and mysterious creatures, lies the long-forgotten city of Aeloria. Once a beacon of knowledge and power, Aeloria was buried beneath the shifting sands of time, lost to the world for centuries. You are an intrepid explorer, known for your unparalleled curiosity and courage, who has stumbled upon an ancient map hinting at ests that Aeloria holds a secret so profound that it has the potential to reshape the very fabric of reality. Your journey will take you through treacherous deserts, enchanted forests, and across perilous mountain ranges. Your Task: Character Background: Develop a detailed background for your character. Describe their motivations for seeking out Aeloria, their skills and weaknesses, and any personal connections to the ancient city or its legends. Are they driven by a quest for knowledge, a search for lost familt clue is hidden."
    }
    ],
    "stream":false,
    "max_tokens": 30
  }'
```

### 响应

```json
{"id":"chatcmpl-bd0670d9-0342-4eea-97c1-99b69f1f931f","choices":[{"index":0,"message":{"content":"Okay, here's a detailed character background for your intrepid explorer, tailored to fit the premise of Aeloria, with a focus on a","refusal":null,"tool_calls":null,"role":"assistant","function_call":null,"audio":null},"finish_reason":"stop","logprobs":null}],"created":1756336263,"model":"Qwen/Qwen3-0.6B","service_tier":null,"system_fingerprint":null,"object":"chat.completion","usage":{"prompt_tokens":190,"completion_tokens":29,"total_tokens":219,"prompt_tokens_details":null,"completion_tokens_details":null}}
```
