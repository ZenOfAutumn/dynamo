---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 已知问题与缓解措施
---

有关 TensorRT-LLM 的通用特性与配置，请参阅[参考指南](trtllm-reference-guide.md)。

---

### KV 缓存耗尽导致 Worker 死锁（分离式服务）

**问题：** 在分离式服务模式下，TensorRT-LLM 的 worker 在长时间高负载流量后可能会卡住、停止响应。一旦进入此状态，worker 需要重启 Pod 或进程才能恢复。

**症状：**
- worker 初始工作正常，但在重负载测试后挂起
- 推理请求被卡住并最终超时
- 日志中出现警告：`num_fitting_reqs=0 and fitting_disagg_gen_init_requests is empty, may not have enough kvCache`
- 错误日志中可能包含：`asyncio.exceptions.InvalidStateError: invalid state`

**根因：** 当缓存收发器配置中的 `max_tokens_in_buffer` 小于正在处理的最大输入序列长度（ISL）时，在重负载下可能发生 KV 缓存耗尽。这会导致上下文传输超时，使 worker 卡住等待已不存在的传输，进入无法恢复的死锁状态。

**缓解措施：** 确保 `max_tokens_in_buffer` 超过你预期的最大输入序列长度。更新引擎配置文件（例如 `prefill.yaml` 与 `decode.yaml`）：

```yaml
cache_transceiver_config:
  backend: DEFAULT
  max_tokens_in_buffer: 65536  # Must exceed max ISL
```

例如，参见 `examples/backends/trtllm/engine_configs/gpt-oss-120b/prefill.yaml`。

**相关 Issue：** [#4327](https://github.com/ai-dynamo/dynamo/issues/4327)
