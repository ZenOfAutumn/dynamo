<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Dynamo 生产级 Recipes

经过生产环境验证的 Kubernetes 部署 recipe，使用 NVIDIA Dynamo 进行 LLM 推理（inference）。

> **先决条件：** 本指南假设你已经安装了 Dynamo Kubernetes Platform。
> 若未安装，请先按照 **[Kubernetes 部署指南](../docs/kubernetes/README.md)** 操作。

## 可用 Recipe

### 特性对比 Recipe

这些 recipe 通过基准测试结果对比 Dynamo 的性能特性，每个都包含基线与优化后的部署配置：

| 模型 | 框架 | 配置 | GPU | 特性 |
|-------|-----------|---------------|------|----------|
| **[Qwen3-32B](qwen3-32b/)** | vLLM | Disagg + KV-Router | 16x H200 | **解耦（Disaggregated）服务 + KV 感知路由** —— 使用真实 Mooncake trace 进行基准对比 |
| **[DeepSeek-V3.2-NVFP4](deepseek-v32-fp4/)** | TensorRT-LLM | Agg + Disagg WideEP | 32x GB200 | **解耦服务 + KV 感知路由** —— 基于 Mooncake 的合成编程 trace 基准对比 |
| **[Qwen3-VL-30B-A3B-FP8](qwen3-vl-30b/)** | vLLM | Agg + Embedding Cache | 1x GB200 | **多模态 Embedding Cache** —— 基准对比显示吞吐 +16%、TTFT -28% |

### 聚合（Aggregated）与解耦 Recipe

下列 recipe 演示聚合或解耦服务：

**GAIE 列**：表示该 recipe 是否包含与 [Gateway API Inference Extension（GAIE）](../deploy/inference-gateway/README.md)的集成 —— 这是一个 Kubernetes SIG 项目，扩展 Gateway API 以支持 AI 推理工作负载，提供负载均衡、模型路由与请求管理。

| 模型 | 框架 | 模式 | GPU | 部署 | 基准 | 备注 | GAIE |
|-------|-----------|------|------|------------|-----------|-------|------|
| **[Llama-3-70B](llama-3-70b/vllm/agg/)** | vLLM | 聚合 | 4x H100/H200 | ✅ | ✅ | FP8 动态量化 | ✅ |
| **[Llama-3-70B](llama-3-70b/vllm/disagg-single-node/)** | vLLM | 解耦（单节点） | 8x H100/H200 | ✅ | ✅ | Prefill + Decode 分离 | ❌ |
| **[Llama-3-70B](llama-3-70b/vllm/disagg-multi-node/)** | vLLM | 解耦（多节点） | 16x H100/H200 | ✅ | ✅ | 2 节点，每节点 8 GPU | ❌ |
| **[Qwen3-32B-FP8](qwen3-32b-fp8/trtllm/agg/)** | TensorRT-LLM | 聚合 | 2x H100/H200/A100 | ✅ | ✅ | FP8 量化 | ❌ |
| **[Qwen3-32B-FP8](qwen3-32b-fp8/trtllm/disagg/)** | TensorRT-LLM | 解耦 | 8x H100/H200/A100 | ✅ | ✅ | Prefill + Decode 分离 | ❌ |
| **[Qwen3-32B-FP8](qwen3-32b-fp8/vllm/disagg/)** | vLLM | 解耦（单节点） | 8x A100 | ✅ | ✅ | 2× TP2 prefill + 1× TP4 decode，NixlConnector KV 传输 | ❌ |
| **[Qwen3-235B-A22B-FP8](qwen3-235b-a22b-fp8/trtllm/agg/hopper/)** | TensorRT-LLM | 聚合（Hopper） | 16x H100/H200 | ✅ | ✅ | MoE 模型，TP4×EP4 | ❌ |
| **[Qwen3-235B-A22B-FP8](qwen3-235b-a22b-fp8/trtllm/agg/blackwell/)** | TensorRT-LLM | 聚合（Blackwell） | 16x B100/B200 | ✅ | ✅ | MoE 模型，TP4×EP4，DEEPGEMM 后端 | ❌ |
| **[Qwen3-235B-A22B-FP8](qwen3-235b-a22b-fp8/trtllm/disagg/hopper/)** | TensorRT-LLM | 解耦（Hopper） | 16x H100/H200 | ✅ | ✅ | MoE 模型，Prefill + Decode | ❌ |
| **[Qwen3-235B-A22B-FP8](qwen3-235b-a22b-fp8/trtllm/disagg/blackwell/)** | TensorRT-LLM | 解耦（Blackwell） | 16x B100/B200 | ✅ | ✅ | MoE 模型，Prefill + Decode，DEEPGEMM 后端 | ❌ |
| **[GPT-OSS-120B](gpt-oss-120b/trtllm/agg/)** | TensorRT-LLM | 聚合 | 4x GB200 | ✅ | ✅ | 仅 Blackwell，WideEP | ❌ |
| **[GPT-OSS-120B](gpt-oss-120b/trtllm/disagg/)** | TensorRT-LLM | 解耦 | 5x Blackwell（GB200/B200） | ✅ | ✅ | Prefill/Decode 分离 | ❌ |
| **[DeepSeek-R1](deepseek-r1/sglang/disagg-8gpu/)** | SGLang | Disagg WideEP | 16x H200 | ✅ | ❌ | TP=8，单节点。使用 `model-download-sglang.yaml` | ❌ |
| **[DeepSeek-R1](deepseek-r1/sglang/disagg-16gpu/)** | SGLang | Disagg WideEP | 32x H200 | ✅ | ❌ | TP=16，多节点。使用 `model-download-sglang.yaml` | ❌ |
| **[DeepSeek-R1](deepseek-r1/trtllm/disagg/wide_ep/gb200/)** | TensorRT-LLM | Disagg WideEP（GB200） | 36x GB200 | ✅ | ✅ | 多节点：8 个 decode + 1 个 prefill 节点 | ❌ |
| **[DeepSeek-R1](deepseek-r1/)** | vLLM | Disagg DEP16 | 32x H200 | ✅ | ❌ | 多节点，data-expert parallel | ❌ |

**图例：**
- **部署**：✅ = 提供完整 `deploy.yaml` 清单
- **基准**：✅ = 包含 `perf.yaml`，可运行 AIPerf 基准测试

### 功能性 Recipe（暂无基准）

下列 recipe 演示带 Dynamo 特性的可工作部署，但尚未做性能调优或配套基准清单。

| 模型 | 框架 | 模式 | GPU | 部署 | 备注 |
|-------|-----------|-------|------|------------|-------|
| **[Nemotron-3-Super-FP8](nemotron-3-super-fp8/vllm/agg/)** | vLLM | 聚合 | 4x H100/H200 | ✅ | TP=4，KV 感知路由 |
| **[Nemotron-3-Super-FP8](nemotron-3-super-fp8/sglang/agg/)** | SGLang | 聚合 | 4x H100/H200 | ✅ | TP=4，KV 感知路由，1.0+ |
| **[Nemotron-3-Super-FP8](nemotron-3-super-fp8/trtllm/disagg/)** | TensorRT-LLM | 解耦 | 4x H100/H200 | ✅ | TP=2 prefill/decode 分离，UCX KV 传输 |
| **[Nemotron-3-Super-FP8](nemotron-3-super-fp8/sglang/disagg/)** | SGLang | 解耦 | 4x H100/H200 | ✅ | TP=2 prefill/decode 分离，nixl KV 传输，1.0+ |
| **[Kimi-K2.5（Baseten）](kimi-k2.5/trtllm/agg/baseten/)** | TensorRT-LLM | 聚合 | 8x B200 | ✅ | 仅文本 —— MoE 模型，TP8×EP8，reasoning + tool calling |

### 实验性 Recipe

下列 recipe 处于积极开发中，可能需要额外步骤（如自定义容器构建）。它们功能上可工作，但尚未完全验证用于生产。

| 模型 | 框架 | 模式 | GPU | 部署 | 备注 |
|-------|-----------|------|------|------------|-------|
| **[GLM-5-NVFP4](glm-5-nvfp4/sglang/disagg/)** | SGLang | Disagg Prefill/Decode | 20x GB200 | ✅ | NVFP4，EAGLE 推测解码，TP16 decode + TP4 prefill。需要[自定义容器构建](glm-5-nvfp4/)。 |
| **[Nemotron-3-Nano-Omni-NVFP4](nemotron-3-nano-omni/vllm/agg/)** | vLLM | 聚合 | 1x GPU | ✅ | 多模态 文本/图像/视频/音频 服务。需要[自定义容器构建](nemotron-3-nano-omni/)。 |
| **[nvidia/Kimi-K2.5-NVFP4](kimi-k2.5/trtllm/agg/nvidia/)** | TensorRT-LLM | 聚合 | 8x B200 | ✅ | 仅文本 —— MoE 模型，TP8×EP8，reasoning + tool calling。视觉输入尚未可用。 |
| **[nvidia/Kimi-K2.5-NVFP4](kimi-k2.5/tokenspeed/agg/nvidia/)** | TokenSpeed | 聚合 | 4x B200 | ✅ | 仅文本 —— MoE 模型，TP4×EP4，reasoning + tool calling。需要[自定义容器构建](kimi-k2.5/tokenspeed/agg/nvidia/Dockerfile)（暂无公开 Dynamo+TokenSpeed 镜像），且使用原生 `Deployment`/`Service` 而非 `DynamoGraphDeployment`（operator 后端支持待补）。 |
| **[DeepSeek-V4-Flash](deepseek-v4/deepseek-v4-flash/vllm/agg_b200/)** | vLLM | 聚合 | 4x B200 | ✅ | 仅文本 —— MoE 模型（284B / 13B 活跃），DP=4 + EP，FP8 KV 缓存，reasoning + tool calling。需要[自定义容器构建](deepseek-v4/container/)。 |
| **[DeepSeek-V4-Flash](deepseek-v4/deepseek-v4-flash/vllm/agg_gb200/)** | vLLM | 聚合 | 4x GB200 | ✅ | 仅文本 —— MoE 模型（284B / 13B 活跃），TP=4 + EP，`deep_gemm_mega_moe`，FP8 KV 缓存，reasoning + tool calling（单 NVL4 tray）。需要[自定义容器构建](deepseek-v4/container/)。 |
| **[DeepSeek-V4-Flash](deepseek-v4/deepseek-v4-flash/sglang/agg/)** | SGLang | 聚合 | 4x B200 | ✅ | 仅文本 —— MoE 模型（284B / 13B 活跃），TP=4，通过 FlashInfer 的 MXFP4 MoE，EAGLE MTP（3 步 / 4 个 draft token），reasoning + tool calling。提供预构建镜像；可选[自定义容器构建](deepseek-v4/container/)。 |
| **[DeepSeek-V4-Pro](deepseek-v4/deepseek-v4-pro/vllm/agg/b200/)** | vLLM | 聚合 | 8x B200 | ✅ | 仅文本 —— MoE 模型（1.6T / 49B 活跃，1M 上下文），TP=8 + EP，FP4+FP8 混合 checkpoint，FP8 KV 缓存，CSA+HCA attention，tool calling。Day-0 时 thinking 模式不稳定 —— 请以 `thinking: false` 运行。需要[自定义容器构建](deepseek-v4/container/)。 |
| **[DeepSeek-V4-Pro](deepseek-v4/deepseek-v4-pro/vllm/agg/gb200/)** | vLLM | 聚合 | 8x GB200（2 个 NVL4 tray） | ✅ | 仅文本 —— 同 B200 agg；TP=8 + EP，跨节点通过 NVLink72（MNNVL）+ ComputeDomain。需要[自定义容器构建](deepseek-v4/container/)。 |
| **[DeepSeek-V4-Pro](deepseek-v4/deepseek-v4-pro/vllm/disagg/gb200/)** | vLLM | 解耦 | 16x GB200（4 个 NVL4 tray） | ✅ | 仅文本 —— 每个 worker DP=8 + EP，1P + 1D，NVLink72（MNNVL）+ ComputeDomain。需要[自定义容器构建](deepseek-v4/container/)。 |
| **[DeepSeek-V4-Pro](deepseek-v4/deepseek-v4-pro/sglang/agg/)** | SGLang | 聚合 | 8x B200 | ✅ | 仅文本 —— MoE 模型（1.6T / 49B 活跃，1M 上下文），TP=8，通过 FlashInfer 的 MXFP4 MoE，EAGLE MTP（3 步 / 4 个 draft token），reasoning + tool calling。提供预构建镜像（与 [DeepSeek-V4-Flash](deepseek-v4/deepseek-v4-flash/sglang/agg/) 共用）。 |

## Recipe 结构

每个完整 recipe 都遵循以下标准结构：

```
<model-name>/
├── README.md (optional)           # Model-specific deployment notes
├── model-cache/
│   ├── model-cache.yaml          # PersistentVolumeClaim for model storage
│   └── model-download.yaml       # Job to download model from HuggingFace
└── <framework>/                  # vllm, sglang, or trtllm
    └── <deployment-mode>/        # agg, disagg, disagg-single-node, etc.
        ├── deploy.yaml           # Complete DynamoGraphDeployment manifest
        └── perf.yaml (optional)  # AIPerf benchmark job
```

## 快速开始

### 先决条件

**1. 已安装 Dynamo 平台**

这些 recipe 需要先安装 Dynamo Kubernetes Platform。安装指南：

- **[Kubernetes 部署指南](../docs/kubernetes/README.md)** - 快速上手（约 10 分钟）
- **[详细安装指南](../docs/kubernetes/installation-guide.md)** - 高级选项

**2. GPU 集群要求**

请确保你的集群（cluster）：
- GPU 节点（node）满足 recipe 要求（见上方表格）
- 已安装 GPU operator
- 合适的 GPU 驱动与容器运行时（runtime）

**3. HuggingFace 访问**

配置认证以下载模型：

```bash
export NAMESPACE=your-namespace
kubectl create namespace ${NAMESPACE}

# Create HuggingFace token secret
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token-here" \
  -n ${NAMESPACE}
```

**4. 存储配置**

将 `<model>/model-cache/model-cache.yaml` 中的 `storageClassName` 修改为与你的集群匹配的值：

```bash
# Find your storage class name
kubectl get storageclass

# Edit the model-cache.yaml file and update:
# spec:
#   storageClassName: "your-actual-storage-class"
```

### 部署一个 Recipe

**步骤 1：下载模型**

```bash
cd recipes
# Update storageClassName in model-cache.yaml first!
kubectl apply -f <model>/model-cache/ -n ${NAMESPACE}

# Wait for download to complete (may take 10-60 minutes depending on model size)
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=6000s

# Monitor progress
kubectl logs -f job/model-download -n ${NAMESPACE}
```

**步骤 2：部署服务**

更新 `<model>/<framework>/<mode>/deploy.yaml` 中的镜像。

```bash
kubectl apply -f <model>/<framework>/<mode>/deploy.yaml -n ${NAMESPACE}

# Check deployment status
kubectl get dynamographdeployment -n ${NAMESPACE}

# Check pod status
kubectl get pods -n ${NAMESPACE}

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l nvidia.com/dynamo-graph-deployment-name=<deployment-name> -n ${NAMESPACE} --timeout=600s
```

**步骤 3：测试部署**

```bash
# Port forward to access the service locally
kubectl port-forward svc/<deployment-name>-frontend 8000:8000 -n ${NAMESPACE}

# In another terminal, test the endpoint
curl http://localhost:8000/v1/models

# Send a test request
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "<model-name>",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'
```

**步骤 4：运行基准测试（可选）**

```bash
# Only if perf.yaml exists in the recipe directory
kubectl apply -f <model>/<framework>/<mode>/perf.yaml -n ${NAMESPACE}

# Monitor benchmark progress
kubectl logs -f job/<benchmark-job-name> -n ${NAMESPACE}

# View results after completion
kubectl logs job/<benchmark-job-name> -n ${NAMESPACE} | tail -50
```


## 部署示例

### Llama-3-70B + vLLM（聚合）

```bash
export NAMESPACE=dynamo-demo
kubectl create namespace ${NAMESPACE}

# Create HF token secret
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token" \
  -n ${NAMESPACE}

# Deploy
cd recipes
kubectl apply -f llama-3-70b/model-cache/ -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=6000s
kubectl apply -f llama-3-70b/vllm/agg/deploy.yaml -n ${NAMESPACE}

# Test
kubectl port-forward svc/llama3-70b-agg-frontend 8000:8000 -n ${NAMESPACE}
```

### Inference Gateway（GAIE）集成（可选）

针对 Llama-3-70B + vLLM（聚合），提供了一个与 Inference Gateway 集成的示例。

请先按上文部署 Dynamo Graph。

然后按 [Deploy Inference Gateway 第 2 节](../deploy/inference-gateway/README.md#2-deploy-inference-gateway) 安装 GAIE。

更新部署文件中的 containers.epp.image，例如 llama-3-70b/vllm/agg/gaie/k8s-manifests/epp/deployment.yaml。它应与发布 tag 匹配，格式为 `nvcr.io/nvidia/ai-dynamo/frontend:<version>`，例如 `nvcr.io/nvidia/ai-dynamo/frontend:0.9.0`
该 recipe 假设你使用 Kubernetes 服务发现后端，并在 epp 部署中设置 `DYN_DISCOVERY_BACKEND` 环境变量。如果你想用 etcd，请启用下面这几行并移除 DYN_DISCOVERY_BACKEND 环境变量。
```bash
- name: ETCD_ENDPOINTS
  value: "dynamo-platform-etcd.$(PLATFORM_NAMESPACE):2379" #  update dynamo-platform to appropriate namespace
```

```bash
export DEPLOY_PATH=llama-3-70b/vllm/agg/
# DEPLOY_PATH=<model>/<framework>/<mode>/
kubectl apply -R -f "$DEPLOY_PATH/gaie/k8s-manifests" -n "$NAMESPACE"
```

### DeepSeek-R1 在 GB200 上（多节点）

完整的多节点 WideEP 配置参见 [deepseek-r1/trtllm/disagg/wide_ep/gb200/deploy.yaml](deepseek-r1/trtllm/disagg/wide_ep/gb200/deploy.yaml)。

## 自定义

每个 `deploy.yaml` 包含：
- **ConfigMap**：引擎专用配置（嵌入清单中）
- **DynamoGraphDeployment**：Kubernetes 资源定义
- **资源限制**：GPU 数量、内存、CPU requests/limits
- **镜像引用**：带版本 tag 的容器镜像

### 关键自定义点

**模型配置：**
```yaml
# In deploy.yaml under worker args:
args:
  - python3 -m dynamo.vllm --model <your-model-path> --served-model-name <name>
```

**GPU 资源：**
```yaml
resources:
  limits:
    gpu: "4"  # Adjust based on your requirements
  requests:
    gpu: "4"
```

**伸缩：**
```yaml
services:
  VllmDecodeWorker:
    replicas: 2  # Scale to multiple workers
```

**路由模式：**
```yaml
# In Frontend args:
args:
  - python3 -m dynamo.frontend --router-mode kv --http-port 8000
# Options: round-robin, kv (KV-aware routing)
```

**容器镜像：**
```yaml
image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:x.y.z
# Update version tag as needed
```

## 故障排查

### 常见问题

**Pod 卡在 Pending：**
- 检查 GPU 可用性：`kubectl describe node <node-name>`
- 验证 storage class 存在：`kubectl get storageclass`
- 检查资源请求 vs 可用资源

**模型下载失败：**
- 验证 HuggingFace token 是否正确
- 检查集群网络连通性
- 查看 Job 日志：`kubectl logs job/model-download -n ${NAMESPACE}`

**worker 无法启动：**
- 检查 GPU 兼容性（驱动版本、CUDA 版本）
- 如使用私有 registry 请验证 image pull secret
- 查看 Pod 日志：`kubectl logs <pod-name> -n ${NAMESPACE}`

**更多故障排查：**
- [Kubernetes 部署指南](../docs/kubernetes/README.md#troubleshooting)
- [可观测性文档](../docs/kubernetes/observability/)

## 相关文档

- **[Kubernetes 部署指南](../docs/kubernetes/README.md)** - 平台安装与概念
- **[API Reference](../docs/kubernetes/api-reference.md)** - DynamoGraphDeployment CRD 规格
- **[vLLM 后端指南](../docs/backends/vllm/README.md)** - vLLM 专用特性
- **[SGLang 后端指南](../docs/backends/sglang/README.md)** - SGLang 专用特性
- **[TensorRT-LLM 后端指南](../docs/backends/trtllm/README.md)** - TensorRT-LLM 特性
- **[可观测性](../docs/kubernetes/observability/)** - 监控与日志
- **[基准测试指南](../docs/benchmarks/benchmarking.md)** - 性能测试

## 贡献

我们欢迎贡献新的 recipe！参见 [CONTRIBUTING.md](CONTRIBUTING.md)：
- recipe 提交指南
- 必备组件清单
- 测试与验证要求
- 文档规范

### Recipe 质量标准

生产级 recipe 必须包含：
- ✅ 含 DynamoGraphDeployment 的完整 `deploy.yaml`
- ✅ 模型缓存 PVC 与下载 Job
- ✅ 用于性能测试的基准 recipe（`perf.yaml`）
- ✅ 在目标硬件上的验证
- ✅ GPU 要求文档
