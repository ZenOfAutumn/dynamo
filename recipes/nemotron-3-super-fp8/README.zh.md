<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Nemotron-3-Super FP8 Recipes

适用于 **nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-FP8**（约 124B 参数的混合 Mamba/Attention/MoE 模型）的多后端可工作部署示例。

这些 recipe 面向 **Dynamo 1.0**。在旧容器上运行的注意事项参见 [Dynamo 0.9.1 兼容性](#dynamo-091-兼容性)。

## 可用配置

| 配置 | GPU | 后端 | 模式 | 描述 |
|--------------|------|---------|------|-------------|
| [**vllm/agg**](vllm/agg/) | 4x H100/H200 | vLLM | 聚合 | TP=4，KV-aware 路由 |
| [**sglang/agg**](sglang/agg/) | 4x H100/H200 | SGLang | 聚合 | TP=4，KV-aware 路由（在 0.9.1 上不可用） |
| [**trtllm/disagg**](trtllm/disagg/) | 4x H100/H200 | TensorRT-LLM | 解耦 | TP=2 P/D 拆分，UCX KV 传输 |
| [**sglang/disagg**](sglang/disagg/) | 4x H100/H200 | SGLang | 解耦 | TP=2 P/D 拆分，nixl KV 传输（在 0.9.1 上不可用） |

## 前置条件

1. **已安装 Dynamo Platform** —— 参见 [Kubernetes 部署指南](../../docs/kubernetes/README.md)
2. **GPU 集群**，配 4x H100 80GB（或 H200）GPU
3. **HuggingFace token**，需具备访问 NVIDIA 模型的权限

## 快速开始

```bash
# 设置 namespace
export NAMESPACE=dynamo-demo
kubectl create namespace ${NAMESPACE}

# 创建 HuggingFace token secret
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token-here" \
  -n ${NAMESPACE}

# 下载模型（先更新 model-cache.yaml 中的 storageClassName！）
kubectl apply -f model-cache/ -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=3600s

# 部署（任选一种配置）
kubectl apply -f vllm/agg/deploy.yaml -n ${NAMESPACE}
# 或：kubectl apply -f trtllm/disagg/deploy.yaml -n ${NAMESPACE}
# 或：kubectl apply -f sglang/agg/deploy.yaml -n ${NAMESPACE}
# 或：kubectl apply -f sglang/disagg/deploy.yaml -n ${NAMESPACE}
```

## 测试部署

```bash
# 端口转发到前端
# 如果部署的是 vllm/agg：
kubectl port-forward svc/nemotron-super-fp8-vllm-agg-frontend 8000:8000 -n ${NAMESPACE}
# 如果部署的是 trtllm/disagg：
# kubectl port-forward svc/nemotron-super-fp8-trtllm-disagg-frontend 8000:8000 -n ${NAMESPACE}
# 如果部署的是 sglang/agg：
# kubectl port-forward svc/nemotron-super-fp8-sglang-agg-frontend 8000:8000 -n ${NAMESPACE}
# 如果部署的是 sglang/disagg：
# kubectl port-forward svc/nemotron-super-fp8-sglang-disagg-frontend 8000:8000 -n ${NAMESPACE}

# 基础对话（带推理）
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-FP8",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 100
  }'

# 工具调用
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-FP8",
    "messages": [{"role": "user", "content": "What is the weather in SF?"}],
    "tools": [{"type": "function", "function": {"name": "get_weather", "parameters": {"type": "object", "properties": {"location": {"type": "string"}}, "required": ["location"]}}}],
    "max_tokens": 256
  }'

# 关闭 thinking（仅在 1.0+ 的 nemotron_nano reasoning parser 上有效）
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-FP8",
    "messages": [{"role": "user", "content": "What is 2+2?"}],
    "chat_template_kwargs": {"enable_thinking": false},
    "max_tokens": 64
  }'
```

## 模型详情

- **模型**：`nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-FP8`
- **架构**：Nemotron-H（混合 Mamba/Attention/MoE，88 层）
- **参数量**：约 124B 总参数（约 119B FP8，约 4.7B BF16）
- **量化**：ModelOpt FP8（F8_E4M3）配 FP8 KV 缓存

## Parser 配置

所有 recipe 都包含工具调用与推理 parser：

- `--dyn-reasoning-parser nemotron_nano`：将 `<think>...</think>` 提取为 `reasoning_content`。能正确处理 `enable_thinking: true` 与 `enable_thinking: false` 两种情况。
- `--dyn-tool-call-parser nemotron_nano`：将 `<tool_call><function=name>` 解析为结构化的 `tool_calls`。

要在请求时禁用推理，传入 `"chat_template_kwargs": {"enable_thinking": false}`。该模型也支持 `"chat_template_kwargs": {"low_effort": true}` 进行更轻量的推理。

## 路由

- **vLLM** 与 **SGLang** recipe 使用**近似 KV-aware 路由**（前端使用 `--router-mode kv --no-kv-events`）。前端使用前缀哈希将请求路由到最可能拥有相关 KV 缓存块的 worker，对共享系统提示或多轮会话的工作负载有帮助。
- **TensorRT-LLM** 解耦 recipe 使用**轮询路由**。Nemotron-H 在 TRT-LLM 上仍要求 `enable_block_reuse: false`，因此 KV overlap 路由在这里不会带来真实的缓存复用收益，反而只会增加误导性的 overlap 记账。

vLLM 与 SGLang 变体使用近似（基于哈希的）路由，因为这些 recipe 中混合 Mamba+Attention 模型尚未具备可靠的 KV 事件路径（vLLM/SGLang 的 `--kv-events-config`、TRT-LLM 的 `--publish-events-and-metrics`）。

## 后端特定说明

### vLLM
- 1.0 中无需 connector flag（默认无 connector）
- 需要 `--is-decode-worker` 跳过 KV 事件 publisher 的初始化
- 需要 `--mamba-cache-mode align` 以绕开 [vllm#34865](https://github.com/vllm-project/vllm/issues/34865)：在默认 `mamba_cache_mode="all"` 下启用前缀缓存会让 Nemotron-H 产生 NaN logprob 和垃圾 token。已在 vLLM 0.17.0 ([vllm#34874](https://github.com/vllm-project/vllm/pull/34874)) 修复；1.0 容器内置 vLLM 0.16.0，因此仍需此 workaround。
- **Attention 后端**：在 Hopper 上默认（`FLASH_ATTN`）是安全的。在 Blackwell 上，vLLM 默认 FlashInfer，它在混合 Mamba 模型上有一个 [陈旧的 NaN bug](https://github.com/vllm-project/vllm/issues/35138)（[vllm#35219](https://github.com/vllm-project/vllm/pull/35219)）。Blackwell 上请指定 `--attention-backend FLASH_ATTN` 或 `--attention-backend TRITON_ATTN` 来回避该问题。
- 设置 `VLLM_FLASHINFER_ALLREDUCE_BACKEND=trtllm` 来避免 TP>1 时 [CUDA graph 捕获过程中的 hang](https://github.com/vllm-project/vllm/issues/35772)。这是较新 vLLM 版本中的[新默认值](https://github.com/vllm-project/vllm/pull/35793)，但在 0.16.0 中必须显式设置。


### TensorRT-LLM
- 使用 PyTorch 后端（engine 配置中 `backend: pytorch`）
- Nemotron-H / Mamba 混合缓存仍不支持 block reuse。所有 TRT-LLM Nemotron 配置中显式设置 `enable_block_reuse: false`。如果省略该字段，当前 TRT-LLM 构建可能仍然能启动，仅仅是因为 Nemotron 模型类悄悄地应用了 `enable_block_reuse: false` 的模型默认值；block reuse 实际上并未启用。
- TRT-LLM 解耦 recipe 使用 `--router-mode round-robin` 而非 KV 路由。在 block reuse 被禁用的情况下，KV-overlap 评分对 Nemotron-H 没有真正的运行时收益。
- **解耦模式**要求 `cache_transceiver_config: backend: UCX`。NIXL 与 MOONCAKE 后端不支持具有 Mamba SSM 状态的混合模型——只有 UCX（或 MPI）能在 worker 之间传输 attention KV 缓存与 Mamba conv/SSM 状态。

### SGLang
- 需要 sglang >= v0.5.9（1.0 内置 v0.5.9；0.9.1 内置 v0.5.8，存在阻塞性 bug）
- **解耦模式可用**，使用 nixl KV 传输（每个 worker TP=2，每个 2 GPU）。Mooncake（`--disaggregation-transfer-backend mooncake`）作为另一种传输后端也受支持。
- 已知问题：prefill 预热日志中会出现 `Prefill warmup failed: 'SamplingParams' object is not subscriptable`——非阻塞，不影响功能

## Dynamo 0.9.1 兼容性

这些 recipe 面向 Dynamo v1.0.0。在 v0.9.1 容器上运行需要做以下变更：

### vLLM（`vllm-runtime:0.9.1`）
- 把镜像 tag 从 `:1.1.1` 改为 `:0.9.1`
- **添加** `--connector none` 到 worker 参数（0.9.1 中需要用它禁用 nixl KV connector；在 1.0 中会被拒绝）
- 把 `--dyn-reasoning-parser` 从 `nemotron_nano` 改为 `deepseek_r1`（nemotron_nano reasoning parser 在 0.9.1 中损坏）
- `enable_thinking: false` 与 `deepseek_r1` parser **不可一起工作**（响应内容会进入 `reasoning_content`，`content` 为 null）
- 仍需要 `--mamba-cache-mode align`（0.9.1 内置 vLLM 0.14.1，同样受 [vllm#34865](https://github.com/vllm-project/vllm/issues/34865) 影响）

### TensorRT-LLM（`tensorrtllm-runtime:0.9.1`）
- 把镜像 tag 从 `:1.1.1` 改为 `:0.9.1`
- 把 `--dyn-reasoning-parser` 从 `nemotron_nano` 改为 `deepseek_r1`
- 同上 vLLM 的 `enable_thinking: false` 警告
- 在 ConfigMap 的 `kv_cache_config` 中保留 `enable_block_reuse: false`。这仍然是 Nemotron-H 在当前 TRT-LLM 构建上的有效设置；省略该字段看起来能工作，仅仅是因为 TRT-LLM 之后会悄悄应用相同的模型默认值。

### SGLang（`sglang-runtime:0.9.1`）
- **不支持。** 内置的 sglang v0.5.8 有两个阻塞性 bug：
  1. FP8 量化 bug（`ModelOptFp8LinearMethod.create_weights()` 签名不匹配）
  2. 配置格式不匹配（`hybrid_override_pattern` vs `layers_block_type`）
- 两者都已在 sglang v0.5.9 修复，但 0.9.1 容器内置 v0.5.8

## 备注

- **解耦模式**：在 TRT-LLM 上通过 UCX（`trtllm/disagg`）以及 SGLang 上通过 nixl 或 mooncake（`sglang/disagg`）受支持。由于混合 KV 缓存不兼容，vLLM 不支持。TRT-LLM disagg 需要 UCX，因为 NIXL/MOONCAKE 不能传输 Mamba SSM 状态。
- **Storage class**：部署前请更新 `model-cache/model-cache.yaml` 中的 `storageClassName`。
- **模型大小**：约 240GB 下载量；视带宽不同，需要 30–60 分钟。
