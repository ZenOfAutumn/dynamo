# DeepSeek-R1 Recipes

面向生产环境的 **DeepSeek-R1**（671B MoE）部署方案，覆盖多种后端（backend）与硬件配置。

## 可选配置

| 配置 | GPU | 后端 | 模式 | 描述 |
|--------------|------|---------|------|-------------|
| [**sglang/disagg-8gpu**](sglang/disagg-8gpu/) | 16x H200 | SGLang | 解耦（disaggregated） WideEP | 每个 worker TP=8，单节点 |
| [**sglang/disagg-16gpu**](sglang/disagg-16gpu/) | 32x H200 | SGLang | 解耦 WideEP | 每个 worker TP=16，多节点 |
| [**trtllm/disagg/wide_ep/gb200**](trtllm/disagg/wide_ep/gb200/) | 36x GB200 | TensorRT-LLM | 解耦 WideEP | 8 个解码（decode）节点 + 1 个预填充（prefill）节点 |
| [**vllm/disagg**](vllm/disagg/) | 32x H200 | vLLM | 解耦 DEP16 | 多节点，data-expert 并行 |

## 前置条件

1. **已安装 Dynamo Platform** —— 参见 [Kubernetes 部署指南](../../docs/kubernetes/README.md)
2. **GPU 集群**，使用 H200 或 GB200 GPU，且满足对应配置的硬件要求
3. **HuggingFace token**，需要有访问 DeepSeek 模型的权限
4. **高带宽网络** —— 多节点部署推荐 InfiniBand 或 RoCE

## 快速开始

```bash
# Set namespace
export NAMESPACE=dynamo-demo
kubectl create namespace ${NAMESPACE}

# Create HuggingFace token secret
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token-here" \
  -n ${NAMESPACE}

# Download model (update storageClassName in model-cache.yaml first!)
# For SGLang deployments:
kubectl apply -f model-cache/model-cache.yaml -n ${NAMESPACE}
kubectl apply -f model-cache/model-download-sglang.yaml -n ${NAMESPACE}

# For vLLM/TRT-LLM deployments:
kubectl apply -f model-cache/model-cache.yaml -n ${NAMESPACE}
kubectl apply -f model-cache/model-download.yaml -n ${NAMESPACE}

# Wait for download (this is a large model - may take 1+ hours)
# For SGLang: kubectl wait --for=condition=Complete job/model-download-sglang ...
# For vLLM/TRT-LLM: kubectl wait --for=condition=Complete job/model-download ...
kubectl wait --for=condition=Complete job/model-download-sglang -n ${NAMESPACE} --timeout=7200s

# Deploy (choose one configuration)
kubectl apply -f sglang/disagg-8gpu/deploy.yaml -n ${NAMESPACE}
```

## 测试部署

```bash
# Port-forward the frontend (service name varies by deployment)
kubectl port-forward svc/sgl-dsr1-8gpu-frontend 8000:8000 -n ${NAMESPACE}

# Send a test request
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-ai/DeepSeek-R1",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 100
  }'
```

## 模型详情

- **模型**：`deepseek-ai/DeepSeek-R1`
- **架构**：671B 参数 Mixture-of-Experts（MoE）
- **每个 token 的活跃参数量**：约 37B
- **推荐**：生产部署建议使用 FP8 量化

## 硬件需求

DeepSeek-R1 是一个非常大的模型，需要可观的 GPU 显存：

| 配置 | 最低 GPU 显存 | 推荐 |
|--------------|----------------|-------------|
| 16x H200（SGLang TP=8） | 总计 1.1TB | H200 SXM（每张 141GB） |
| 32x H200（SGLang TP=16，vLLM） | 总计 2.2TB | H200 SXM（每张 141GB） |
| 36x GB200（TRT-LLM） | 约 2.5TB | GB200 NVL72 |

## 注意事项

- **模型下载耗时**：DeepSeek-R1 约 1.3TB；预计需要 1-2 小时
- **NCCL 错误**：通常意味着显存不足（OOM）。可以减小 worker 参数中的 `--mem-fraction-static`
- **多节点**：需要启用 InfiniBand/IBGDA。参见 [vLLM EP 文档](https://docs.vllm.ai/en/latest/serving/expert_parallel_deployment/)
- **存储类（storage class）**：部署前请先更新 `model-cache/model-cache.yaml` 中的 `storageClassName`

## 各后端注意事项

### SGLang
- 使用 WideEP（Wide Expert Parallel）实现高效 MoE 推理（inference）
- SGLang 的具体配置见 [sglang/README.md](sglang/README.md)

### TensorRT-LLM
- 需要 FP4 量化的 checkpoint
- 包含 GB200 专属优化

### vLLM
- 使用 DEP（Data-Expert Parallel）以及混合负载均衡
- 详细搭建步骤见 [vllm/disagg/README.md](vllm/disagg/README.md)
