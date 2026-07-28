# Qwen3-32B-FP8 部署配方

面向生产环境的 **Qwen3-32B-FP8** 部署，基于 FP8 量化，使用 TensorRT-LLM 与 vLLM。

## 可用配置

| 配置 | GPU | 模式 | 描述 |
|--------------|------|------|-------------|
| [**trtllm/agg**](trtllm/agg/) | 2x GPU | 聚合（Aggregated） | TP2，轮询路由 |
| [**trtllm/disagg**](trtllm/disagg/) | 8x GPU | 解耦（Disaggregated） | 预填充（prefill）与解码（decode）分离 |
| [**vllm/disagg**](vllm/disagg/) | 8x GPU | 解耦（Disaggregated） | 2× TP2 prefill + 1× TP4 decode |

## 前置条件

1. **已安装 Dynamo Platform** —— 参见 [Kubernetes 部署指南](../../docs/kubernetes/README.md)
2. **GPU 集群**，配备 H100/H200/A100 GPU
3. **HuggingFace token**，且具有 Qwen 模型的访问权限

## 快速开始

```bash
# 设置命名空间
export NAMESPACE=dynamo-demo
kubectl create namespace ${NAMESPACE}

# 创建 HuggingFace token secret
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token-here" \
  -n ${NAMESPACE}

# 下载模型（先在 model-cache.yaml 中更新 storageClassName！）
kubectl apply -f model-cache/ -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=1800s

# 部署（任选其一）
kubectl apply -f trtllm/agg/deploy.yaml -n ${NAMESPACE}
# 或： kubectl apply -f trtllm/disagg/deploy.yaml -n ${NAMESPACE}
# 或： kubectl apply -f vllm/disagg/deploy.yaml -n ${NAMESPACE}
```

## 测试部署

```bash
# 端口转发到前端
# 如部署的是 trtllm/agg：
kubectl port-forward svc/qwen3-32b-fp8-agg-frontend 8000:8000 -n ${NAMESPACE}
# 如部署的是 trtllm/disagg：
# kubectl port-forward svc/qwen3-32b-fp8-disagg-frontend 8000:8000 -n ${NAMESPACE}
# 如部署的是 vllm/disagg：
# kubectl port-forward svc/qwen3-32b-fp8-vllm-disagg-frontend 8000:8000 -n ${NAMESPACE}

# 发送一个测试请求
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-32B-FP8",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'
```

## 模型详情

- **模型**：`Qwen/Qwen3-32B-FP8`
- **后端（backend）**：TensorRT-LLM（PyTorch 后端）与 vLLM
- **量化**：FP8
- **TensorRT-LLM 聚合**：TP=2
- **TensorRT-LLM 解耦**：4× prefill TP=1 + 2× decode TP=2
- **vLLM 解耦**：2× prefill TP=2 + 1× decode TP=4

## 备注

- 部署前请先更新 `model-cache/model-cache.yaml` 中的 `storageClassName`
- 聚合（aggregated）配置使用 CUDA graphs 以优化推理（inference）性能
- KV 缓存（KV cache）使用 FP8 dtype 以提高内存效率
- `vllm/disagg` 配置将 8 张 GPU 切分为 2× prefill（TP=2）+ 1× decode（TP=4），通过
  NixlConnector 进行 KV 传输；所有 worker 必须共置于同一节点
- `vllm/disagg/deploy.yaml` 中设置了 `--max-model-len 8192` 以兼容 A100 40 GB；在
  H100/H200 上可以移除或调大该参数
