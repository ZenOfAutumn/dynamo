---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Model Caching
subtitle: Download models once and share across all pods in a Kubernetes cluster
---

大语言模型（LLM）的下载可能需要数分钟。如果没有缓存，每个 pod 都会独立下载完整模型，浪费带宽并延迟启动。Dynamo 支持两种方式确保模型只下载一次，并在集群中共享。

## 选项 1：PVC + 下载 Job（推荐）

最简单的方式：创建一个共享 PVC，运行一次性 Job 下载模型，然后在 DynamoGraphDeployment 中挂载该 PVC。

这是当前所有 Dynamo recipes 使用的模式。

### 步骤 1：创建共享 PVC

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
```

<Note>
需要 `ReadWriteMany` 访问模式，使多个 pod 能同时挂载该 PVC。请确保你的 storage class 支持 RWX（如 NFS、CephFS 或云厂商共享文件系统）。
</Note>

### 步骤 2：下载模型

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: model-download
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: downloader
          image: python:3.12-slim
          command: ["sh", "-c"]
          args:
            - |
              pip install huggingface_hub hf_transfer
              HF_HUB_ENABLE_HF_TRANSFER=1 huggingface-cli download \
                $MODEL_NAME --revision $MODEL_REVISION
          env:
            - name: MODEL_NAME
              value: "Qwen/Qwen3-0.6B"
            - name: MODEL_REVISION
              value: "main"
            - name: HF_HOME
              value: /cache/huggingface
          envFrom:
            - secretRef:
                name: hf-token-secret
          volumeMounts:
            - name: model-cache
              mountPath: /cache/huggingface
      volumes:
        - name: model-cache
          persistentVolumeClaim:
            claimName: model-cache
```

### 查找快照路径

Job 完成后，模型按 HuggingFace 缓存目录布局存储：

```
hub/models--<org>--<model>/snapshots/<commit-hash>/
```

例如，`meta-llama/Llama-3.1-70B-Instruct` 会变为：

```
hub/models--meta-llama--Llama-3.1-70B-Instruct/snapshots/9d3b8e0f71f8c1e0f9b7c2a3d4e5f6a7b8c9d0e1/
```

下载 Job 完成后，可通过以下方式查询确切的 commit hash：

```bash
kubectl run find-snapshot --rm -it --image=busybox --restart=Never \
  --overrides='{
    "spec": {
      "volumes": [{"name": "c", "persistentVolumeClaim": {"claimName": "model-cache"}}],
      "containers": [{
        "name": "f", "image": "busybox",
        "command": ["find", "/c/hub", "-mindepth", "3", "-maxdepth", "3", "-type", "d"],
        "volumeMounts": [{"name": "c", "mountPath": "/c"}]
      }]
    }
  }'
```

或者，在 HuggingFace Hub 模型页面的 **Files and versions** 下查阅 commit hash。

DGDR 规约中 `pvcModelPath` 字段需要这个路径（参见 [模型部署指南——模型缓存](model-deployment-guide.md#model-caching)）。

### 步骤 3：在 DynamoGraphDeployment 中挂载

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-deployment
spec:
  pvcs:
    - create: false
      name: model-cache
  services:
    VllmWorker:
      volumeMounts:
        - name: model-cache
          mountPoint: /home/dynamo/.cache/huggingface
```

挂载了 `model-cache` 的所有 `VllmWorker` pod 现在会从共享缓存读取，避免每个 pod 重复下载。如希望前端（frontend）也复用 tokenizer 与 config 文件，将相同 PVC 挂载到那里即可。

### 编译缓存

对于 vLLM，你也可以用第二个 PVC 缓存编译产物（CUDA graphs 等）：

```yaml
spec:
  pvcs:
    - create: false
      name: model-cache
    - create: false
      name: compilation-cache
  services:
    VllmWorker:
      volumeMounts:
        - name: model-cache
          mountPoint: /home/dynamo/.cache/huggingface
        - name: compilation-cache
          mountPoint: /home/dynamo/.cache/vllm
```

## 选项 2：Model Express（P2P 分发）

[Model Express](https://github.com/ai-dynamo/modelexpress) 是一个 P2P 模型分发服务器，将模型下载一次，然后通过网络服务给所有 pod。它通过自定义 load format 直接集成到 vLLM 的权重加载流程中。

### 工作原理

1. 集群中运行一个 Model Express 服务器，缓存模型权重
2. Worker 使用 `--load-format=mx-source` 或 `--load-format=mx-target` 从该服务器加载
3. K8s operator 自动将 `MODEL_EXPRESS_URL` 注入到所有 pod

### 安装

**与 Dynamo Platform 一同安装：**

```bash
helm install dynamo-platform dynamo-platform-${RELEASE_VERSION}.tgz \
  --namespace ${NAMESPACE} \
  --set "dynamo-operator.modelExpressURL=http://model-express-server.model-express.svc.cluster.local:8080"
```

**配置 worker 使用 Model Express：**

```yaml
services:
  VllmWorker:
    envs:
      - name: VLLM_LOAD_FORMAT
        value: mx-target
```

当 operator 中配置了 `MODEL_EXPRESS_URL` 时，它会作为环境变量自动注入到所有组件 pod。使用 `mx-source` 或 `mx-target` 加载格式的 worker 会连接到该服务器进行模型权重分发。

### 何时使用 Model Express

| 场景 | 推荐方式 |
|----------|---------------------|
| 小型集群，简单部署 | PVC + 下载 Job |
| 大型集群，节点众多 | Model Express |
| 模型已在共享存储（NFS）上 | PVC |
| 模型在集群中频繁更新 | Model Express |

## 另请参阅

- [使用 DynamoModel 管理模型](deployment/dynamomodel-guide.md) —— 声明式模型管理 CRD
- [详细安装指南](installation-guide.md) —— 包含 Model Express 的 Helm chart 配置
- [LoRA 适配器](../features/lora/README.md) —— 动态适配器加载（与基础模型缓存独立）
