# Kimi-K2.5 Recipes

使用 TensorRT-LLM 配合 Dynamo KV 感知路由（KV-aware routing）部署 **Kimi-K2.5** 的方案。

另有一个独立的、基于 TokenSpeed 的聚合（aggregated）方案，位于
[`tokenspeed/agg/nvidia/`](tokenspeed/agg/nvidia/README.md)（需要本地构建镜像 —— 目前还没有公共的 Dynamo+TokenSpeed 镜像）。

## 可用配置

存在两种模型权重变体，每种都对应各自的模型下载与部署清单：

| 变体 | 模型 | 状态 | 模态 | 部署配置 | 备注 |
|---------|-------|--------|----------|---------------|-------|
| **baseten** | `baseten-admin/Kimi-2.5-text-nvfp4-v3` | 可用 | 仅文本 | [`deploy.yaml`](trtllm/agg/baseten/deploy.yaml) | 与官方镜像兼容，但尚未做性能优化 |
| **nvidia** | `nvidia/Kimi-K2.5-NVFP4` | 实验性 | 仅文本 | [`deploy.yaml`](trtllm/agg/nvidia/deploy.yaml) 与 [`deploy-specdec.yaml`](trtllm/agg/nvidia/deploy-specdec.yaml) | 所有配置均与当前 top-of-tree 的 Dynamo TRT-LLM 镜像兼容。视觉输入暂未启用 |

所有配置均使用 TP8、EP8、聚合（aggregated）模式 + KV 感知路由。

## 前置条件

1. **已安装 Dynamo Platform** —— 见 [Kubernetes 部署指南](../../docs/kubernetes/README.md)
2. **GPU 集群**：每个 Worker 配 8x B200，或 4 个 Worker（每 Worker 跨 2 节点 2x4）配 GB200
3. **HuggingFace token** 且具备访问该模型的权限

## 硬件要求

| 配置 | GPU |
|--------------|------|
| 聚合（Aggregated）| 8x B200 |
| 聚合 + 投机解码（Speculative Decoding）| 8x4 GB200（4 个 Worker，每个跨 2 节点）|

---

## baseten-admin/Kimi-2.5-text-nvfp4-v3

**状态：** 可用（尚未做性能优化）| **模态：** 仅文本

baseten 变体使用基于底层 DeepSeek-V3 架构的纯文本后端，因此可直接使用官方 TensorRT-LLM 容器镜像 —— 无需打补丁或自定义构建。该方案能够支持基于推理（reasoning）与工具调用（tool calling）的文本推理，但尚未进行性能调优或基准评测。

### 快速开始

baseten 部署清单中预置了占位镜像 `nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:my-tag`。
部署前请将 [`trtllm/agg/baseten/deploy.yaml`](trtllm/agg/baseten/deploy.yaml) 中的 `image:` 字段更新为你实际使用的 Dynamo 发布版本标签。

```bash
# 设置命名空间
export NAMESPACE=dynamo-demo
kubectl create namespace ${NAMESPACE}

# 创建 HuggingFace token Secret
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token-here" \
  -n ${NAMESPACE}

# 下载模型（请先更新 model-cache/model-cache.yaml 中的 storageClassName!）
kubectl apply -f model-cache/baseten/ -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=3600s

# 在 trtllm/agg/baseten/deploy.yaml 中将镜像标签更新为你 Dynamo 发布的标签

# 部署
kubectl apply -f trtllm/agg/baseten/deploy.yaml -n ${NAMESPACE}
```

### 测试部署

```bash
# 端口转发到前端
kubectl port-forward svc/kimi-k25-agg-frontend 8000:8000 -n ${NAMESPACE}

# 发送一次测试请求
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "baseten-admin/Kimi-2.5-text-nvfp4-v3",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 100
  }'
```

---

## nvidia/Kimi-K2.5-NVFP4

**状态：** 可用 | **模态：** 上游仅支持文本

> **仅文本：** 当前上游 TensorRT-LLM 仅通过加载 DeepSeek-V3 的文本主干（`text_config`）来支持 Kimi-K2.5 模型。视觉编码器不会被加载，因此图像输入不会被处理。完整的多模态支持需要上游 TRT-LLM 原生支持 Kimi K2.5。

nvidia 变体支持文本推理，含推理解析（`--dyn-reasoning-parser kimi_k25`）与工具调用（`--dyn-tool-call-parser kimi_k2`）。同时还提供使用投机解码（speculative decoding）的 `deploy-specdec.yaml`。

### 快速开始

nvidia 部署清单使用占位的 top-of-tree 镜像：`nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:my-tag`

部署前请更新所选清单中的 `image:` 字段。

```bash
# 设置命名空间
export NAMESPACE=dynamo-demo
kubectl create namespace ${NAMESPACE}

# 创建 HuggingFace token Secret
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token-here" \
  -n ${NAMESPACE}

# 下载模型（请先更新 model-cache/model-cache.yaml 中的 storageClassName!）
kubectl apply -f model-cache/nvidia/ -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=3600s

# 将部署清单中的镜像更新为容器标签（或打过补丁的标签）

# 部署
kubectl apply -f trtllm/agg/nvidia/deploy.yaml -n ${NAMESPACE}
```

### 测试部署

```bash
# 端口转发到前端
kubectl port-forward svc/kimi-k25-agg-frontend 8000:8000 -n ${NAMESPACE}

# 发送一次测试请求
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Kimi-K2.5-NVFP4",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 100
  }'
```

---

## 模型详情

- **架构**：MoE（Mixture-of-Experts），基于 DeepSeek-V3 架构
- **后端**：TensorRT-LLM（PyTorch 后端）
- **并行**：TP8、EP8（Expert Parallel）
- **量化**：NV FP4

## 验证 Reasoning

部署使用 `--dyn-reasoning-parser kimi_k25` 将模型的思维链（chain-of-thought）抽取到独立的 `reasoning_content` 字段。验证推理是否与最终回答正确分离：

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Kimi-K2.5-NVFP4",
    "messages": [{"role": "user", "content": "What is 2+2? Answer briefly."}],
    "max_tokens": 200
  }' | python3 -m json.tool
```

**预期行为：**

- `message.reasoning_content` 包含模型的思考过程
- `message.content` 仅包含最终回答（如 `"4"`)
- 任一字段中都不会出现原始的 `</think>` 标签

**示例响应：**

```json
{
  "choices": [{
    "message": {
      "content": "4",
      "role": "assistant",
      "reasoning_content": "The user is asking a simple math question: \"What is 2+2?\" and wants a brief answer.\n\n2+2 equals 4.\n\nI should answer briefly as requested."
    },
    "finish_reason": "stop"
  }]
}
```

如果 `reasoning_content` 为 `null`，且 `content` 中包含原始 `</think>` 标签，说明推理解析器未配置。请确认 worker 启用了 `--dyn-reasoning-parser kimi_k25`。

## 验证 Tool Calling

部署使用 `--dyn-tool-call-parser kimi_k2` 将函数调用抽取为 OpenAI 兼容的结构化 `tool_calls`。发送一个带有工具定义的请求：

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Kimi-K2.5-NVFP4",
    "messages": [{"role": "user", "content": "What is the weather in San Francisco?"}],
    "tools": [{
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get the current weather for a location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {"type": "string", "description": "City name"}
          },
          "required": ["location"]
        }
      }
    }],
    "max_tokens": 300
  }' | python3 -m json.tool
```

**预期行为：**

- `message.tool_calls` 是结构化数组，包含 `name`、`arguments` 与 `id`
- `message.content` 仅包含自然语言部分
- `message.reasoning_content` 包含模型选择该工具的推理过程
- `finish_reason` 为 `"tool_calls"`
- `content` 中没有原始 `<|tool_calls_section_begin|>` token

**示例响应：**

```json
{
  "choices": [{
    "message": {
      "content": "I'll check the weather in San Francisco for you.",
      "tool_calls": [{
        "id": "functions.get_weather:0",
        "type": "function",
        "function": {
          "name": "get_weather",
          "arguments": "{\"location\":\"San Francisco\"}"
        }
      }],
      "role": "assistant",
      "reasoning_content": "The user is asking for the weather in San Francisco. I have a function called get_weather that can retrieve weather information. I need to call this function with \"San Francisco\" as the location parameter."
    },
    "finish_reason": "tool_calls"
  }]
}
```

如果 `tool_calls` 缺失，且 `content` 中含有原始的 `<|tool_calls_section_begin|>` token，说明工具调用解析器未配置。请确认 worker 启用了 `--dyn-tool-call-parser kimi_k2`。

## 备注

- 部署前请先更新 `model-cache/model-cache.yaml` 中的 `storageClassName`
