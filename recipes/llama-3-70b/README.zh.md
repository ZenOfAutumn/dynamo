# Llama-3.3-70B 部署食谱

基于 vLLM 与 FP8 动态量化的 **Llama-3.3-70B-Instruct** 生产可用部署方案。

## 可选配置

| 配置 | GPU | 模式 | 说明 |
|--------------|------|------|-------------|
| [**vllm/agg**](vllm/agg/) | 4x H100/H200 | 聚合（Aggregated） | 单节点，TP4 |
| [**vllm/disagg-single-node**](vllm/disagg-single-node/) | 8x H100/H200 | 解耦（Disaggregated） | 单节点上的预填充（prefill）/解码（decode）分离 |
| [**vllm/disagg-multi-node**](vllm/disagg-multi-node/) | 16x H100/H200 | 解耦（Disaggregated） | 2 节点，每节点 8 GPU |

## 前置条件

1. **已安装 Dynamo Platform** — 参见 [Kubernetes 部署指南](../../docs/kubernetes/README.md)
2. **GPU 集群**，使用满足配置要求的 H100 或 H200 GPU
3. **HuggingFace token**，需具备访问 Llama 模型的权限

## 快速开始

```bash
# 设置命名空间
export NAMESPACE=dynamo-demo
kubectl create namespace ${NAMESPACE}

# 创建 HuggingFace token secret
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token-here" \
  -n ${NAMESPACE}

# 下载模型（请先更新 model-cache.yaml 中的 storageClassName！）
kubectl apply -f model-cache/ -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=3600s

# 部署（任选其一）
kubectl apply -f vllm/agg/deploy.yaml -n ${NAMESPACE}
# 或：kubectl apply -f vllm/disagg-single-node/deploy.yaml -n ${NAMESPACE}
# 或：kubectl apply -f vllm/disagg-multi-node/deploy.yaml -n ${NAMESPACE}
```

## 测试部署

```bash
# 端口转发前端（frontend）
kubectl port-forward svc/llama3-70b-agg-frontend 8000:8000 -n ${NAMESPACE}

# 发送测试请求
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "RedHatAI/Llama-3.3-70B-Instruct-FP8-dynamic",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'
```

## 模型详情

- **模型**：`RedHatAI/Llama-3.3-70B-Instruct-FP8-dynamic`
- **量化**：FP8 动态量化（运行时应用）
- **上下文长度**：模型默认上下文

## 注意事项

- 部署前，请将 `model-cache/model-cache.yaml` 中的 `storageClassName` 修改为与你的集群匹配
- 模型下载大约需要 15-30 分钟，具体取决于网络速度
- 如需集成 GAIE（Gateway API Inference Extension），请 `kubectl apply` 对应子目录中的文件，例如 [vllm/agg/gaie/](vllm/agg/gaie/)
