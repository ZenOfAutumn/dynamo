---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 编码器分离（Encoder Disaggregation）
subtitle: 将视觉编码独立为专门的 worker，便于独立扩缩容
---

## 概述

编码器分离将视觉编码器从 prefill/decode 流水线中拆出，作为独立 worker。不再内联运行图像编码，而是由专门的 encode worker 处理媒体并通过 NIXL（RDMA）将得到的嵌入传输给下游 worker。

这带来：

- 根据视觉工作负载独立扩缩 encode worker
- 减轻 prefill/decode worker 的 GPU 显存压力
- 通过将 worker 数量与实际瓶颈匹配来获得更好的 GPU 利用率

## 何时使用

在以下情形使用编码器分离：

- 视觉编码是瓶颈，需要独立于 LLM worker 进行扩缩
- 希望将视觉编码器运行在不同硬件上（例如更小的 GPU 用于编码、更大的 GPU 用于 LLM 推理）
- 部署需处理大量多模态请求，编码吞吐量是限制因素

对于简单部署或开发 / 测试，聚合式（EPD）模式更易上手。

## 支持矩阵

| 后端 | E/PD | E/P/D | 备注 |
|---------|------|-------|-------|
| **vLLM** | ✅ | ✅ | 独立 encode worker 当前处理 `image_url` 输入；`video_url` 输入仍走 prefill/PD 路径 |
| **TRT-LLM** | ❌ | ✅ | 通过 `MultimodalEncoder` 支持图像 URL，并通过 NIXL 支持预先计算的嵌入 |
| **SGLang** | ✅ | ✅ | 嵌入使用 NIXL；P/D KV 传输使用 bootstrap 机制 |

## 部署模式

**E/PD** — 独立 encoder，prefill+decode 合并：

```text
Frontend → Processor → Encode Worker → PD Worker → Response
                           (NIXL)
```

Encode worker 运行视觉模型并通过 NIXL 将嵌入传输给合并的 prefill+decode worker。

**E/P/D** — 所有阶段都分离：

```text
Frontend → Processor → Encode Worker → Prefill Worker → Decode Worker → Response
                           (NIXL)          (KV transfer)
```

每个阶段都使用独立的 worker，完全分离。Encode worker 将嵌入传输给 prefill worker，prefill worker 再将 KV 缓存传输给 decode worker。

## 启动

### vLLM

```bash
cd $DYNAMO_HOME/examples/backends/vllm

# E/PD
bash launch/disagg_multimodal_e_pd.sh --model "Qwen/Qwen3-VL-30B-A3B-Instruct-FP8"

# E/P/D
bash launch/disagg_multimodal_epd.sh --model "Qwen/Qwen3-VL-30B-A3B-Instruct-FP8"
```

### TRT-LLM

```bash
cd $DYNAMO_HOME/examples/backends/trtllm

# E/PD
bash launch/disagg_e_pd.sh

# E/P/D
./launch/epd_multimodal_image_and_embeddings.sh
```

### SGLang

```bash
cd $DYNAMO_HOME/examples/backends/sglang

# E/PD
./launch/multimodal_epd.sh

# E/P/D
./launch/multimodal_disagg.sh
```

完整配置细节与组件标志请参阅各后端文档（[vLLM](https://github.com/ai-dynamo/dynamo/blob/main/docs/features/multimodal/multimodal-vllm.md)、[TRT-LLM](https://github.com/ai-dynamo/dynamo/blob/main/docs/features/multimodal/multimodal-trtllm.md)、[SGLang](https://github.com/ai-dynamo/dynamo/blob/main/docs/features/multimodal/multimodal-sglang.md)）。
