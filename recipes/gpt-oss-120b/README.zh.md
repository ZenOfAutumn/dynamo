# GPT-OSS-120B 部署示例

基于 Blackwell（GB200）硬件、使用 TensorRT-LLM 部署 **GPT-OSS-120B** 的生产级方案。

## 可用配置

| 配置 | GPU | 模式 | 说明 |
|--------------|------|------|-------------|
| [**trtllm/agg**](trtllm/agg/) | 4x GB200 | 聚合（Aggregated） | WideEP，ARM64 |
| [**trtllm/disagg**](trtllm/disagg/) | 5x Blackwell（GB200/B200） | 解耦（Disaggregated） | 预填充/解码拆分 |

## 先决条件

1. **已安装 Dynamo 平台** —— 参见 [Kubernetes 部署指南](../../docs/kubernetes/README.md)
2. 一个具有 GB200（Blackwell）GPU 的 **GPU 集群（cluster）**
3. **HuggingFace token**，并具备访问该模型的权限

## 快速开始

```bash
# 设置命名空间
export NAMESPACE=dynamo-demo
kubectl create namespace ${NAMESPACE}

# 创建 HuggingFace token Secret
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token-here" \
  -n ${NAMESPACE}

# 下载模型（请先更新 model-cache/model-cache.yaml 中的 storageClassName！）
kubectl apply -f model-cache/ -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=3600s

# 部署
kubectl apply -f trtllm/agg/deploy.yaml -n ${NAMESPACE}
```

## 测试部署

```bash
# 端口转发前端（frontend）
kubectl port-forward svc/gpt-oss-agg-frontend 8000:8000 -n ${NAMESPACE}

# 发送一个测试请求
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-oss-120b",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'
```

## 注意事项

- 部署前请更新 `model-cache/model-cache.yaml` 中的 `storageClassName`
- 该 recipe 需要 ARM64（GB200）节点（node）—— 无法在 x86 Hopper/Ampere 硬件上运行
- 请将 `deploy.yaml` 中的容器镜像 tag 更新为与你的 Dynamo 发行版匹配的版本
