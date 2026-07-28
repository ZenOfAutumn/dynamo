---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Planner 设计
---

> **Tier 3 设计文档**，面向贡献者和架构师。面向用户的文档见 [docs/components/planner/](../components/planner/README.md)。

## 概览

Planner 是 Dynamo 的自动扩缩控制器。它支持两种扩缩模式：**基于吞吐**（throughput-based，使用 profiling 数据和流量预测）和 **基于负载**（load-based，使用实时引擎指标和在线回归）。本文介绍两种模式的内部架构、算法和设计取舍。

## 基于吞吐的扩缩

![Planner 架构：Metric Collector、Load Predictor、Performance Interpolator 共同驱动 Scaling Algorithm 和 Connector Layer](../assets/img/planner-architecture.svg)

## 扩缩算法

### Step 1：指标采集

每隔 `adjustment_interval` 秒，Planner 向 Prometheus 查询：

- 区间内平均 TTFT 和 ITL
- 总请求数
- 平均输入序列长度（ISL）和输出序列长度（OSL）

Prometheus 查询面向 Frontend 的 `/metrics` 端点，该端点暴露 histogram 和 counter 指标。

### Step 2：修正因子（Correction Factor）计算

Planner 维护"修正因子"，把基于 profiling 的预测调整到符合实际行为：

```text
prefill_correction = actual_ttft / expected_ttft
decode_correction  = actual_itl  / expected_itl
```

这些因子用于吸收难以建模的因素，例如：

- **请求排队**：突发流量导致 TTFT 高于 profiling 的稳态值
- **Prefix cache 命中**：KV 复用降低了有效 prefill token 数，实际 TTFT 偏低
- **decode 中的 chunked prefill**：小 prefill 混入 decode 引擎，影响 ITL
- **指标方差**：平均 ISL/OSL 可能无法代表真实分布

修正因子作为乘子作用于下一次扩缩决策。使用 `--no-correction` 可在调试或冷启动伪影主导时关闭。

### Step 3：负载预测

Planner 对下一个区间预测三个量：

- `next_num_req`：请求数
- `next_isl`：平均输入序列长度
- `next_osl`：平均输出序列长度

有四种预测器实现：

| 预测器       | 算法                                       | 适用场景                         |
| ------------ | ------------------------------------------ | -------------------------------- |
| **Constant** | `next = current`                           | 稳定负载、长区间                 |
| **ARIMA**    | Auto-ARIMA，可选 log1p 变换                | 趋势/季节性模式                 |
| **Kalman**   | 局部线性趋势 Kalman 滤波                   | 突发流量                         |
| **Prophet**  | Facebook Prophet 时间序列模型              | 复杂季节性                       |

所有预测器都支持从 trace 文件做 warm-start（`--load-predictor-warmup-trace`）。

### Step 4：副本数计算

**Prefill 副本：**

```python
predicted_load = next_requests * next_isl / interval * min(1, prefill_correction)
prefill_replicas = ceil(predicted_load / interpolated_throughput / gpus_per_engine)
```

prefill 修正因子对吞吐是线性影响，因为 prefill 是单批的。

**Decode 副本：**

```python
# 把修正因子应用到 ITL SLA 目标
corrected_itl = target_itl / decode_correction_factor

# 在预测的上下文长度下，找到达成 corrected ITL 的最佳 throughput/GPU
throughput_per_gpu = decode_interpolator.find_best_throughput_per_gpu(
    itl=corrected_itl,
    context_length=next_isl + next_osl / 2
)

# 计算所需副本
decode_replicas = ceil(next_num_req * next_osl / interval / throughput_per_gpu / gpus_per_engine)
```

### Step 5：扩缩执行

Planner 调用 `connector.set_component_replicas()`，传入计算出的目标值。默认非阻塞：Planner 在副本调整期间继续监控。

## Connector 设计

### 接口

```python
class PlannerConnector(ABC):
    async def add_component(self, component_name)
    async def remove_component(self, component_name)
    # 扩展接口（不在 ABC 上，但两种 connector 都实现了）：
    async def set_component_replicas(self, targets, blocking)
    async def validate_deployment(self, ...)
    async def wait_for_deployment_ready(self)
```

### KubernetesConnector

直接 PATCH DGD 资源以更新副本数。Operator 监听 DGD 变化并 reconcile 组件部署。

**设计要点：**

- 通过 `DYN_PARENT_DGD_K8S_NAME` 找到父 DGD（由 operator 注入）
- 通过 `subComponentType` 字段（prefill/decode）解析服务，并 fallback 到旧组件名
- 启动时校验部署结构：确认 prefill 和 decode 服务都存在且模型名一致

### VirtualConnector

用于非原生环境（如自定义编排器）。通过 `VirtualConnectorCoordinator`（Rust binding）把扩缩决策写入分布式运行时。外部系统通过 `VirtualConnectorClient` 轮询决策并报告完成。

**扩缩决策流程：**

1. Planner 把 `(num_prefill, num_decode, decision_id)` 写入运行时
2. 外部系统通过 `client.wait()` 读取决策
3. 外部系统执行扩缩
4. 外部系统通过 `client.complete(decision)` 报告完成
5. Planner 看到 `scaled_decision_id >= decision_id`，继续推进

**超时**：1800 秒（可配）内未收到扩缩确认，Planner 也会推进新决策。

## 性能插值

Planner 使用预部署阶段的 profiling 数据（NPZ 文件）建立 (throughput, ISL/OSL, context_length) → (TTFT, ITL) 的映射。这些数据来自 SLA 驱动的 profiling 流程（在线 GPU profiling 或 AI Configurator 估算）。

维护两个插值器：

- **Prefill 插值器**：(throughput_per_gpu, ISL) → TTFT
- **Decode 插值器**：(throughput_per_gpu, context_length) → ITL

插值器精度由 profiling 扫描粒度决定。粒度越细，profiling 样本越多，插值越精确。

## 初始化

Planner 当前会等 30 秒（`INIT_PLANNER_START_DELAY`，位于 `components/src/dynamo/planner/__main__.py`），作为其他组件（frontend、worker）注册和稳定的临时方案；参见 [已知限制](#known-limitations) 中关于使用就绪探针替代该等待的规划。

延时之后：

1. 初始化 connector（按 `--environment` 选 K8s 或 Virtual）
2. 校验部署结构
3. 加载 profiling 结果
4. 构建插值器
5. 初始化负载预测器
6. 进入主扩缩循环

## 性能考量

- **adjustment_interval 设置**：区间必须长于扩缩操作完成时间。如果 `adjustment_interval` 短于添加/移除 worker 的时间（包括 Pod 调度、模型加载、服务注册），扩缩决策会叠加。默认 180 秒较保守；模型加载快的负载可以用更短区间。
- **修正因子稳定性**：每个区间都会重算修正因子。流量切换期间（如 ramp-up）可能震荡。`--no-correction` 用于冷启动伪影主导、扭曲修正因子的场景。
- **插值精度 vs profiling 成本**：profiling 扫描中提高 `prefillInterpolationGranularity` 和 `decodeInterpolationGranularity` 能提升插值精度，但线性增加 profiling 时间。默认粒度（prefill=16、decode=6）在精度和时长之间取得平衡。
- **预测器 warm-up 期**：所有预测器都需要历史观测才能稳定预测。ARIMA 和 Prophet 需要多个 adjustment 区间的数据；Kalman 在累计 `--kalman-min-points` 个观测后才开始预测。在 warm-up 期间，Planner 用 Constant 预测器兜底。

## 基于负载的扩缩

负载模式（load-based）使用 Dynamo 事件平面投递的 ForwardPassMetrics（FPM）做 SLA 感知的扩缩决策，不依赖 profiling 数据或 KV Router。

### 指标

每个引擎通过 ZMQ → FpmEventRelay → 事件平面 发布每轮迭代的 `ForwardPassMetrics`。Planner 通过 `FpmEventSubscriber` 订阅，自动发现引擎，并基于 MDC 做生命周期追踪。关键字段：
- **wall_time**：单次迭代执行时间（回归目标）
- **scheduled_requests.sum_prefill_tokens**：prefill 回归输入
- **scheduled_requests.sum_decode_kv_tokens**：decode 回归输入
- **queued_requests**：排队中的 prefill/decode 负载，用于 TTFT/ITL 模拟
- 心跳空闲帧（wall_time=0）会被跳过

### 诊断

每个 tick，扩缩状态机会把中间决策数据填充进 `TickDiagnostics`——预估延迟、预测负载、每引擎 RPS、决策原因——通过内部 `_diag_*` 字段。适配层从 `PlannerEffects.diagnostics` 读取，然后：

- 设置 Prometheus gauge（如 `dynamo_planner_estimated_ttft_ms` 等）
- 用 enum metric 记录 load-scaling 决策原因（`dynamo_planner_load_scaling_decision`）
- 喂给 `DiagnosticsRecorder`，按周期累积 tick 快照，并基于 Plotly 生成 HTML 报告

`_collect_fpm()` 收集的每引擎 FPM 队列深度作为带 label 的 Prometheus gauge 导出。

### 回归模型

三个专用回归模型（`fpm_regression.py`）：
- **PrefillRegressionModel**：一维回归 `sum_prefill_tokens → wall_time`。通过模拟 chunked prefill 调度（按 `max_num_batched_tokens` 切 chunk）估算 TTFT。
- **DecodeRegressionModel**：一维回归 `sum_decode_kv_tokens → wall_time`。估算总 decode 负载（已调度 + 排队 + 平均 decode 长度）的 ITL。
- **AggRegressionModel**：二维回归 `(sum_prefill_tokens, sum_decode_kv_tokens) → wall_time`。同时估算 TTFT（在 decode 搭载下模拟 prefill）和 ITL（在平均搭载 prefill 的情况下做 decode）。

### 扩缩决策

- **Prefill/Decode**：所有引擎的预估 TTFT/ITL 都 > SLA 才扩；都 < SLA × sensitivity 才缩
- **Agg**：(所有 TTFT > SLA) **或** (所有 ITL > SLA) 才扩；(所有 TTFT < SLA × sensitivity) **且** (所有 ITL < SLA × sensitivity) 才缩
- 每个区间只 ±1（非阻塞 + pending-desired 保护：扩缩进行中仍继续观测指标，但在前一次扩缩完成前不会下达新动作）

### 与基于吞吐的扩缩共存

两种模式同时启用时：基于吞吐的扩缩（长区间）为副本数设下限，基于负载的扩缩（短区间）在该下限之上做实时调整。

### Aggregated 模式

聚合模式下（`--mode agg`），引擎通过 chunked prefill 同时承担 prefill 和 decode。Planner 同时维护 TTFT 和 ITL 回归模型，但回归训练用每个 worker 的时间平均指标（而非瞬时值），以平滑 chunked prefill 的噪声。任一信号过载就扩，两个信号都不饱和才缩。

## 已知限制

1. **30 秒启动延时**：硬编码的组件注册等待，应替换为基于运行时就绪探针的方案。
2. **adjustment_interval vs 扩缩延迟**：若 `adjustment_interval` < 扩缩耗时，决策会堆积。Planner 会记 warning 但不入队。
3. **基于平均值的插值**：基于吞吐的扩缩使用平均 ISL/OSL，对双峰或重尾分布表达力有限。
4. **单 DGD 作用域**：每个 Planner 实例只管理一个 DGD，不支持跨模型/跨 DGD 协同。

## 未来工作

- 跨 DGD 协同（共享集群场景）
- 分布感知的插值（超越 ISL/OSL 均值）
- 基于实测扩缩延迟的自适应 adjustment_interval

## 文件地图

| 文件                         | 用途                                                  |
| ---------------------------- | ----------------------------------------------------- |
| `planner_core.py`            | Base planner，共享扩缩主循环，算法核心                 |
| `disagg_planner.py`          | 分离模式编排器（prefill + decode）                     |
| `agg_planner.py`             | 聚合模式编排器（仅 load-based）                        |
| `prefill_planner.py`         | Prefill 专属扩缩逻辑                                   |
| `decode_planner.py`          | Decode 专属扩缩逻辑                                    |
| `load_based_regression.py`   | load-based 扩缩用的滑窗线性回归                        |
| `prometheus.py`              | Prometheus / Router 指标客户端、数据类                 |
| `perf_interpolation.py`      | NPZ 数据加载与吞吐/延迟插值                            |
| `load_predictor.py`          | ARIMA、Prophet、Kalman、Constant 预测器                |
| `pre_swept_results_utils.py` | 预计算的 H100/H200 profiling 数据加载器                |
| `kubernetes_connector.py`    | K8s API 对接，DGD 扩缩                                 |
| `kube.py`                    | 底层 K8s 客户端封装                                    |
| `exceptions.py`              | 自定义异常体系                                         |
| `defaults.py`                | 默认配置、后端名映射                                   |
| `planner_argparse.py`        | CLI 参数定义                                           |

