<!--
SPDX-FileCopyrightText: Copyright (c) 2024-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Dynamo KVBM

Dynamo KVBM 是面向可扩展 LLM 推理（inference）的分布式 KV 缓存（KV cache）块管理系统。它将内存管理与推理运行时（vLLM、TensorRT-LLM、SGLang）彻底解耦，从而支持 GPU↔CPU↔Disk/Remote 分级存储、异步块的下卸载（offload）/上载（onboard），以及高效的块复用。

![一张展示 Dynamo KV Block manager 分层架构的方框图。](../../../docs/assets/img/kvbm-components.svg)


## 特性亮点

- **分布式 KV 缓存管理：** 统一的 GPU↔CPU↔Disk↔Remote 分级，面向可扩展的 LLM 推理。
- **异步下卸载与复用：** 通过 NIXL 加持的 GDS 加速传输，无需重新计算即可在内存层级之间无缝迁移 KV 块。
- **运行时无关：** 通过轻量连接器与 vLLM、TensorRT-LLM、SGLang 即插即用。
- **内存安全且模块化：** RAII 生命周期 + 可插拔设计，兼顾可靠性、可移植性与后端可扩展性。

## 安装

```bash
pip install kvbm
```

版本兼容性请参考[支持矩阵](../../../docs/reference/support-matrix.md)。

## 从源码构建

pip wheel 通过 Docker 构建流程产出：

```bash
# 渲染并构建启用了 KVBM 的 Docker 镜像（在 dynamo 仓库根目录执行）
python container/render.py --framework dynamo --target runtime --output-short-filename
docker build --build-arg ENABLE_KVBM="true" -f container/rendered.Dockerfile .
```

构建完成后，可二选一：

**方式 1：直接运行并使用容器**
```bash
./container/run.sh --framework none -it
```

**方式 2：将 wheel 文件提取到本地文件系统**
```bash
# 基于已构建的镜像创建一个临时容器
docker create --name temp-kvbm-container local-kvbm:latest

# 将 KVBM wheel 复制到当前目录
docker cp temp-kvbm-container:/opt/dynamo/wheelhouse/ ./dynamo_wheelhouse

# 清理临时容器
docker rm temp-kvbm-container

# 在本地安装 wheel
pip install ./dynamo_wheelhouse/kvbm*.whl
```

注意：当前默认构建的 pip wheel 与 CUDA 13 暂不兼容。


## 集成

### 环境变量

| 变量 | 描述 | 默认值 |
|-----------|--------------|----------|
| `DYN_KVBM_CPU_CACHE_GB` | CPU pinned memory 缓存大小（GB）| 必填 |
| `DYN_KVBM_DISK_CACHE_GB` | SSD 磁盘/存储系统缓存大小（GB）| 可选 |
| `DYN_KVBM_DISK_CACHE_DIR` | 磁盘缓存目录 | `/tmp/` |
| `DYN_KVBM_DISK_ZEROFILL_FALLBACK` | 当 `fallocate()` 不被支持（如 Lustre）时启用零填充 | `false` |
| `DYN_KVBM_DISK_DISABLE_O_DIRECT` | 禁用磁盘 I/O 的 O_DIRECT（用于调试/兼容）| `false` |
| `DYN_KVBM_LEADER_WORKER_INIT_TIMEOUT_SECS` | KVBM leader 与 worker 完成同步并完成所需内存/存储分配的超时时间（秒）。分配大量内存或存储时请适当增大此值。| 120 |
| `DYN_KVBM_METRICS` | 启用指标（metrics）端点 | `false` |
| `DYN_KVBM_METRICS_PORT` | 指标端口 | `6880` |
| `DYN_KVBM_DISABLE_DISK_OFFLOAD_FILTER` | 禁用磁盘下卸载过滤，从而取消 SSD 寿命保护 | `false` |
| `DYN_KVBM_HOST_OFFLOAD_PREFIX_MIN_PRIORITY` | 具备前缀（连续）语义的 CPU 下卸载所需的最低优先级（0-100）：当遇到第一个低于阈值的块时停止下卸载，且后续所有块都会被跳过。用于基于优先级的过滤。| `0`（不过滤）|
| `DYN_KVBM_NCCL_MLA_MODE` | 为 MLA（Multi-Layer Attention）模型（如 DeepSeek）启用 NCCL 复制模式。设为 `true` 时，由 rank 0 从 G2/G3 存储加载 KV 块，再通过 NCCL 广播到所有 GPU，而非每个 GPU 独立加载。需要 MPI，可选启用 `nccl` feature 以获得最佳行为。| `false` |

#### 磁盘存储配置

**为什么可能需要特殊配置：**

某些文件系统（如 Lustre、部分网络文件系统）不支持 `fallocate()`，而 KVBM 用它做快速磁盘空间分配。此外，KVBM 使用 O_DIRECT I/O 以发挥 GPU DirectStorage（GDS）性能，要求严格的 4096 字节对齐。

**针对不支持 fallocate() 的文件系统的设置：**
```bash
export DYN_KVBM_DISK_CACHE_DIR=/mnt/storage/kvbm_cache
export DYN_KVBM_DISK_ZEROFILL_FALLBACK=true  # 当 fallocate() 不支持时启用零填充回退
```

**含义：**
- 不设 `ZEROFILL_FALLBACK=true`：磁盘缓存分配可能因 “Operation not supported” 失败
- 设 `ZEROFILL_FALLBACK=true`：KVBM 使用与 O_DIRECT 兼容的页对齐缓冲区写零

**故障排查：** 如遇 “write all error” 或 EINVAL（errno 22），可尝试禁用 O_DIRECT：`export DYN_KVBM_DISK_DISABLE_O_DIRECT=true`

### vLLM

```bash
DYN_KVBM_CPU_CACHE_GB=100 vllm serve \
  --kv-transfer-config '{"kv_connector":"DynamoConnector","kv_role":"kv_both","kv_connector_module_path":"kvbm.vllm_integration.connector"}' \
  Qwen/Qwen3-8B
```

更多关于与 dynamo 的集成、解耦式服务以及基准测试，请参见 [vllm-setup](../../../docs/components/kvbm/kvbm-guide.md#run-kvbm-in-dynamo-with-vllm)

### TensorRT-LLM

```bash
cat >/tmp/kvbm_llm_api_config.yaml <<EOF
cuda_graph_config: null
kv_cache_config:
  enable_partial_reuse: false
  free_gpu_memory_fraction: 0.80
kv_connector_config:
  connector_module: kvbm.trtllm_integration.connector
  connector_scheduler_class: DynamoKVBMConnectorLeader
  connector_worker_class: DynamoKVBMConnectorWorker
EOF

DYN_KVBM_CPU_CACHE_GB=100 trtllm-serve Qwen/Qwen3-8B \
  --host localhost --port 8000 \
  --backend pytorch \
  --extra_llm_api_options /tmp/kvbm_llm_api_config.yaml
```

更多关于与 dynamo 的集成与基准测试，请参见 [trtllm-setup](../../../docs/components/kvbm/kvbm-guide.md#run-kvbm-in-dynamo-with-tensorrt-llm)


## 📚 文档

- [架构](../../../docs/components/kvbm/README.md#architecture)
- [设计深入](../../../docs/design-docs/kvbm-design.md)
- [NIXL 概览](https://github.com/ai-dynamo/nixl/blob/main/docs/nixl.md)
