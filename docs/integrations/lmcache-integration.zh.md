---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: LMCache
---

## 简介

LMCache 是一个高性能的 KV 缓存（KV cache）层，通过实现 **prefill-once、reuse-everywhere** 的语义来显著加速 LLM 服务。如 [官方文档](https://docs.lmcache.ai/index.html) 所述，LMCache 让 LLM 对每段文本只需进行一次预填充（prefill），它会保存所有可复用文本的 KV 缓存，并允许在任意服务引擎实例之间复用任意可复用文本（不必是前缀）的 KV 缓存。

本文档介绍 LMCache 如何集成到 Dynamo 的 vLLM 后端（backend）中以提升性能与显存效率。

## 聚合式服务

### 配置

通过 `--kv-transfer-config` 标志启用 LMCache：

```bash
python -m dynamo.vllm --model <model_name> --kv-transfer-config '{"kv_connector":"LMCacheConnectorV1","kv_role":"kv_both"}'
```

### 自定义

可通过[此处](https://docs.lmcache.ai/api_reference/configurations.html)列出的环境变量来自定义 LMCache 的配置。

对于更进阶的配置，LMCache 支持多种[存储后端](https://docs.lmcache.ai/index.html)：

- **CPU RAM**：快速的本地内存卸载
- **本地存储**：基于磁盘的持久化
- **Redis**：分布式缓存共享
- **GDS Backend**：用于高吞吐的 GPU Direct Storage
- **InfiniStore/Mooncake**：云原生存储方案

### 部署

可使用提供的启动脚本快速搭建：

```bash
./examples/backends/vllm/launch/agg_lmcache.sh
```

它将：
1. 启动 Dynamo 前端（frontend）
2. 启动一个启用 LMCache 的 vLLM worker

### 聚合模式下的架构

聚合模式下，系统使用：

- **KV Connector**：`LMCacheConnectorV1`
- **KV Role**：`kv_both`（同时处理读取与写入）

## 解耦式服务

解耦式服务将预填充（prefill）与解码（decode）拆分为各自专门的 worker。在生产部署中，这能带来更好的资源利用率与可扩展性。

### 部署

使用提供的解耦启动脚本（至少需要 2 块 GPU）：

```bash
./examples/backends/vllm/launch/disagg_lmcache.sh
```

它将：
1. 启动 Dynamo 前端
2. 在 GPU 0 上启动 decode worker
3. 等待初始化完成
4. 在 GPU 1 上启动启用 LMCache 的 prefill worker

### Worker 角色

#### Decode Worker

- **作用**：负责生成 token（decode 阶段）
- **GPU 分配**：CUDA_VISIBLE_DEVICES=0
- **LMCache 配置**：仅使用 `NixlConnector` 进行 prefill 与 decode worker 之间的 KV 传输

#### Prefill Worker

- **作用**：负责处理 prompt（prefill 阶段）
- **GPU 分配**：CUDA_VISIBLE_DEVICES=1
- **LMCache 配置**：使用同时包含 LMCache 与 NIXL connector 的 `MultiConnector`。这样 prefill worker 既可以使用 LMCache 进行 KV 卸载，也可以使用 NIXL 在 prefill 与 decode worker 之间传输 KV。
- **标志**：`--disaggregation-mode prefill`

## 架构

### KV 传输配置

系统会根据部署模式与 worker 类型自动配置 KV 传输：

#### Prefill Worker（解耦模式）

```python
kv_transfer_config = KVTransferConfig(
    kv_connector="PdConnector",
    kv_role="kv_both",
    kv_connector_extra_config={
        "connectors": [
            {"kv_connector": "LMCacheConnectorV1", "kv_role": "kv_both"},
            {"kv_connector": "NixlConnector", "kv_role": "kv_both"}
        ]
    }
)
```

#### Decode Worker 或聚合模式

```python
kv_transfer_config = KVTransferConfig(
    kv_connector="LMCacheConnectorV1",
    kv_role="kv_both"
)
```

#### 回退（不启用 LMCache）

```python
kv_transfer_config = KVTransferConfig(
    kv_connector="NixlConnector",
    kv_role="kv_both"
)
```

### 集成点

1. **参数解析**（`args.py`）：
   - 配置合适的 KV 传输设置
   - 根据 worker 类型设置 connector 配置

2. **引擎初始化**（`main.py`）：
   - 初始化 LMCache 相关环境变量
   - 使用合适的 KV 传输配置创建 vLLM 引擎
   - 同时支持聚合与解耦模式

### 最佳实践

1. **Chunk 大小调优**：根据使用场景调整 `LMCACHE_CHUNK_SIZE`：
   - 较小的 chunk（128-256）：对内容多变的场景，复用粒度更好
   - 较大的 chunk（512-1024）：对重复模式较多的场景更高效

2. **内存分配**：保守地设置 `LMCACHE_MAX_LOCAL_CPU_SIZE`：
   - 给系统其他进程留足 RAM
   - 在峰值负载时关注内存占用

3. **工作负载优化**：LMCache 在以下场景下表现最佳：
   - 重复出现的 prompt 模式（RAG、多轮对话）
   - 跨会话共享的上下文
   - 长时间运行、缓存已充分预热的服务

## 指标与监控

当通过 `--kv-transfer-config '{"kv_connector":"LMCacheConnectorV1","kv_role":"kv_both"}'` 启用 LMCache 并设置了 `DYN_SYSTEM_PORT` 时，LMCache 指标（metrics）会与 vLLM、Dynamo 指标一起，自动通过 Dynamo 的 `/metrics` 端点暴露。

**访问 LMCache 指标的前提：**

- `--kv-transfer-config '{"kv_connector":"LMCacheConnectorV1","kv_role":"kv_both"}'` —— 启用 LMCache
- `DYN_SYSTEM_PORT=8081` —— 启用 metrics HTTP 端点
- `PROMETHEUS_MULTIPROC_DIR`（可选）—— 未设置时由 Dynamo 内部管理

关于 LMCache 指标的详细信息，包括完整指标列表与访问方式，请参阅 vLLM Prometheus 指标指南中的 **[LMCache Metrics 章节](../backends/vllm/vllm-observability.md#lmcache-metrics)**。

## 故障排查

### LMCache 日志：`PrometheusLogger instance already created with different metadata`

你可能会看到如下错误：

```text
LMCache ERROR: PrometheusLogger instance already created with different metadata. This should not happen except in test
```

**版本说明**：我们在 **vLLM v0.12.0** 中复现了该行为，但在 **vLLM v0.11.0** 中未复现，因此该问题可能仅在（或起源于）v0.12.0 中存在。

当 LMCache connector 在同一进程中被多次初始化时（例如先以 `WORKER` 角色初始化，后以 `SCHEDULER` 角色初始化），LMCache 会输出该日志。LMCache 的 Prometheus logger 是进程级单例，因此第二次初始化的 metadata 与之前不一致时，就会输出该警告。

- **影响**：这是仅日志级别的错误，在我们的测试中不会阻止 vLLM/Dynamo 处理请求。如果你关心 LMCache 的指标标签，需要注意 logger 单例采用的是首次创建时见到的 metadata。
- **不依赖 Dynamo 的复现方式**（vLLM v0.12.0）：

```bash
vllm serve Qwen/Qwen3-0.6B \
  --host 127.0.0.1 --port 18000 \
  --gpu-memory-utilization 0.24 \
  --enforce-eager \
  --no-enable-prefix-caching \
  --max-num-seqs 2 \
  --kv-offloading-backend lmcache \
  --kv-offloading-size 1 \
  --disable-hybrid-kv-cache-manager
```

- **缓解（静默）**：设置 `LMCACHE_LOG_LEVEL=CRITICAL`。
- **上游 issue**：[vLLM issue #30996](https://github.com/vllm-project/vllm/issues/30996)。

### vLLM 日志：`Found PROMETHEUS_MULTIPROC_DIR was set by user`

vLLM v1 使用 `prometheus_client.multiprocess`，并将中间指标值保存在 `PROMETHEUS_MULTIPROC_DIR` 中。

- 如果你**自己设置了 `PROMETHEUS_MULTIPROC_DIR`**，vLLM 会警告该目录必须在每次运行之间被清空，以避免脏的或错误的指标数据。
- 通过 Dynamo 运行时，vLLM 包装层可能会在内部把 `PROMETHEUS_MULTIPROC_DIR` 设到一个临时目录，以避免 vLLM 的清理问题。如果仍然看到该警告，请确认你没有在 shell 或容器环境中显式导出 `PROMETHEUS_MULTIPROC_DIR`。

## 参考资料与扩展阅读

- [LMCache 文档](https://docs.lmcache.ai/index.html) - 全面的指南与 API 参考
- [配置参考](https://docs.lmcache.ai/api_reference/configurations.html) - 详尽的配置项
- [LMCache 可观测性指南](https://docs.lmcache.ai/production/observability/vllm_endpoint.html) - 指标与监控细节

