# Qwen3-235B-A22B-FP8 部署配方

使用 TensorRT-LLM 的 **Qwen3-235B-A22B**（22B 激活参数的 MoE 模型）生产级部署。

## 可用配置

| 配置 | GPUs | 硬件 | 模式 | 说明 |
|--------------|------|----------|------|-------------|
| [**trtllm/agg/hopper**](trtllm/agg/hopper/) | 16x GPU | H100/H200 | 聚合（Aggregated） | TP4、EP4、KV 感知路由 |
| [**trtllm/agg/blackwell**](trtllm/agg/blackwell/) | 16x GPU | B100/B200 | 聚合（Aggregated） | TP4、EP4、KV 感知路由、DEEPGEMM |
| [**trtllm/disagg/hopper**](trtllm/disagg/hopper/) | 16x GPU | H100/H200 | 解耦（Disaggregated） | 预填充/解码分离 |
| [**trtllm/disagg/blackwell**](trtllm/disagg/blackwell/) | 16x GPU | B100/B200 | 解耦（Disaggregated） | 预填充/解码分离、DEEPGEMM |

## 前置条件

1. **已安装 Dynamo Platform** —— 参见 [Kubernetes 部署指南](../../docs/kubernetes/README.md)
2. **GPU 集群**，配备 H100/H200（Hopper）或 B100/B200（Blackwell）GPU —— 参见 [硬件要求](#hardware-requirements)
3. **HuggingFace token**，可访问 Qwen 模型

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
kubectl apply -f model-cache/ -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=3600s

# Deploy — choose the variant matching your hardware:
kubectl apply -f trtllm/agg/hopper/deploy.yaml -n ${NAMESPACE}       # H100/H200
# OR: kubectl apply -f trtllm/agg/blackwell/deploy.yaml -n ${NAMESPACE}   # B100/B200
# OR: kubectl apply -f trtllm/disagg/hopper/deploy.yaml -n ${NAMESPACE}   # H100/H200
# OR: kubectl apply -f trtllm/disagg/blackwell/deploy.yaml -n ${NAMESPACE} # B100/B200
```

## 测试部署

```bash
# Port-forward the frontend
kubectl port-forward svc/qwen3-235b-a22b-agg-frontend 8000:8000 -n ${NAMESPACE}

# Send a test request
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-235B-A22B-FP8",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'
```

## 模型详情

- **模型**：`Qwen/Qwen3-235B-A22B-FP8`
- **架构**：235B 参数的混合专家（Mixture-of-Experts，MoE）
- **激活参数**：每个 token 约 ~22B
- **后端（Backend）**：TensorRT-LLM（PyTorch 后端）
- **并行**：TP4 × EP4（Expert Parallel）

## 硬件要求

由于在 TRT-LLM 1.3.x 中两种架构需要不同的 MoE 后端配置，因此本配方对 Hopper 与 Blackwell 提供了不同的变体：

- **Hopper（H100/H200，SM90）**：使用默认 MoE 后端 —— `trtllm/{agg,disagg}/hopper/`
- **Blackwell（B100/B200，SM100+）**：要求 `moe_config.backend: DEEPGEMM` —— `trtllm/{agg,disagg}/blackwell/`

差异说明：TRT-LLM 1.3.x 中默认的 CUTLASS MoE 后端在 SM100 上会落入仅适用于 Hopper 的 JIT 路径，导致崩溃。在该版本下，DEEPGEMM 是 Blackwell 上必要的变通方案。而 DEEPGEMM 又会因 scale-factor dtype 不匹配在 Hopper 上崩溃。因此提供两种独立变体。

| 配置 | GPUs | 最低 GPU 显存（合计） |
|--------------|------|----------------------|
| 聚合（Hopper） | 16x H100/H200 | ~1.3TB |
| 聚合（Blackwell） | 16x B100/B200 | ~1.3TB |
| 解耦（Hopper） | 16x H100/H200 | ~1.3TB |
| 解耦（Blackwell） | 16x B100/B200 | ~1.3TB |

## 注意事项

- 部署前请更新 `model-cache/model-cache.yaml` 中的 `storageClassName`
- 模型下载可能需要 30-60 分钟
- 使用 KV 感知路由以高效利用缓存
- 聚合模式启用了分块预填充（chunked prefill），解耦模式下禁用
