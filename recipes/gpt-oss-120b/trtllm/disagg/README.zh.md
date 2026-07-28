<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# GPT-OSS-120B 解耦式预填充/解码（Disaggregated Prefill/Decode）

在 GB200 节点上，使用 TensorRT-LLM 通过 Dynamo 以解耦的预填充（prefill）/解码（decode）方式提供 [openai/gpt-oss-120b](https://huggingface.co/openai/gpt-oss-120b) 模型服务。

## 拓扑

| 角色    | 节点数 | 每节点 GPU 数 | 总 GPU 数 | 并行策略 |
|---------|-------|-----------|------------|-------------|
| Prefill | 1     | 1         | 1          | TP1         |
| Decode  | 1     | 4         | 4          | TP4         |

## 前置条件

1. **已安装 Dynamo Platform** —— 见 [Kubernetes 部署指南](../../../../docs/kubernetes/README.md)
2. **Blackwell GPU 节点**（GB200 或 B200）
3. **HuggingFace token**，且具备访问该模型的权限

## 部署

按照[顶层快速开始](../../README.md)完成命名空间、HuggingFace token Secret 与模型下载的准备，然后执行：

```bash
kubectl apply -f trtllm/disagg/deploy.yaml -n ${NAMESPACE}
```

观察启动过程（视存储速度而定，模型加载约需 15–30 分钟）：

```bash
kubectl get pods -n ${NAMESPACE} -l app.kubernetes.io/part-of=gpt-oss-disagg -w
```

## 测试

```bash
kubectl port-forward svc/gpt-oss-disagg-frontend 8000:8000 -n ${NAMESPACE} &
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"openai/gpt-oss-120b","messages":[{"role":"user","content":"Hello!"}],"max_tokens":50}'
```

## 基准测试（可选）

编辑 `perf.yaml` 设置你的 namespace 与 PVC，然后执行：

```bash
kubectl apply -f trtllm/disagg/perf.yaml -n ${NAMESPACE}
kubectl logs -f -l job-name=gpt-oss-120b-disagg-bench -n ${NAMESPACE}
```

## 关键配置说明

### 引擎配置（Engine Configs）

`deploy.yaml` 包含一个 ConfigMap，分别给出预填充与解码 Worker 的引擎配置。主要差异：

- **Prefill**：TP1，`max_batch_size=64`，`free_gpu_memory_fraction=0.8`，禁用 overlap scheduler
- **Decode**：TP4，`max_batch_size=1280`，`free_gpu_memory_fraction=0.85`，启用 overlap scheduler

### KV 传输

使用基于 UCX 的 cache transceiver（`max_tokens_in_buffer=9216`），用于在预填充与解码 Worker 之间传输 KV 缓存（KV cache）。

### 量化

通过环境变量 `OVERRIDE_QUANT_ALGO` 启用 `W4A8_MXFP4_MXFP8` 量化方案。
