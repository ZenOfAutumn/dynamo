<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# 测试模型尺寸准则

CI 在共享的 L4 GPU（24 GB 显存）上运行。为仅验证管线连通性的测试加载多 GB 大小的模型，会让每次运行多耗费几分钟，并拖慢队列中的其他 PR。

## 适用范围

仅适用于会**加载或引用真实模型**的测试 —— 通常是位于 `tests/serve/`、`tests/router/`、`tests/mm_router/`、`tests/fault_tolerance/`、`tests/kvbm_integration/`、`tests/basic/` 等目录下的 e2e 与集成测试。如果测试 mock 了模型，或者从未引用 Hugging Face 模型 id、`--model` 参数或框架特定的模型路径，则无需遵循本规则。

## 在以下情况下需标注并建议替换为更小的模型

测试加载或引用的模型满足：

- **参数量 ≥7B**（即模型 id 中匹配 `*-7B*`、`*-8B*`、`*-13B*`、`*-32B*`、`*-70B*` 等），**或**
- **任意常见 dtype 下磁盘占用 ≥10 GB**。

如果测试只在验证管线（路由、调度、前端、容错），建议使用本仓库已在使用的小模型之一：

- 文本：`Qwen/Qwen3-0.6B`
- 多模态：`Qwen/Qwen3-VL-2B-Instruct`（或 `Qwen/Qwen2-VL-2B-Instruct`）
- 文本兜底：`TinyLlama/TinyLlama-1.1B-Chat-v1.0`

## 硬性禁止：峰值显存 > 24 GB

如果 `@pytest.mark.profiled_vram_gib(N)`（或等价标记）的 `N > 24`，该测试**无法运行**在 pre/post/nightly CI runner 上。这种情况应作为阻塞项处理，而非建议项：要么将模型/批量/KV 上限缩到 24 GB 以下，要么把测试迁移到运行于更大显存等级的 marker（并确认 CI 矩阵中存在该等级）。

参见 `tests/README.md` 了解 marker 参考以及如何通过 `tests/utils/profile_pytest.py` 推导该数值。

## 缺失 marker：请作者进行 profile

如果测试加载了模型但**未**带上 `@pytest.mark.profiled_vram_gib(N)` marker（以及框架特定的 KV marker —— `requested_vllm_kv_cache_bytes`、`requested_sglang_kv_tokens` 或 `requested_trtllm_kv_tokens`），并行调度器将无法估算其规模，`--max-vram-gib` 过滤会因不安全而排除它。请建议作者使用 `tests/utils/profile_pytest.py` 对测试进行 profile 并补全相应 marker；各框架下的具体调用方式见 `tests/README.md`。

## 确实需要更大模型时的豁免

如果测试满足以下条件，则保留原有模型：

1. 包含注释，说明只有更大的模型才能暴露的内容。
2. 被限定在 `pre_merge` 之外（例如 `@pytest.mark.post_merge`）。
3. 带有 `@pytest.mark.profiled_vram_gib(...)` marker，使并行调度器能够估算其规模。
