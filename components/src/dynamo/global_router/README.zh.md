<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 全局路由（Global Router）

位于 Dynamo 前端（frontend）与各 pool namespace 中本地路由（router）之间的分层路由服务。全局路由同时支持解耦（disaggregated）与聚合（aggregated）部署，并能基于请求特征灵活选择 pool。

## 概览

全局路由支持两种模式：

- **解耦模式（Disagg，默认）**：同时注册为 prefill 与 decode worker。基于 (ISL, TTFT) 路由 prefill 请求，基于 (context_length, ITL) 路由 decode 请求，分别投递到不同类型的 pool。
- **聚合模式（Agg）**：注册为单一 generate worker。基于 (TTFT 目标, ITL 目标) 路由所有请求到同时处理 prefill 与 decode 的统一 pool。

两种模式都支持基于优先级的、由 agent hint 触发的 pool 覆盖（override）。

## 受支持后端

- **vLLM** — 使用同步 prefill 路径（前端等待 prefill 完成）
- **Mocker** — 使用与 vLLM 相同的同步路径

**未支持：**
- **SGLang** — 尚未实现 bootstrap 路径（异步 KV 传输）
- **TensorRT-LLM** — 尚未实现 bootstrap 路径

## 架构

### 解耦（Disagg）模式

```
Frontend
    |
    v
Global Router (registers as both prefill + decode)
    |
    +---> Prefill Pool 0 (namespace: prefill_pool_0)
    |         |
    |         +---> Local Router ---> Prefill Worker 0
    |                           +---> Prefill Worker 1
    |
    +---> Prefill Pool ...
    |
    +---> Decode Pool 0 (namespace: decode_pool_0)
    |         |
    |         +---> Local Router ---> Decode Worker 0
    |                           +---> Decode Worker 1
    |
    +---> Decode Pool ...
```

### 聚合（Agg）模式

```
Frontend
    |
    v
Global Router (registers as Chat + Completions)
    |
    +---> Agg Pool 0 (namespace: agg_pool_0)
    |         |
    |         +---> Local Router ---> Worker 0 (prefill + decode)
    |                           +---> Worker 1 (prefill + decode)
    |
    +---> Agg Pool 1 (namespace: agg_pool_1)
    |         |
    |         +---> Local Router ---> Worker 0 (prefill + decode)
    |                           +---> Worker 1 (prefill + decode)
    |
    +---> Agg Pool ...
```

## 用法

```bash
python -m dynamo.global_router \
  --config path/to/global_router_config.json \
  --model-name Qwen/Qwen3-0.6B \
  --namespace dynamo
```

### 参数

所有选项均可通过 CLI 标志或环境变量设置。CLI 标志优先于环境变量。

| 参数 | 必填（CLI 或 env） | 环境变量 | 默认值 | 说明 |
|----------|----------------------|---------|---------|-------------|
| `--config` | 是 | `DYN_GLOBAL_ROUTER_CONFIG` | - | JSON 配置文件路径 |
| `--model-name` | 是 | `DYN_GLOBAL_ROUTER_MODEL_NAME` | - | 用于注册的模型名（必须与 worker 一致） |
| `--namespace` | 否 | `DYN_NAMESPACE` | "dynamo" | 全局路由所在 namespace |
| `--component-name` | 否 | `DYN_GLOBAL_ROUTER_COMPONENT_NAME` | "global_router" | 组件名称 |
| `--default-ttft-target-ms` | 否 | `DYN_GLOBAL_ROUTER_DEFAULT_TTFT_TARGET_MS` | None | prefill pool 选择的默认 TTFT 目标（ms） |
| `--default-itl-target-ms` | 否 | `DYN_GLOBAL_ROUTER_DEFAULT_ITL_TARGET_MS` | None | pool 选择的默认 ITL 目标（ms） |

## 配置

配置文件格式因模式而异。`mode` 字段决定使用哪种模式；省略时默认为 `"disagg"`。

### 解耦（Disagg）模式配置

```jsonc
{
    "mode": "disagg",                     // Optional, defaults to "disagg"
    "num_prefill_pools": <int>,
    "num_decode_pools": <int>,
    "prefill_pool_dynamo_namespaces": [],
    "decode_pool_dynamo_namespaces": [],

    "prefill_pool_selection_strategy": {
        "isl_min": <int>,
        "isl_max": <int>,
        "isl_resolution": <int>,
        "ttft_min_ms": <float>,
        "ttft_max_ms": <float>,
        "ttft_resolution": <int>,
        "prefill_pool_mapping": [[]],     // 2D array [isl_resolution][ttft_resolution] -> pool index
        "priority_overrides": []          // Optional
    },

    "decode_pool_selection_strategy": {
        "context_length_min": <int>,
        "context_length_max": <int>,
        "context_length_resolution": <int>,
        "itl_min_ms": <float>,
        "itl_max_ms": <float>,
        "itl_resolution": <int>,
        "decode_pool_mapping": [[]],      // 2D array [context_length_resolution][itl_resolution] -> pool index
        "priority_overrides": []          // Optional
    }
}
```

### 聚合（Agg）模式配置

```jsonc
{
    "mode": "agg",
    "num_agg_pools": <int>,
    "agg_pool_dynamo_namespaces": [],

    "agg_pool_selection_strategy": {
        "ttft_min_ms": <float>,              // Minimum TTFT target (ms)
        "ttft_max_ms": <float>,              // Maximum TTFT target (ms)
        "ttft_resolution": <int>,         // Number of grid rows for TTFT dimension
        "itl_min_ms": <float>,              // Minimum ITL target (ms)
        "itl_max_ms": <float>,              // Maximum ITL target (ms)
        "itl_resolution": <int>,          // Number of grid columns for ITL dimension
        "agg_pool_mapping": [[]],         // 2D array [ttft_resolution][itl_resolution] -> pool index
        "priority_overrides": []          // Optional
    }
}
```

### 为什么聚合模式使用 TTFT × ITL

在聚合模式下，同一个 pool 同时处理 prefill 与 decode。两个 SLA 目标对单次路由决策都很重要：

- **TTFT 目标** 体现用户对 prefill 时延的要求。ISL 已隐式计入 — 发送大 prompt 同时要求严格 TTFT 的用户即在表达"我需要一个快速 pool"。
- **ITL 目标** 体现用户对 decode 时延的要求。在启用分块 prefill（chunked prefill）时，ITL 反映 prefill+decode 的合并争用；不启用时则反映纯 decode 性能。

由此自然形成 pool 划分：
- 严格 TTFT + 严格 ITL → 旗舰交互 pool
- 宽松 TTFT + 严格 ITL → decode 优化 pool
- 严格 TTFT + 宽松 ITL → prefill 优化 pool
- 宽松 TTFT + 宽松 ITL → 批处理/吞吐 pool

### Pool 选择

Pool 选择使用 2D 网格查找。每个维度按 resolution 划分桶。

**Prefill Pool 选择**（解耦模式，基于 ISL 与 TTFT 目标）：

1. 计算 `isl_step = (isl_max - isl_min) / isl_resolution`
2. 计算 `ttft_step_ms = (ttft_max_ms - ttft_min_ms) / ttft_resolution`
3. 对于输入序列长度为 `ISL` 且目标 TTFT 已给的请求：
   - `isl_idx = clamp((ISL - isl_min) / isl_step, 0, isl_resolution - 1)`
   - `ttft_idx = clamp((ttft_target_ms - ttft_min_ms) / ttft_step_ms, 0, ttft_resolution - 1)`
4. 查找 pool：`pool_index = prefill_pool_mapping[isl_idx][ttft_idx]`

**Decode Pool 选择**（解耦模式，基于 context length 与 ITL 目标）：

逻辑相同，但使用 `context_length` 和 `itl_target` 配合 `decode_pool_mapping`。

**Agg Pool 选择**（聚合模式，基于 TTFT 与 ITL 目标）：

逻辑相同，使用 `ttft_target` 与 `itl_target` 配合 `agg_pool_mapping`。

### 基于优先级的 Pool 覆盖

所有策略都支持可选的 `priority_overrides` 规则。当请求带有优先级值（来自 `nvext.agent_hints.priority`）时，全局路由会在网格查找之后再评估覆盖规则。第一条 `[min_priority, max_priority]` 范围包含该优先级的规则获胜，请求被路由到该规则的 `target_pool` 而非网格结果。如果没有规则匹配（或没有优先级），则使用网格结果。

这对 RL 工作负载中的尾部请求缓解很有用：RL 框架可将慢请求标记为高优先级，全局路由将其重定向到专门的低延迟 pool。

```jsonc
"priority_overrides": [
    {
        "min_priority": 10,     // inclusive lower bound
        "max_priority": 100,    // inclusive upper bound
        "target_pool": 1        // pool index to route to
    }
]
```

客户端通过 NVIDIA OpenAI 扩展设置优先级：

```json
{
    "messages": [...],
    "nvext": {
        "agent_hints": {
            "priority": 50
        }
    }
}
```

### 基于优先级的 Pool 覆盖

prefill 与 decode 策略都支持可选的 `priority_overrides` 规则。
当请求携带优先级值（来自 `nvext.agent_hints.priority`）时，
全局路由会**在**网格查找之后评估覆盖规则。第一条 `[min_priority, max_priority]` 范围包含该优先级的规则获胜，
请求被路由到该规则的 `target_pool` 而非
网格结果。若无规则匹配（或没有优先级），则使用网格结果。

这对 RL 工作负载中的尾部请求缓解很有用：RL 框架可将慢请求标记为高优先级，全局路由将其重定向到专门的低延迟 pool。

```jsonc
"priority_overrides": [
    {
        "min_priority": 10,     // inclusive lower bound
        "max_priority": 100,    // inclusive upper bound
        "target_pool": 1        // pool index to route to
    }
]
```

客户端通过 NVIDIA OpenAI 扩展设置优先级：

```json
{
    "messages": [...],
    "nvext": {
        "agent_hints": {
            "priority": 50
        }
    }
}
```

### 传递 SLA 目标

客户端可在请求的 `extra_args` 中传递 TTFT 与 ITL 目标：

```json
{
    "messages": [...],
    "extra_args": {
        "ttft_target": 100,
        "itl_target": 20
    }
}
```

如未提供，则使用配置范围的中间值作为默认值。在解耦模式下，`ttft_target` 驱动 prefill pool 选择，`itl_target` 驱动 decode pool 选择。在聚合模式下，`ttft_target` 与 `itl_target` 共同驱动 pool 选择。

## 请求流

### 解耦模式

1. 前端接收请求并发送给全局路由（注册为 prefill）
2. 全局路由基于 (ISL, TTFT_target, priority) 选择 prefill pool
3. 请求被转发到所选 prefill pool namespace 中的本地路由
4. 本地路由再转发到具体 prefill worker
5. Prefill 响应携带 `disaggregated_params` 返回
6. 前端将 decode 请求发送给全局路由（注册为 decode）
7. 全局路由基于 (context_length, ITL_target, priority) 选择 decode pool
8. 请求被转发到所选 decode pool namespace 中的本地路由
9. tokens 沿链路流式返回

### 聚合模式

1. 前端接收请求并发送给全局路由（注册为 Chat + Completions）
2. 全局路由基于 (TTFT_target, ITL_target, priority) 选择 agg pool
3. 请求被转发到所选 agg pool namespace 中的本地路由
4. 本地路由再转发到同时处理 prefill 与 decode 的 worker
5. tokens 沿链路流式返回

## 示例

完整示例参见 `examples/global_planner/`，包含：
- 全局路由配置
- 各 pool 的本地路由搭建
- 用于测试的 mocker worker
