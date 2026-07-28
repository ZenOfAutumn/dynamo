---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 通过 Fluid 进行模型缓存
---

Fluid 是面向 Kubernetes 的开源云原生数据编排与加速平台。它将来自各种来源（对象存储、分布式文件系统、云存储）的数据访问进行虚拟化和加速，非常适合 AI、机器学习与大数据工作负载。

## 关键特性

- **数据缓存与加速：** 在靠近计算工作负载处缓存远端数据，以更快访问。
- **统一数据访问：** 通过单一接口访问 S3、HDFS、NFS 等数据。
- **Kubernetes 原生：** 通过 CRD 与 Kubernetes 集成进行数据管理。
- **可扩展性：** 支持大规模数据与计算集群。

## 安装

可在任意 Kubernetes 集群上通过 Helm 安装 Fluid。

**前置条件：**
- Kubernetes >= 1.18
- `kubectl` >= 1.18
- `Helm` >= 3.5

**快速安装：**
```sh
kubectl create ns fluid-system
helm repo add fluid https://fluid-cloudnative.github.io/charts
helm repo update
helm install fluid fluid/fluid -n fluid-system
```
高级配置参见 [Fluid 安装指南](https://fluid-cloudnative.github.io/docs/get-started/installation)。

## 部署前步骤

1. 安装 Fluid（参见 [安装](#安装)）。
2. 创建 Dataset 与 Runtime（参见 [下方示例](#webufs-示例)）。
3. 在你的工作负载中挂载得到的 PVC。


## 挂载数据源

### WebUFS 示例

WebUFS 允许将 HTTP/HTTPS 来源挂载为文件系统。

```yaml
# 将一个公开的 HTTP 目录挂载为 Fluid Dataset
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: webufs-model
spec:
  mounts:
    - mountPoint: https://myhost.org/path_to_my_model  # 替换为你的 HTTP 来源
      name: webufs-model
---
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: webufs-model
spec:
  replicas: 2
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 2Gi
        high: "0.95"
        low: "0.7"
```
应用后，Fluid 会创建一个名为 `webufs-model` 的 PersistentVolumeClaim（PVC），其中包含相应文件。

### S3 示例

将一个 S3 桶挂载为 Fluid Dataset。

```yaml
# 将 S3 桶挂载为 Fluid Dataset
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: s3-model
spec:
  mounts:
    - mountPoint: s3://<your-bucket>  # 替换为你的桶名
      options:
        alluxio.underfs.s3.endpoint: http://minio:9000  # S3 端点（如 MinIO）
        alluxio.underfs.s3.disable.dns.buckets: "true"
        aws.secretKey: "<your-secret>"
        aws.accessKeyId: "<your-access-key>"
---
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: s3-model
spec:
  replicas: 1
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 1Gi
        high: "0.95"
        low: "0.7"
---
apiVersion: data.fluid.io/v1alpha1
kind: DataLoad
metadata:
  name: s3-model-loader
spec:
  dataset:
    name: s3-model
    namespace: <your-namespace>  # 替换为你的命名空间
  loadMetadata: true
  target:
    - path: "/"
      replicas: 1
```

得到的 PVC 名为 `s3-model`。

## 在 Fluid 中使用 HuggingFace 模型

**限制：**
- HuggingFace 模型并不以简单的文件系统或桶形式暴露。
- Fluid 与 HuggingFace Hub API 之间没有原生集成。

**变通方案：下载并上传到 S3/MinIO**

1. 使用 HuggingFace CLI 或 SDK 下载模型。
2. 将模型文件上传到受支持的存储后端（S3、GCS、NFS）。
3. 用 Fluid 挂载该后端。

**用于下载并上传的示例 Pod：**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: download-hf-to-minio
spec:
  restartPolicy: Never
  containers:
    - name: downloader
      image: python:3.10-slim
      command: ["sh", "-c"]
      args:
        - |
          set -eux
          pip install --no-cache-dir huggingface_hub awscli
          BUCKET_NAME=hf-models
          ENDPOINT_URL=http://minio:9000
          MODEL_NAME=deepseek-ai/DeepSeek-R1-Distill-Llama-8B
          LOCAL_DIR=/tmp/model
          if ! aws --endpoint-url $ENDPOINT_URL s3 ls "s3://$BUCKET_NAME" > /dev/null 2>&1; then
            aws --endpoint-url $ENDPOINT_URL s3 mb "s3://$BUCKET_NAME"
          fi
          huggingface-cli download $MODEL_NAME --local-dir $LOCAL_DIR --local-dir-use-symlinks False
          aws --endpoint-url $ENDPOINT_URL s3 cp $LOCAL_DIR s3://$BUCKET_NAME/$MODEL_NAME --recursive
      env:
        - name: AWS_ACCESS_KEY_ID
          value: "<your-access-key>"
        - name: AWS_SECRET_ACCESS_KEY
          value: "<your-secret>"
      volumeMounts:
        - name: tmp-volume
          mountPath: /tmp/model
  volumes:
    - name: tmp-volume
      emptyDir: {}
```

随后即可使用 `s3://hf-models/deepseek-ai/DeepSeek-R1-Distill-Llama-8B` 作为 Dataset 的挂载点。

## 与 Dynamo 配合使用

在你的 DynamoGraphDeployment 中挂载 Fluid 生成的 PVC：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: model-caching
spec:
  pvcs:
    - name: s3-model
  envs:
    - name: HF_HOME
      value: /model
    - name: DYN_DEPLOYMENT_CONFIG
      value: '{"Common": {"model": "/model", ...}}'
  services:
    VllmWorker:
      volumeMounts:
        - name: s3-model
          mountPoint: /model
    Processor:
      volumeMounts:
        - name: s3-model
          mountPoint: /model
```


## 完整示例：llama3.3 70B

### 性能

部署 LLaMA 3.3 70B 时，使用 Fluid 作为缓存层，我们观察到将单节点缓存配置为本地完整持有 100% 的模型文件能获得最佳性能。通过确保 vllm worker pod 调度到与 Fluid 缓存相同的节点上，我们消除了网络 I/O 瓶颈，从而在测试中获得了最快的模型启动时间和最高的推理效率。

| 缓存配置                                     | vLLM Pod 放置                  | 启动时间        |
|----------------------------------------------|----------------------------------|-----------------|
| ❌ 无缓存（从 HuggingFace 下载）             | 不适用                           | ~9 分钟         |
| 🟡 多节点缓存（100% 模型已缓存）             | 不在缓存节点                     | ~18 分钟        |
| 🟡 多节点缓存（100% 模型已缓存）             | 在缓存节点                       | ~10 分钟        |
| ✅ 单节点缓存（100% 模型已缓存）             | 在缓存节点                       | ~80 秒          |


### 资源

```yaml
# dataset.yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: llama-3-3-70b-instruct-model
  namespace: my-namespace
spec:
  mounts:
    - mountPoint: s3://hf-models/meta-llama/Llama-3.3-70B-Instruct
      options:
        alluxio.underfs.s3.endpoint: http://minio:9000
        alluxio.underfs.s3.disable.dns.buckets: "true"
        aws.secretKey: "minioadmin"
        aws.accessKeyId: "minioadmin"
        alluxio.underfs.s3.streaming.upload.enabled: "true"
        alluxio.underfs.s3.multipart.upload.threads: "20"
        alluxio.underfs.s3.socket.timeout: "50s"
        alluxio.underfs.s3.request.timeout: "60s"
---
# runtime.yaml
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: llama-3-3-70b-instruct-model
  namespace: my-namespace
spec:
  replicas: 1
  properties:
    alluxio.user.file.readtype.default: CACHE_PROMOTE
    alluxio.user.file.write.type.default: CACHE_THROUGH
    alluxio.user.block.size.bytes.default: 128MB
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 300Gi
        high: "1.0"
        low: "0.7"
---
# DataLoad - 将模型预加载到缓存
apiVersion: data.fluid.io/v1alpha1
kind: DataLoad
metadata:
  name: llama-3-3-70b-instruct-model-loader
spec:
  dataset:
    name: llama-3-3-70b-instruct-model
    namespace: my-namespace
  loadMetadata: true
  target:
    - path: "/"
      replicas: 1
```

以及关联的 DynamoGraphDeployment，使用 pod affinity 将 vllm worker 调度到与 Alluxio 缓存 worker 相同的节点

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-hello-world
spec:
  envs:
  - name: DYN_LOG
    value: "debug"
  - name: DYN_DEPLOYMENT_CONFIG
    value: '{"Common": {"model": "/model", "block-size": 64, "max-model-len": 16384},
      "Frontend": {"served_model_name": "meta-llama/Llama-3.3-70B-Instruct", "endpoint":
      "dynamo.Processor.chat/completions", "port": 8000}, "Processor": {"router":
      "round-robin", "router-num-threads": 4, "common-configs": ["model", "block-size",
      "max-model-len"]}, "VllmWorker": {"tensor-parallel-size": 4, "enforce-eager": true, "max-num-batched-tokens":
      16384, "enable-prefix-caching": true, "ServiceArgs": {"workers": 1, "resources":
      {"gpu": "4", "memory": "40Gi"}}, "common-configs": ["model", "block-size", "max-model-len"]},
      "Planner": {"environment": "kubernetes", "no-operation": true}}'
  pvcs:
    - name: llama-3-3-70b-instruct-model
  services:
    Processor:
      volumeMounts:
        - name: llama-3-3-70b-instruct-model
          mountPoint: /model
    VllmWorker:
      volumeMounts:
        - name: llama-3-3-70b-instruct-model
          mountPoint: /model
      extraPodSpec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
                - matchExpressions:
                  - key: fluid.io/s-alluxio-my-namespace-llama-3-3-70b-instruct-model
                    operator: In
                    values:
                      - "true"
```


## 故障排查与常见问题

- **PVC 未创建？** 查看 Fluid 与 AlluxioRuntime 的 pod 日志。
- **找不到模型？** 确认模型已上传到正确的桶/路径。
- **权限错误？** 检查 S3/MinIO 凭据与桶策略。

## 资源

- [Fluid 文档](https://fluid-cloudnative.github.io/)
- [Alluxio 文档](https://docs.alluxio.io/)
- [MinIO 文档](https://docs.min.io/)
- [Hugging Face Hub](https://huggingface.co/docs/hub/index)
- [Dynamo README](https://github.com/ai-dynamo/dynamo/blob/main/.devcontainer/README.md)
- [Dynamo 文档](https://docs.nvidia.com/dynamo/)
