# Dynamo Metrics 速查表（中文）

> 学习笔记性质的指标汇总。完整规范请见 [`metrics.md`](./metrics.md) 和 [`metrics-developer-guide.md`](./metrics-developer-guide.md)；接 Grafana 的部署见 [`prometheus-grafana.zh.md`](./prometheus-grafana.zh.md)。

按"来源 + 业务面"把 Dynamo 整套可观测体系里出现的所有 Prometheus 指标汇总在一张表里，方便排障时一眼定位。

---

## 一图看懂监控栈

```text
┌────────── 应用层 ──────────┐
│ dynamo_* 指标              │ ← Dynamo 自己代码暴露 (lib/runtime/metrics, lib/llm/...)
└────────────────────────────┘
              │
┌────────── 硬件层 ──────────┐
│ DCGM_FI_*                  │ ← dcgm-exporter 暴露 (NVIDIA 官方 GPU sidecar)
└────────────────────────────┘
              │
┌────────── 集群层 ──────────┐
│ kube_* / container_* /     │ ← kube-state-metrics / cAdvisor / node-exporter
│ node_*                     │
└────────────────────────────┘
              │
              ▼
        Prometheus (统一采集)
              │
              ▼
        Grafana (统一展示)
```

---

## 🟦 一、Frontend 指标（用户视角）

来源：`lib/llm/src/http/` —— LLM 服务的最前端。

| 指标 | 类型 | 含义 |
|---|---|---|
| `dynamo_frontend_requests_total` | Counter | 总请求数（按 request_type/status 分） |
| `dynamo_frontend_inflight_requests` | Gauge | 当前正在处理的请求数 |
| `dynamo_frontend_queued_requests` | Gauge | 当前排队请求数 |
| **`dynamo_frontend_time_to_first_token_seconds`** ⭐ | Histogram | **TTFT**：首字延迟 |
| **`dynamo_frontend_inter_token_latency_seconds`** ⭐ | Histogram | **ITL**：token 间隔延迟 |
| `dynamo_frontend_request_duration_seconds` | Histogram | 整请求耗时 |
| `dynamo_frontend_input_sequence_tokens` | Histogram | 输入 token 数（ISL） |
| `dynamo_frontend_output_sequence_tokens` | Histogram | 输出 token 数（OSL） |
| `dynamo_frontend_output_tokens_total` | Counter | 累计输出 token 数 |
| `dynamo_frontend_cached_tokens` | Histogram | 缓存命中的 token 数 |

> **TTFT 和 ITL 是 LLM 服务最核心的两个延迟指标**：TTFT 由 prefill 决定，ITL 由 decode 决定。

---

## 🟦 二、Component 指标（组件视角）

来源：`lib/runtime/src/metrics/` —— 所有 component（router/worker/prefill/backend...）通用。

| 指标 | 类型 | 含义 |
|---|---|---|
| `dynamo_component_requests_total` | Counter | 组件级请求数（按 `dynamo_component`、`dynamo_endpoint` 分） |
| `dynamo_component_errors_total` | Counter | 错误数 |
| `dynamo_component_inflight_requests` | Gauge | 正在处理请求数 |
| `dynamo_component_request_duration_seconds` | Histogram | 请求耗时（avg/P99 用这个） |
| `dynamo_component_request_bytes_total` | Counter | 入流字节数 |
| `dynamo_component_response_bytes_total` | Counter | 出流字节数 |
| **`dynamo_component_gpu_cache_usage_percent`** ⭐ | Gauge | **KV cache 占用率**（超过 90% 触发抢占） |
| `dynamo_component_total_blocks` | Gauge | 总 KV block 数 |
| `dynamo_component_router_kv_hit_rate` | Histogram | KV 缓存命中率 |

> **关键标签**：
> - `dynamo_component` ∈ {`frontend`, `router`, `prefill`, `backend`(=decode), ...}
> - `dynamo_endpoint` 是具体接口名（如 `generate`、`load`）

---

## 🟦 三、Router 内部开销（性能分析）

来源：`lib/llm/src/kv_router/metrics.rs` —— KV 感知路由器各阶段延迟。

| 指标 | 类型 | 含义 |
|---|---|---|
| `dynamo_router_overhead_total_ms` | Histogram | 路由总耗时 |
| `dynamo_router_overhead_block_hashing_ms` | Histogram | block hash 计算耗时 |
| `dynamo_router_overhead_seq_hashing_ms` | Histogram | 序列 hash 耗时 |
| `dynamo_router_overhead_indexer_find_matches_ms` | Histogram | indexer 前缀匹配耗时 |
| `dynamo_router_overhead_scheduling_ms` | Histogram | 调度决策耗时 |

> 这些指标是 **KV router 性能调优**的核心 —— 看 router 在哪一步慢了。

---

## 🟦 四、Runtime 底层

| 指标 | 类型 | 含义 |
|---|---|---|
| `dynamo_request_plane_roundtrip_ttft_seconds` | Histogram | 请求面 TCP 往返 TTFT |
| `dynamo_tokio_worker_busy_ratio` | Gauge | Tokio worker 线程繁忙比例 |

---

## 🟦 五、Operator 指标（K8s 部署）

来源：`deploy/operator/` —— Dynamo Operator 控制器。

| 指标 | 类型 | 含义 |
|---|---|---|
| `dynamo_operator_reconcile_total` | Counter | reconcile 总次数 |
| `dynamo_operator_reconcile_errors_total` | Counter | reconcile 失败次数 |
| `dynamo_operator_reconcile_duration_seconds` | Histogram | reconcile 耗时 |
| `dynamo_operator_resources_total` | Gauge | 当前管理的 CR 数 |
| `dynamo_operator_webhook_requests_total` | Counter | admission webhook 调用次数 |
| `dynamo_operator_webhook_denials_total` | Counter | webhook 拒绝次数 |
| `dynamo_operator_webhook_duration_seconds` | Histogram | webhook 处理耗时 |

---

## 🟩 六、GPU 指标（dcgm-exporter）

| 指标 | 类型 | 含义 |
|---|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | Gauge | GPU 算力利用率（%） |
| `DCGM_FI_DEV_MEM_COPY_UTIL` | Gauge | 显存带宽利用率（%） |
| `DCGM_FI_DEV_FB_USED` | Gauge | 显存已用（MB） |
| `DCGM_FI_DEV_FB_FREE` | Gauge | 显存空闲（MB） |
| `DCGM_FI_DEV_GPU_TEMP` | Gauge | GPU 温度（°C） |
| `DCGM_FI_DEV_MEMORY_TEMP` | Gauge | 显存温度 |
| `DCGM_FI_DEV_POWER_USAGE` | Gauge | 当前功耗（W） |
| `DCGM_FI_DEV_SM_CLOCK` | Gauge | SM 时钟（MHz） |
| `DCGM_FI_DEV_MEM_CLOCK` | Gauge | 显存时钟 |
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` | Gauge | Graphics engine 活跃比 |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | Gauge | Tensor Core 活跃比 |
| **`DCGM_FI_PROF_NVLINK_TX_BYTES`** ⭐ | Counter | NVLink 发送字节（disagg 必看） |
| **`DCGM_FI_PROF_NVLINK_RX_BYTES`** ⭐ | Counter | NVLink 接收字节 |
| `DCGM_FI_PROF_PCIE_TX_BYTES` | Counter | PCIe 发送字节 |
| `DCGM_FI_PROF_PCIE_RX_BYTES` | Counter | PCIe 接收字节 |

---

## 🟨 七、K8s / 系统指标（外部 exporter）

| 指标 | 来源 | 含义 |
|---|---|---|
| `kube_pod_status_phase` | kube-state-metrics | Pod 状态（Running/Pending/...） |
| `container_cpu_usage_seconds_total` | cAdvisor | 容器 CPU 使用时间 |
| `node_cpu_seconds_total` | node-exporter | Node CPU 时间（按 mode 分） |

---

## 📊 按"看什么用什么"的速查表

| 想观测… | 用这些指标 |
|---|---|
| **用户感知性能** | `frontend_time_to_first_token_seconds` + `frontend_inter_token_latency_seconds` + `frontend_request_duration_seconds` |
| **吞吐量** | `frontend_requests_total` + `frontend_output_tokens_total` |
| **后端瓶颈** | `component_request_duration_seconds{dynamo_component="prefill\|backend"}` + `component_inflight_requests` |
| **KV 缓存效率** | `component_gpu_cache_usage_percent` + `component_router_kv_hit_rate` + `frontend_cached_tokens` |
| **路由开销** | `router_overhead_*_ms`（5 个分阶段指标） |
| **GPU 是否打满** | `DCGM_FI_DEV_GPU_UTIL` + `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` |
| **GPU 内存** | `DCGM_FI_DEV_FB_USED` + `DCGM_FI_DEV_MEM_COPY_UTIL` |
| **disagg KV 传输瓶颈** | `DCGM_FI_PROF_NVLINK_TX/RX_BYTES` |
| **GPU 健康** | `DCGM_FI_DEV_GPU_TEMP` + `DCGM_FI_DEV_POWER_USAGE` |
| **集群健康** | `kube_pod_status_phase` |
| **Tokio runtime 健康** | `tokio_worker_busy_ratio` |
| **K8s Operator 健康** | `operator_reconcile_errors_total` + `operator_webhook_denials_total` |

---

## 📋 命名规范一览

| 前缀 | 暴露方 | 源码位置 |
|---|---|---|
| `dynamo_frontend_*` | Dynamo HTTP 入口 | `lib/llm/src/http/` |
| `dynamo_component_*` | Dynamo 组件通用 | `lib/runtime/src/metrics/` |
| `dynamo_router_*` | KV router | `lib/llm/src/kv_router/metrics.rs` |
| `dynamo_request_plane_*` | RPC 传输层 | `lib/runtime/src/metrics/request_plane.rs` |
| `dynamo_tokio_*` | Tokio runtime | `lib/runtime/src/metrics/tokio_perf.rs` |
| `dynamo_operator_*` | K8s Operator | `deploy/operator/` |
| `DCGM_FI_DEV_*` | GPU 硬件设备 | `dcgm-exporter` |
| `DCGM_FI_PROF_*` | GPU profiling | `dcgm-exporter` |
| `kube_*` | K8s 资源状态 | `kube-state-metrics` |
| `container_*` | 容器资源 | `cAdvisor`（K8s 自带） |
| `node_*` | 主机资源 | `node-exporter` |

---

## 🎯 6 个"必须盯"的核心指标（黄金面板）

```text
1. dynamo_frontend_time_to_first_token_seconds     (TTFT)
2. dynamo_frontend_inter_token_latency_seconds      (ITL)
3. dynamo_frontend_requests_total                   (QPS + error rate)
4. dynamo_component_gpu_cache_usage_percent         (KV 紧不紧)
5. dynamo_component_router_kv_hit_rate              (路由命中率)
6. DCGM_FI_DEV_GPU_UTIL                             (GPU 算力)
```

> 出问题时按这 6 个排查 → 90% 的性能问题能定位到根因。

---

## 一句话总结

> **Dynamo 监控 = 自家 7 大类 `dynamo_*` 指标（frontend / component / router / request_plane / tokio / operator）+ NVIDIA `DCGM_FI_*` GPU 指标 + K8s 系统指标**，覆盖"用户体验 → 组件性能 → 路由内部 → GPU 硬件 → 集群健康"五层，组合起来基本能定位任何性能/可用性问题。

---

## 相关 Dashboard 文件

> 仓库内自带 6 个 Grafana dashboard，可直接导入：

| Dashboard | 路径 | 适用场景 |
|---|---|---|
| 主面板 ★ | `deploy/observability/grafana_dashboards/dynamo.json` | 通用，最先看 |
| Disagg 专属 | `deploy/observability/grafana_dashboards/disagg-dashboard.json` | Prefill/Decode 分离架构 |
| KVBM | `deploy/observability/grafana_dashboards/kvbm.json` | 多层 KV cache |
| Operator | `deploy/observability/grafana_dashboards/dynamo-operator.json` | K8s 部署 |
| DCGM | `deploy/observability/grafana_dashboards/dcgm-metrics.json` | GPU 硬件 |
| SGLang | `deploy/observability/grafana_dashboards/sglang.json` | SGLang backend |

启动完整观测栈：

```bash
docker compose -f deploy/docker-compose.yml -f deploy/docker-observability.yml up -d
# Grafana: http://localhost:3000  (admin/admin)
# Prometheus: http://localhost:9090

