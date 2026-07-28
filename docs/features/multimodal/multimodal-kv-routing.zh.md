---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 多模态 KV 路由
subtitle: 将多模态请求路由到具有最佳 KV 缓存重叠的 worker
---

## 概述

多模态 KV 路由扩展了 Dynamo 的 KV 感知路由（router），在计算缓存重叠分数时考虑图像内容。每个请求都会计算一个图像哈希（`mm_hash`） —— 对 vLLM 后端默认在 Rust 前端中计算，启用 chat-processor 变体时由 vLLM 自有处理器计算，对 TRT-LLM 后端则由专用的 MM 路由 worker 计算 —— 并被纳入按 block 的路由元数据。KV 路由随后选择缓存重叠最高的后端 worker，包括对图像 embedding block 的重叠。

包含相同图像的重复请求会被路由到已缓存了相应 KV 缓存（KV cache）block 的 worker，从而最大化前缀缓存复用。

> 注意：KV cache 与 embedding cache（也称 encoder cache）是分开的，后者复用视觉编码器输出（图像→embedding）以避免重复运行编码器。关于编码器侧复用，参见 [Embedding Cache](https://github.com/ai-dynamo/dynamo/blob/main/docs/features/multimodal/embedding-cache.md)。

## 适用场景

在以下情况使用多模态 KV 路由：

- 你有多个后端 worker 服务多模态请求
- 工作负载中有跨请求重复的图像（如同款商品图、共享参考图像）
- 你希望最大化多模态内容的 KV 缓存命中率

如果没有 MM 感知的路由，标准 router 会把图像 token block 视为不透明，无法识别哪个 worker 缓存了某张图像的 KV block。

## 支持矩阵

| 后端 | 路径 | 是否支持 | 备注 |
|---------|------|-----------|-------|
| **vLLM** | Rust 前端（默认） | ✅ | 使用 lightseek 的 `llm-multimodal` 进行图像 token 计数 + 占位符展开。受支持模型见下表。 |
| **vLLM** | Python chat-processor（`--dyn-chat-processor vllm --router-mode kv`） | ✅ | 使用 vLLM 自有多模态处理器 —— 支持任何 vLLM 支持的 VLM。 |
| **TRT-LLM** | — | ✅ | 使用专用 MM 路由 worker。要求 TRT-LLM worker 启用 `--publish-events-and-metrics`。 |
| **SGLang** | — | ❌ | 暂不支持。 |

## 受支持的模型族（Rust 前端路径）

Rust 前端的 MM 感知路由路径支持 lightseek 的 `llm-multimodal` crate 注册的所有 VLM 系列 —— 最新列表见
[`ImageProcessorRegistry::with_defaults()`](https://docs.rs/llm-multimodal/1.5.0/llm_multimodal/vision/image_processor/struct.ImageProcessorRegistry.html#method.with_defaults)。该 crate 不识别的模型会回退到仅文本前缀的 KV 路由（请求仍可完成；只是图像间没有前缀缓存收益）。

Python chat-processor 变体不受此限制 —— 它委托给 vLLM 自有多模态处理器，支持 vLLM 所支持的任何 VLM。

## 工作原理

### vLLM（默认 —— Rust 前端）

```text
Frontend (Rust + lightseek llm-multimodal + KV router) → 后端 Worker
        │
        ├─ 对图像哈希（原始 URL 的 xxh3_64 —— 全 URL 身份；使用 --frontend-decoding 进行内容寻址哈希）
        ├─ 通过 lightseek 的逐模型 ModelProcessorSpec 解析图像 token id
        ├─ 通过 Range: 0-65535 头部抓取读取 (W, H)（或对内存中 data: 字节读取）
        ├─ lightseek::count_tokens(W, H) → 展开后的图像 token 数 N
        ├─ 在 routing_token_ids 中将占位符展开 × N（worker 的 token_ids 不变）
        ├─ 构造按 block 的 MM 元数据（block_mm_infos）
        ├─ KV router 选择最佳 worker
        └─ 通过 extra_args["mm_hashes"] 将 mm_hash 转发给 worker →
              vLLM 的 multi_modal_uuids（缓存键匹配）
```

1. Rust 前端为每张图像计算 `mm_hash`：对 `data:` URI（以及当模型上启用 `media_decoder` 的 `http(s)://`）使用解码后字节的 `xxh3_64`，否则使用完整 URL 字符串的 `xxh3_64`。仅当两个调用方发送字节相同的 URL 时它们才会共享 `mm_hash`。
2. 图像占位 token id 通过委托给 lightseek 的逐模型 `ModelProcessorSpec`（每种支持的 VLM 家族一份 spec —— Qwen3-VL、Qwen2.5-VL、Qwen2-VL、LLaVA-NeXT、LLaVA-1.5、Phi-3-vision、Llama-4、Kimi-K2.5）解析。每个 spec 读取该模型家族对应的 `config.json` 字段（`image_token_id`、`image_token_index` 或 `media_placeholder_token_id`），当只注册了占位符字符串时回退到探测 tokenizer 词表。注册表不识别的模型会回退到仅文本前缀路由。
3. 每张图像的 `(W, H)` 由一次 64KB 的 `Range` 受限头部抓取读取（或 `data:` URI 的内存字节）；lightseek 的 `llm-multimodal` crate 计算每张图像展开后的 token 数。
4. 在 `routing_token_ids`（仅供 router 使用的视图）中，单个占位 token 被展开为 N 个副本；worker 看到的 `token_ids` 仍为每张图像一个占位符。
5. 按 block 的 MM 元数据（`block_mm_infos`）从展开视图构造；KV router 在所有 worker 之间评估重叠，包括含图像的 block。
6. 前端通过 `extra_args["mm_hashes"]`（16 位十六进制前缀，已对齐填充）转发每张图像的 `mm_hash`；后端 handler 把它们注入为 vLLM 的 `multi_modal_uuids`，使 vLLM 自身的 KV 缓存键与 router 使用的哈希匹配。

### vLLM（替代 —— Python chat-processor 变体）

```text
Frontend (vLLM processor + KV router) → 后端 Worker
        │
        ├─ 下载图像（通过 DynamoMediaConnector，LRU 缓存）
        ├─ 运行 vLLM 的 process_inputs()（HF 处理器，模型无关）
        ├─ 从 mm_features 提取 mm_hash
        ├─ 构造按 block 的 MM 元数据（block_mm_infos）
        ├─ KV router 选择最佳 worker
        └─ 通过 SHM 或 NIXL 传输预处理后的 mm_kwargs
              → 后端跳过 HF 处理器
```

当你希望前端在进程内运行 vLLM 的 HF 图像处理器，并通过共享内存或 NIXL RDMA 把预处理后的 `mm_kwargs` 发到选中的 worker，使后端完全跳过 HF 处理器时，使用此变体（`--dyn-chat-processor=vllm`）。详见下方 [传输模式细节](#传输模式细节-仅-vllm) 中的 `DYNAMO_MM_TRANSFER` flag。

### TRT-LLM

```text
Frontend (round-robin) → MM Router Worker → 后端 Worker
                              │
                              ├─ 下载图像
                              ├─ 计算 mm_hash
                              ├─ 构造按 block 的 MM 元数据
                              └─ KvRouter 选择最佳 worker
```

对于 TRT-LLM，一个专用 MM 路由 worker 位于前端与后端 worker 之间。配置说明见
[TRT-LLM MM Router README](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/trtllm/mm_router_worker/README.md)。

## 启动

### vLLM（默认 —— Rust 前端）

```bash
cd $DYNAMO_HOME
bash examples/backends/vllm/launch/agg_multimodal_router.sh
```

Rust 前端使用 [lightseek 的 `llm-multimodal`][lightseek-crate] crate
（[源码][lightseek-src]）进行逐图像 token 计数与占位符展开。`llm-multimodal` 为每个 VLM 家族（Qwen2/2.5/3-VL、LLaVA、Pixtral 等）提供纯 Rust 实现的 `calculate_num_tokens(W, H, PreProcessorConfig)`，并对照 `transformers` 进行了 golden 测试，因此 router 无需调用 HF 图像处理器即可匹配 vLLM 展开后的图像 token 数。前端随后将每个 `mm_hash` 作为 `multi_modal_uuids` 转发给 worker，使 vLLM 的 KV 事件发布的键与 router 计算的键一致。

[lightseek-crate]: https://crates.io/crates/llm-multimodal
[lightseek-src]: https://github.com/lightseekorg/smg

关键环境变量：

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `MODEL` | `Qwen/Qwen3-VL-2B-Instruct` | 服务模型 |
| `NUM_WORKERS` | `2` | 后端 worker 数量 |
| `BLOCK_SIZE` | `16` | KV 缓存 block 大小（必须与后端一致） |
| `GPU_MEMORY_UTILIZATION` | `0.20` | 每个 worker 的 GPU 显存比例 |
| `SINGLE_GPU` | `false` | 把所有 worker 装到 GPU 0（仅用于测试的覆盖；在单 GPU 机器上做功能测试时传 `--single-gpu` 或设置 `SINGLE_GPU=true`） |
| `KV_EVENTS_PORT_BASE` | `5557` | worker `i` 在 `BASE + i - 1` 上发布 ZMQ KV 事件 |
| `DYN_LOG` | `info,lightseek_mm=debug,...` | 前端日志过滤器 |
| `VLLM_EXTRA_ARGS` | （未设置） | 透传到 `python -m dynamo.vllm` 的参数。设置 `--frontend-decoding` 启用内容寻址 `mm_hash`（跨 URL 的 KV 缓存复用）。 |

如需启用前端图像解码（让前端只下载并解码一次，且 `mm_hash` 变为内容寻址而非 URL 寻址）：

```bash
VLLM_EXTRA_ARGS="--frontend-decoding" \
    bash examples/backends/vllm/launch/agg_multimodal_router.sh
```

worker 随后会在其 model card 上注册一个 `media_decoder`；前端的 `MediaLoader` 在进程内运行，并通过 xxh3 对解码后的 RGB 字节进行哈希。同一图像字节在两个不同（已签名）URL 下会落入相同的路由键。

### vLLM（替代 —— Python chat-processor 变体）

```bash
bash examples/backends/vllm/launch/agg_multimodal_router_chat_processor.sh
```

使用 `--dyn-chat-processor=vllm`，让前端在进程内运行 vLLM 的 HF 处理器。新增 `DYNAMO_MM_TRANSFER` 共享内存/NIXL 预渲染 `mm_kwargs` 在前端与 worker 之间的投递通道。

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `MODEL` | `Qwen/Qwen3-VL-2B-Instruct` | 服务模型 |
| `NUM_WORKERS` | `2` | 后端 worker 数量 |
| `BLOCK_SIZE` | `16` | KV 缓存 block 大小（必须与后端一致） |
| `GPU_MEMORY_UTILIZATION` | `0.40` | 每个 worker 的 GPU 显存比例 |
| `SINGLE_GPU` | `false` | 把所有 worker 装到 GPU 0（仅用于测试的覆盖；在单 GPU 机器上做功能测试时传 `--single-gpu` 或设置 `SINGLE_GPU=true`） |
| `DYNAMO_MM_TRANSFER` | `shm` | 预处理后 mm_kwargs 的传输模式：`shm`（共享内存，同节点）、`nixl`（RDMA，跨节点） |
| `DYNAMO_DISABLE_NIXL_MM` | 未设置 | 设为 `1` 完全禁用 mm_kwargs 传输（后端从 URL 重新处理图像） |

### TRT-LLM

```bash
cd $DYNAMO_HOME/examples/backends/trtllm/mm_router_worker
./launch.sh
```

完整配置说明见
[TRT-LLM MM Router README](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/trtllm/mm_router_worker/README.md)。

## 传输模式细节（仅 vLLM chat-processor 变体）

适用于 `--dyn-chat-processor=vllm` 启动方式（`agg_multimodal_router_chat_processor.sh`），**不**适用于默认的 Rust 前端路径。在 chat-processor 变体下，前端在进程内运行 HF 图像处理器，并把预处理后的 `mm_kwargs` 发到选中的后端 worker，使后端可以跳过重新处理；`DYNAMO_MM_TRANSFER` 环境变量控制该负载的传输方式。

默认 Rust 前端路径不会运行 HF 处理器或预渲染 `mm_kwargs` —— 它只转发 `mm_hashes`，每个 worker 自行重新处理图像。TRT-LLM 后端类似地会重新运行自身的预处理，不响应 `DYNAMO_MM_TRANSFER`。

- **`shm`**（默认）：通过 `/dev/shm` 段使用 POSIX 共享内存。面向同节点部署，前后端共享主机文件系统。如果后端无法访问该段（如运行在不同节点），它会回退到从 URL 重新处理图像。
- **`nixl`**：NIXL RDMA 传输。跨节点部署所需，当 `/dev/shm` 不在前后端之间共享时使用。可跨节点工作于 InfiniBand 或 TCP 之上（由 UCX 选择）。
- **`DYNAMO_DISABLE_NIXL_MM=1`**：完全禁用预处理后 mm_kwargs 传输。后端自行从原始 URL 下载并处理图像。用于调试或当传输开销超过重新处理成本时。
