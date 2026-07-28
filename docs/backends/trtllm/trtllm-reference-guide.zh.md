---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 参考指南
subtitle: TensorRT-LLM 后端的特性、配置与运维细节
---

## 构建自定义容器

要从源码构建 TensorRT-LLM 容器（例如用于自定义修改或不同的 CUDA 版本），请参阅[构建自定义容器](./trtllm-building-custom-container.md)指南。

## KV 缓存传输

Dynamo 与 TensorRT-LLM 在分离式服务中支持两种 KV 缓存传输方法：UCX（默认）和 NIXL（实验性）。每种方法的详细信息与配置说明，请参阅 [KV 缓存传输指南](./trtllm-kv-cache-transfer.md)。

## 请求迁移

Dynamo 支持[请求迁移](../../fault-tolerance/request-migration.md)以优雅地处理 worker 故障。启用后，当某个 worker 在生成过程中失败时，请求可以被自动迁移到健康的 worker 上。配置详情请参阅[请求迁移架构](../../fault-tolerance/request-migration.md)文档。

## 请求取消

当用户取消请求（例如从前端断开连接）时，该请求会在所有 worker 上被自动取消，将算力释放给其他请求。

### 取消支持矩阵

| | 预填充 | 解码 |
|-|---------|--------|
| **聚合式** | ✅ | ✅ |
| **分离式** | ✅ | ✅ |

更多细节请参阅[请求取消架构](../../fault-tolerance/request-cancellation.md)文档。

## 多选项（`n`）

Dynamo 通过 `n` 将 OpenAI 兼容的多选项请求转发给 TensorRT-LLM。在 TensorRT-LLM 默认确定性解码路径上，对 `n > 1` 的请求需要在 TensorRT-LLM worker 环境中设置 `TLLM_ALLOW_N_GREEDY_DECODING=1`。否则 TensorRT-LLM 会在生成前拒绝该请求。

如果某项测试或部署确实需要在该路径上验证 `n > 1`，请设置：

```bash
export TLLM_ALLOW_N_GREEDY_DECODING=1
```

将此环境变量限定到需要 `n > 1` 的特定 TensorRT-LLM worker 或测试配置。在 Dynamo E2E 测试中，请将其设置在相关的 `EngineConfig.env` 上而不是全局，并保持客户端请求遵循 OpenAI 形态使用 `n` 而不是添加 `best_of`。

TensorRT-LLM 在 [`tensorrt_llm/sampling_params.py`](https://github.com/NVIDIA/TensorRT-LLM/blob/main/tensorrt_llm/sampling_params.py) 中描述了 `n` / `best_of` 的行为，并将该 guard 校验为 greedy decoding。

## 多模态支持

Dynamo 与 TensorRT-LLM 后端支持多模态模型，使你能在单个请求中同时处理文本与图像（或预先计算的嵌入）。详细的安装步骤、示例请求与最佳实践请参阅 [TensorRT-LLM 多模态指南](../../features/multimodal/multimodal-trtllm.md)。

## 扩散模型支持（实验性）

Dynamo 支持通过 TensorRT-LLM 使用扩散模型生成视频和图像。要求、支持的模型、API 用法与配置选项请参阅[扩散指南](./trtllm-diffusion.md)。

## Logits 处理

Logits 处理器允许你在每一步解码时修改 next-token logits。Dynamo 提供与后端无关的接口和一个面向 TensorRT-LLM 的适配器。API、示例及如何自带处理器请参阅 [Logits 处理指南](./trtllm-logits-processing.md)。

## DP Rank 路由（注意力数据并行）

TensorRT-LLM 为 DeepSeek 等模型支持注意力数据并行，可以将 KV 缓存感知的路由发送到特定的 DP rank。配置与使用细节请参阅 [DP Rank 路由指南](./trtllm-dp-rank-routing.md)。

## KVBM 集成

Dynamo 与 TensorRT-LLM 目前支持与 Dynamo KV Block Manager 集成。该集成可以显著降低首 token 时间（TTFT），尤其在多轮对话和重复的长上下文请求等使用模式中。

具体说明请参见：[在 TensorRT-LLM 中运行 KVBM](../../components/kvbm/kvbm-guide.md#run-kvbm-in-dynamo-with-tensorrt-llm)。

## 可观测性

TensorRT-LLM 暴露 Prometheus 指标用于监控推理性能。详细的指标参考、采集设置与 Grafana 集成请参阅[可观测性指南](./trtllm-observability.md)。

## 在高并发基准测试中禁用 Python 循环 GC

Dynamo 与 TensorRT-LLM 暴露 `DYN_TRTLLM_SERVER_DISABLE_GC`，以匹配 `trtllm-serve` 中 `TRTLLM_SERVER_DISABLE_GC` 的行为。设置后，TensorRT-LLM worker 在启动时会禁用 Python 的循环垃圾回收器，使得分代 GC 暂停不会落在请求热路径上。引用计数的释放仍然正常运行——仅关闭循环回收器。

```bash
export DYN_TRTLLM_SERVER_DISABLE_GC=1
```

这在高并发基准测试中最为有用，可通过消除 GC 引发的尾延迟尖峰来提升吞吐量并稳定 TTFT/ITL 测量。


## 已知问题与缓解措施

已知问题、变通方法与缓解措施请参阅[已知问题与缓解措施](./trtllm-known-issues.md)页面。
