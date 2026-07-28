# Kimi-K2.5 nvidia/Kimi-K2.5-NVFP4 — Kubernetes 上的聚合（Aggregated）部署

> **仅文本：** 当前上游 TensorRT-LLM 仅通过加载 DeepSeek-V3 文本骨干（`text_config`）来支持 Kimi-K2.5 模型。视觉编码器不会被加载，因此不会处理图像输入。完整多模态支持需要 TRT-LLM 上游对 Kimi K2.5 的原生支持。

本目录包含针对 `nvidia/Kimi-K2.5-NVFP4` 模型的两套聚合部署配置。

| 部署 | 清单 | 说明 | 硬件需求
|-----------|----------|-------------|----|
| **标准聚合（Standard Aggregated）** | [`deploy.yaml`](deploy.yaml) | 启用 KV 感知路由（KV-aware routing）的基础聚合服务 | 1x8 B200 节点 |
| **聚合 + EAGLE 推测解码** | [`deploy-specdec.yaml`](deploy-specdec.yaml) | 高性能聚合部署，启用 EAGLE 推测解码（speculative decoding）与 KV 感知路由 | 8x4 GB200 节点 |

## 前置条件

- 已安装 [Dynamo Operator](https://docs.nvidia.com/dynamo/) 的 Kubernetes 集群
- 1x8 B200 GPU 或 8x4 GB200 GPU
- 包含你的 Hugging Face token 的 `hf-token-secret` Secret
- 一个已经存在的 `model-cache` PVC
- `deploy-specdec.yaml` 使用 `nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:my-tag`，并需配合最新 top-of-tree 的 Dynamo TRT-LLM 镜像使用

---

## 标准聚合部署

使用 [`deploy.yaml`](deploy.yaml)。这是较简单的配置 —— 启用 KV 感知路由的聚合服务，不启用 CPU offload 的 KV 缓存。

```bash
kubectl apply -f deploy.yaml -n ${NAMESPACE}
```

它将创建：
- 一个 **ConfigMap**（`llm-config`），包含 TRT-LLM 引擎参数（TP=8、EP=8、FP8 KV-cache）。
- 一个 **DynamoGraphDeployment**（`kimi-k25-agg`），包含一个 Frontend（KV-router 模式）与一个为 `nvidia/Kimi-K2.5-NVFP4` 服务的 TrtllmWorker。

---

## 聚合 + EAGLE 推测解码 + KV 感知路由部署

使用 [`deploy-specdec.yaml`](deploy-specdec.yaml)。该高性能配置在 GB200 上运行带 EAGLE 推测解码的 KV 感知聚合服务。

### 推测解码前置条件

- 8 个 GB200 节点，每节点 4 个 GPU
- 在部署前，将 [`deploy-specdec.yaml`](deploy-specdec.yaml) 中的占位镜像标签 `nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:my-tag` 更新为正确的镜像。

### 额外的模型资产

该部署需要在 `model-cache` PVC 上同时准备 Kimi 基础权重以及 Eagle 草稿模型（draft model）。

下载基础模型：

```bash
kubectl apply -f ../../../model-cache/model-cache.yaml -n ${NAMESPACE}
kubectl apply -f ../../../model-cache/nvidia/model-download.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=6000s
```

下载 Eagle 草稿模型：

```bash
kubectl apply -f ../../../model-cache/nvidia/eagle-download.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/eagle-download -n ${NAMESPACE} --timeout=6000s
```

worker 配置从如下路径加载草稿模型：

```yaml
speculative_config:
  decoding_type: Eagle
  max_draft_len: 3
  speculative_model_dir: /opt/models/hub/models--nvidia--Kimi-K2.5-Thinking-Eagle3/snapshots/0b0c6ac039089ad2c2418c91c039553381a302d9
```

### 推测解码部署拓扑

该清单运行一个聚合的 frontend 与四个聚合的 worker 副本。每个 worker 跨两个节点：

- `multinode.nodeCount: 2`
- 每节点 `resources.limits.gpu: "4"`
- `tensor_parallel_size: 8`
- `moe_expert_parallel_size: 8`

worker 部分总计为 8 节点部署。

### 部署

```bash
kubectl apply -f deploy-specdec.yaml -n ${NAMESPACE}
```

它将创建：
- 一个 **ConfigMap**（`llm-config-specdec`），包含 TRT-LLM 推测解码配置
- 一个 **DynamoGraphDeployment**（`kimi-k25-agg-specdec`），包含一个 KV 感知路由 frontend 以及四个为 `nvidia/Kimi-K2.5-NVFP4` 服务的多节点 TRT-LLM worker 副本
