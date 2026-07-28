---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 多节点示例
---

有关 TensorRT-LLM 的通用特性与引擎配置，请参阅
[参考指南](../trtllm-reference-guide.md)。

## 推荐路径

对于多节点 TensorRT-LLM 部署，建议从代码仓库中的 [`recipes/`](../../../../recipes/README.md) 下的 Kubernetes recipe 入手。这些清单是启动多节点 worker、前端服务以及相关路由组件的受支持入口。

主要的 TRT-LLM recipe 入口如下：

- [DeepSeek-R1 WideEP on GB200](../../../../recipes/deepseek-r1/trtllm/disagg/wide_ep/gb200/deploy.yaml)
- [Qwen3-235B-A22B-FP8 aggregated](../../../../recipes/qwen3-235b-a22b-fp8/trtllm/agg/deploy.yaml)
- [Qwen3-235B-A22B-FP8 disaggregated](../../../../recipes/qwen3-235b-a22b-fp8/trtllm/disagg/deploy.yaml)
- [Qwen3-32B-FP8 aggregated](../../../../recipes/qwen3-32b-fp8/trtllm/agg/deploy.yaml)
- [Qwen3-32B-FP8 disaggregated](../../../../recipes/qwen3-32b-fp8/trtllm/disagg/deploy.yaml)
- [GPT-OSS-120B aggregated](../../../../recipes/gpt-oss-120b/trtllm/agg/deploy.yaml)
- [GPT-OSS-120B disaggregated](../../../../recipes/gpt-oss-120b/trtllm/disagg/deploy.yaml)
- [Nemotron-3-Super-FP8 disaggregated](../../../../recipes/nemotron-3-super-fp8/trtllm/disagg/deploy.yaml)

模型层面的安装、前置条件与硬件说明，请参阅各 recipe 的 README 文件：

- [DeepSeek-R1 recipes](../../../../recipes/deepseek-r1/README.md)
- [Qwen3-235B-A22B-FP8 recipes](../../../../recipes/qwen3-235b-a22b-fp8/README.md)
- [Qwen3-32B-FP8 recipes](../../../../recipes/qwen3-32b-fp8/README.md)
- [GPT-OSS-120B recipes](../../../../recipes/gpt-oss-120b/README.md)
- [Kimi-K2.5 recipes](../../../../recipes/kimi-k2.5/README.md)

## 快速开始

整体来说，Kubernetes 工作流程为：

1. 在 Kubernetes 上安装 Dynamo 平台。参见
   [Kubernetes 部署指南](../../../kubernetes/README.md)。
2. 创建命名空间以及所需的密钥，例如 Hugging Face token。
3. 当 recipe 包含模型缓存与模型下载清单时，将其应用到集群。
4. 应用该 recipe 的 `deploy.yaml`。
5. 端口转发前端服务，并向 `/v1/models` 或 `/v1/chat/completions` 发送测试请求。

示例流程：

```bash
export NAMESPACE=dynamo-demo
kubectl create namespace ${NAMESPACE}

kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token-here" \
  -n ${NAMESPACE}

# Example: deploy DeepSeek-R1 TRT-LLM WideEP on GB200.
kubectl apply -f recipes/deepseek-r1/model-cache/model-cache.yaml -n ${NAMESPACE}
kubectl apply -f recipes/deepseek-r1/model-cache/model-download.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=7200s
kubectl apply -f recipes/deepseek-r1/trtllm/disagg/wide_ep/gb200/deploy.yaml -n ${NAMESPACE}
```

部署就绪后，端口转发由 recipe 命名的前端服务，并发送测试请求：

```bash
kubectl port-forward svc/<frontend-service> 8000:8000 -n ${NAMESPACE}

curl http://localhost:8000/v1/models
```

## 注意事项

- 启动与部署流程使用的 TRT-LLM 引擎配置文件位于
  [`examples/backends/trtllm/engine_configs/`](../../../../examples/backends/trtllm/engine_configs/README.md)。
- 如需自定义模型并行、副本数或路由模式，请编辑 recipe 本地的清单，而不是引入额外的、与调度器绑定的指南。
- 当前受支持的 recipe 目录请见 [recipes/README.md](../../../../recipes/README.md)。
