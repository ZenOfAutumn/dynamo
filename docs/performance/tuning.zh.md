---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Tuning Disaggregated Performance
---

解耦（disaggregation）通过将预填充（prefill）和解码（decode）分离到不同引擎上来减少二者之间的相互干扰，从而获得性能提升。
然而，要让解耦真正发挥性能，需要对推理（inference）参数进行精细调优。
具体来说，需要调优三组参数：

  1. 引擎配置与选项（如并行映射、最大 token 数等）。
  2. 解耦路由（router）配置与选项。
  3. prefill 与 decode 引擎的数量。

本指南描述了这些参数的调优过程。


## 引擎配置与调优

最重要的引擎配置是并行映射。
对于大多数 dense 模型而言，最佳设置是节点（node）内 TP、节点之间 PP。
例如对于 H100 上的 Llama-405b w8a8，单节点 TP8 或两节点 TP8PP2 通常是最佳选择。
接下来要决定的是用多少 GPU 来服务模型。
通常 GPU 数量与性能的关系符合以下模式：

| GPU 数量                                          | 性能
| :-------------------------------------------------- | :---------------------------------------------------------------------------------------- |
| 无法把权重装入 VRAM                                  | OOM                                                                                       |
| （刚好能把权重装入 VRAM）                            | （KV 缓存（KV cache）太小，无法维持足够长的序列长度或合理 batch size） |
| 拥有相当 KV 缓存的最少 GPU 数量                     | 整体单 GPU 吞吐最佳，单用户延迟最差                                           |
| 介于最少与最多之间                                  | 单 GPU 吞吐与单用户延迟之间的权衡                                          |
| 受通信可扩展性限制的最大 GPU 数量                    | 整体单 GPU 吞吐最差，单用户延迟最佳                                        |
| 超过最大数量                                       | 通信开销主导，性能很差                                                                    |

> [!Note]
> 对于纯 decode 引擎，有时更多 GPU 反而能让每张 GPU 上有更大的 KV 缓存、并发的 decode 请求更多，从而同时获得更好的单 GPU 吞吐与更佳的单用户延迟。
>
> 例如，B200 上 Llama-3.3-70b NVFP4 量化在 vLLM 中以 0.9 的空闲 GPU 显存比例运行：

| TP 大小 | KV 缓存大小（GB） | 每 GPU KV 缓存（GB） | 相对 TP1 的单 GPU 提升 |
| ------: | -----------------: | --------------------: | ---------------------------: |
|       1 |                113 |                   113 |                        1.00x |
|       2 |                269 |                   135 |                        1.19x |
|       4 |                578 |                   144 |                        1.28x |

prefill 与 decode 引擎中最佳 GPU 数量可通过用 [AIPerf](https://github.com/ai-dynamo/aiperf/tree/main) 跑几组固定 ISL/OSL/并发的测试，并与 SLA 比较来确定。
AIPerf 已预装在 dynamo 容器中。

> [!Tip]
> 如果你不熟悉 AIPerf，请先看这份非常实用的[教程](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorial.md)入门。

除并行映射外，其他常调的旋钮还有最大 batch size、最大 token 数与 block size。
对于 prefill 引擎，通常更倾向较小的 batch size 与较大的 `max_num_token`。
对于 decode 引擎，通常更倾向较大的 batch size 与中等的 `max_num_token`。
关于 `max_num_token` 与 max_batch_size 的调优细节见下一节。

关于 block size：太小会导致 P->D KV 缓存传输中的内存块过小、性能差。
过小的 block size 也会在 attention 计算中造成内存碎片，但其影响通常可忽略。
而过大的 block size 会导致前缀缓存命中率低。
对于大多数 dense 模型，我们认为 block size 取 128 是不错的选择。


### GPU 显存比例

各引擎后端（backend）都有自己的 CLI flag 来控制为 KV 缓存预留的 GPU 显存比例（在分配模型权重与激活缓冲之后）：

| 引擎  | CLI flag                         | 引擎专用环境变量                            | 默认值
|---------|----------------------------------|--------------------------------------------|--------
| vLLM    | `--gpu-memory-utilization`       | —                                          | 0.9
| SGLang  | `--mem-fraction-static`          | —                                          | 0.88
| TRT-LLM | `--free-gpu-memory-fraction`    | `DYN_TRTLLM_FREE_GPU_MEMORY_FRACTION`      | 0.9

Dynamo 启动脚本使用绝对的 KV 缓存覆盖值，以实现确定性的、并行安全的 GPU 显存控制。对于 vLLM，`_PROFILE_OVERRIDE_VLLM_KV_CACHE_BYTES` 映射到 `--kv-cache-memory-bytes`。对于 SGLang，`_PROFILE_OVERRIDE_SGLANG_MAX_TOTAL_TOKENS` 映射到 `--max-total-tokens`。这些值由 `tests/utils/profile_pytest.py` 在二分搜索 profiling 时设置，并由 `tests/utils/pytest_parallel_gpu.py` 在运行时设置。

将显存比例调低可以为其他 CUDA 分配（如激活缓冲、NCCL 缓冲）留出更多余量，但代价是更小的 KV 缓存。调高则允许更多并发请求，但有因非 KV 缓存分配 OOM 的风险。生产环境的典型值是 0.85–0.95。

> [!Important]
> 在 vLLM 中，当显式设置 `--kv-cache-memory-bytes`（不是 None）时，KV 缓存大小**会覆盖并忽略** `--gpu-memory-utilization`（参见 [vLLM CacheConfig 文档](https://docs.vllm.ai/en/stable/api/vllm/config/cache/)）。这正是我们使用 `--kv-cache-memory-bytes` 进行并行安全分配的原因：它提供了一个确定性、绝对的 KV 缓存上限，不受 profiling 竞态影响。


## 解耦路由（Disaggregated Router）

解耦路由决定一个请求是在远端 prefill 引擎中预填充，还是在本地 decode 引擎使用 chunked prefill 进行预填充。
对大多数框架，启用 chunked prefill 后，如果一次前向迭代里同时包含 prefill 与 decode 请求，会启动三个 kernel：

  1. 上下文 token 的 attention kernel（TRTLLM 中的 context_fmha kernel）。

  2. decode token 的 attention kernel（TRTLLM 中的 xqa kernel）。

  3. 用于 prefill 与 decode 中合并的活跃 token 的 dense kernel。

### Prefill 引擎

在 prefill 引擎中，最佳策略是工作在能让 GPU 饱和的最小 batch size，从而最小化平均 TTFT（time to first token）。
例如对于 vLLM 中 B200 TP1 的 Llama3.3-70b NVFP4 量化，下图展示了不同 isl 的 prefill 时间（关闭前缀缓存）：

![柱状图与折线图组合，标题为 "Prefill Time"。柱状图为 TTFT（time to first token，毫秒）随 ISL（输入序列长度）的变化。折线图为 TTFT/ISL（每 token 毫秒）随 ISL 的变化。](../assets/img/prefill-time.png)

对于 isl 小于 1000 的情形，prefill 效率较低，因为 GPU 没有充分饱和。
对于 isl 大于 4000 的情形，每 token 的 prefill 时间会变长，因为 attention 在更长历史上的计算更耗时。

当前 Dynamo 中的 prefill 引擎以 batch size 1 工作。
为确保 prefill 引擎处于饱和状态，用户可将 `max-local-prefill-length` 设为饱和点，从而保证 prefill 引擎处于最优。

### Decode 引擎

在 decode 引擎中，最大 batch size 与最大 token 数会影响中间张量的大小。
batch size 与 token 数越大，中间张量越大，KV 缓存越小。
TensorRT-LLM（TRTLLM）有一个不错的[内存占用总结](https://nvidia.github.io/TensorRT-LLM/reference/memory.html)，类似的思想也适用于其他 LLM 框架。

启用 chunked prefill 后，最大 token 数控制能与 decode 一起跑的最长 prefill，并控制 inter-token latency（ITL）。
对于相同的 prefill 请求，较大的最大 token 数会导致生成中较少但较长的停顿，较小的则导致较多但较短的停顿。
然而 chunked prefill 当前在 Dynamo（vLLM 后端）中尚不支持。
因此当前最佳策略是把最大 batch size 设为优化后的 KV 缓存大小，并把最大 token 数设为最大本地 prefill 长度 + 最大 batch size（因为一个 decode 请求贡献一个活跃 token）。


## prefill 与 decode 引擎数量

最佳的 dynamo 旋钮选择取决于模型的运行条件。
我们根据负载定义三种运行条件：

  1. **低负载**：
     端点大多数时间被单个用户（single-stream）访问。

  2. **中负载**：
     端点被多个用户访问，但 decode 引擎的 KV 缓存从未被完全用满。

  3. **高负载**：
     端点被多个用户访问，且由于 decode 引擎中没有可用 KV 缓存而出现请求排队。

低负载下解耦带来的收益不大，因为 prefill 与 decode 通常本来就是分别计算的。
此时通常使用单一整体引擎更好。

中负载下，相比 prefill 优先与 chunked prefill 引擎，解耦能带来更优的 ITL；相比 chunked prefill 引擎与纯 decode 引擎，又能为每个用户带来更佳的 TTFT。
Dynamo 用户可根据 TTFT 与 ITL 的 SLA 调整 prefill 与 decode 引擎数量。

在 KV 缓存容量成为瓶颈的高负载下，解耦对 decode 引擎中 KV 缓存使用的影响如下：

  * 增加 KV 缓存总量：

    * 在 decode 引擎中可使用更大 TP 值，每 GPU 可获得更多 KV 缓存与更高前缀缓存命中率。

    * 当请求被远端预填充时，decode 引擎不需要维护其 KV 缓存（当前 Dynamo 尚未实现）。

    * 更低的 ITL 缩短 decode 时间，让相同 KV 缓存能服务更多请求。

  * 减少 KV 缓存总量：

    * 部分 GPU 被配置为 prefill 引擎，其 KV 缓存在 decode 阶段不可用。

由于 Dynamo 目前在 decode 引擎收到请求时立即分配 KV block，
建议尽量使用更少的 prefill 引擎（甚至不使用 prefill 引擎），以最大化 decode 引擎中可用的 KV 缓存。
为防止在 prefill 引擎排队，用户可设较大的 `max-local-prefill-length`，并让更多 prefill 请求在 decode 引擎处搭车进行。
