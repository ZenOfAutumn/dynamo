# Grafana Dashboard 指标说明

本文档介绍 `disagg-dashboard.json` 中每个面板的数据来源以及它们的展示方式。

## 仪表板组织

仪表板按**逻辑请求流顺序**组织（共 21 个面板，分布在 6 行中）：

**第 1 行：前端（Frontend）健康度**（面向用户的指标 - y=0）
- Frontend Requests/Sec（x=0）、Avg TTFT（x=8）、Avg Request Duration（x=16）

**第 2 行：前端详情**（y=8）
- Avg Inter-Token Latency（x=0）、Avg ISL/OSL（x=8）、**Queued Requests** ⭐（x=16）

**第 3 行：Prefill Worker**（典型瓶颈所在！- y=16）
- Prefill Worker Processing Time ⭐（x=0）、Prefill Worker Throughput（x=8）、Component Latency Comparison（x=16）

**第 4 行：Decode Worker**（y=24）
- Request Throughput（x=0）、Avg Request Duration（x=8）、KV Cache Utilization (%)（x=16）

**第 5 行：KV 缓存（KV cache） + GPU**（y=32）
- KV Cache Blocks (Total)（x=0）、GPU Compute Utilization（x=8）、GPU Memory Used（x=16）

**第 6 行：NIXL Transfer 指标**（y=40）
- GPU Memory Bandwidth（x=0）、NVLink Bandwidth (GB/s)（x=8）、Worker CPU Usage（x=16）

**第 7 行：节点 + Worker**（y=48）
- Node CPU Utilization（x=0）、Worker Request Throughput（x=8）、Worker Data Transfer（x=16）

⭐ = 用于诊断 TTFT 瓶颈的关键指标

## 指标来源

### 前端指标（来自 Frontend Pod）
这些指标位于 `dynamo_frontend_*` 命名空间中，从 frontend 部署的 pod 收集。

| 面板 | 指标 | 公式 | 描述 |
|-------|--------|---------|-------------|
| **Frontend Requests / Sec** | `dynamo_frontend_requests_total` | `rate(...[30s])` | 打到 frontend 的每秒请求数，按 request_type 与 status 拆分 |
| **Frontend Avg Time to First Token** ⭐ | `dynamo_frontend_time_to_first_token_seconds_{sum,count}` | `1000 * (rate(sum[5m]) / rate(count[5m]))` | 过去 5 分钟内，从请求到达到首 token 的平均耗时（毫秒）。包含排队等待、prefill 计算与 NIXL 传输时间。**最核心的性能指标** |
| **Frontend Avg Request Duration** | `dynamo_frontend_request_duration_seconds_{sum,count}` | `1000 * (rate(sum[5m]) / rate(count[5m]))` | 过去 5 分钟内端到端请求总耗时（毫秒） |
| **Frontend Avg Inter-Token Latency** | `dynamo_frontend_inter_token_latency_seconds_{sum,count}` | `1000 * (rate(sum[5m]) / rate(count[5m]))` | 过去 5 分钟内 decode 阶段相邻 token 的平均间隔（毫秒） |
| **Frontend Avg Input/Output Sequence Length** | `dynamo_frontend_input_sequence_tokens_{sum,count}` 与 `dynamo_frontend_output_sequence_tokens_{sum,count}` | 各自 `rate(sum[5m]) / rate(count[5m])` | 过去 5 分钟的平均输入 prompt 长度（ISL）与输出生成长度（OSL），单位 token |
| **Frontend Queued Requests** ⭐⭐⭐ | `dynamo_frontend_queued_requests` | 原值 | 排队中的请求数。**诊断 worker 饱和度的关键指标。** 高值（>10）说明 worker 跟不上负载。黄色阈值 10、红色 50 |

### GPU 指标（来自 DCGM Exporter）
这些指标来自运行在 `gpu-operator` 命名空间内、以 DaemonSet 形式部署的 DCGM（Data Center GPU Manager）exporter。DCGM 收集硬件级 GPU 指标。

| 面板 | 指标 | 公式 | 描述 |
|-------|--------|---------|-------------|
| **GPU Compute Utilization** | `DCGM_FI_DEV_GPU_UTIL` | 原值 | 每张 GPU 的计算利用率百分比（0-100）。Prefill worker 在 prefill 阶段会出现高利用率 |
| **GPU Memory Bandwidth** | `DCGM_FI_DEV_MEM_COPY_UTIL` | 原值 | GPU memory copy 带宽利用率百分比（0-100）。**尖峰意味着 KV 缓存正在通过 NIXL 传输**。在单节点部署中，NIXL 使用 CUDA IPC（GPU→Host→Host→GPU），并不是直接 GPU-to-GPU。黄色阈值 60%，红色 80% |
| **NVLink Bandwidth (GB/s)** | `DCGM_FI_PROF_NVLINK_TX_BYTES` 与 `DCGM_FI_PROF_NVLINK_RX_BYTES` | `(rate(TX_BYTES[1m]) + rate(RX_BYTES[1m])) / 1e9` | 每张 GPU 的 NVLink 传输带宽（GB/s），通过 DCGM profiling 指标计算变化率。展示双向（TX + RX）总带宽。这包括 pod 内 TP 通信（prefill TP=2，decode TP=4）。低带宽（<1 GB/s）暗示跨 pod 的 NIXL KV 缓存传输可能正在使用 host memory 拷贝，而不是直接走 NVLink/GPUDirect。黄色阈值 5 GB/s，红色 10 GB/s |
| **GPU Memory Used** | `DCGM_FI_DEV_FB_USED` | `value / 1024` | 已使用的 GPU framebuffer 内存（GB）。Prefill worker 通过 NIXL 在 decode worker 上分配 KV block |

### Prefill Worker 指标（来自 Prefill Worker Pod）
这些指标来自 prefill worker pod 的系统 endpoint（端口 9090），追踪 prefill 操作的请求处理。

| 面板 | 指标 | 公式 | 描述 |
|-------|--------|---------|-------------|
| **Prefill Worker Processing Time** | `dynamo_component_request_duration_seconds_{sum,count,bucket}{dynamo_component="prefill",dynamo_endpoint="generate"}` | 平均：`1000 * rate(sum[5m]) / rate(count[5m])`；P99：`histogram_quantile(0.99, ...)` | 处理 prefill 请求的平均与 P99 耗时（毫秒）。**包含 prefill 计算与通过 NIXL 的 KV 缓存传输** |
| **Prefill Worker Throughput** | `dynamo_component_requests_total{dynamo_component="prefill",dynamo_endpoint="generate"}` | `rate(...[5m])` | 每秒被处理的 prefill 请求数 |

### Decode Worker 指标（来自 Decode Worker Pod）
这些指标来自 decode worker pod 的系统 endpoint（端口 9090）。在解耦（disaggregated）模式下，decode worker 接收来自 prefill worker 的 KV 缓存并执行 token 生成。

| 面板 | 指标 | 公式 | 描述 |
|-------|--------|---------|-------------|
| **Component Latency - Prefill vs Decode** | `dynamo_component_request_duration_seconds_{sum,count}{dynamo_component="prefill",dynamo_endpoint="generate"}` 与 `{dynamo_component="backend",dynamo_endpoint="generate"}` | `rate(sum[5m]) / rate(count[5m])` | 过去 5 分钟内，prefill worker（含 NIXL 传输）与 decode worker（覆盖整个 decode 会话所有输出 token）的平均请求耗时。**注意**：decode worker 延迟测量的是整段 decode 会话耗时，不仅仅是 TTFT。仅展示 `generate` endpoint（过滤掉 `clear_kv_blocks` 维护操作） |
| **Decode Worker - Request Throughput** | `dynamo_component_requests_total{dynamo_component="backend"}` | `rate(...[5m])` | decode worker 每秒处理的请求数 |
| **Decode Worker - Avg Request Duration** | `dynamo_component_request_duration_seconds_{sum,count}{dynamo_component="backend"}` | `rate(sum[5m]) / rate(count[5m])` | 过去 5 分钟内 decode worker 处理请求的平均耗时（仅 decode 阶段） |
| **KV Cache Utilization** | `dynamo_component_gpu_cache_usage_percent` | 原值（0-100%） | 处理活跃请求时用于 KV 缓存存储的 GPU 显存利用率。高值（>90%）意味着 worker 已满，请求正在排队。**注意**：仅 decode worker 提供，解耦模式下的 prefill worker 不暴露此指标。要观察 prefill 容量，请改看 Prefill Worker Processing Time |
| **KV Cache Blocks (Total)** | `dynamo_component_total_blocks` | 原值 | decode worker 上可用的 KV 缓存块总数。**注意**：仅 decode worker |

### CPU 指标（来自 cAdvisor 与 Node Exporter）
这些指标来自 Kubernetes cAdvisor（容器指标）与 Node Exporter（节点级指标）。CPU 瓶颈会影响 prefill/decode 性能。

| 面板 | 指标 | 公式 | 描述 |
|-------|--------|---------|-------------|
| **Worker CPU Usage** | `container_cpu_usage_seconds_total{namespace="robert",pod=~".*worker.*",container="main"}` | `rate(...[5m])` | worker pod 使用的 CPU 核数。值代表实际 CPU 消耗（如 2.5 = 2.5 核）。黄色 30 核，红色 50 核 |
| **Node CPU Utilization** | `node_cpu_seconds_total{mode="idle"}` | `100 - (avg(rate(idle)) * 100)` | 节点 CPU 总利用率百分比。展示所有核的总体 CPU 使用情况 |

### Worker 指标（来自 Worker Pod）
这些指标跟踪所有 worker pod（prefill 与 decode）上的请求处理。

| 面板 | 指标 | 公式 | 描述 |
|-------|--------|---------|-------------|
| **Worker Request Throughput** | `dynamo_component_requests_total{dynamo_endpoint="generate"}` | `rate(...[5m])` | 每个 worker 每秒处理的请求数，按组件类型（prefill、backend）拆分。展示整体系统吞吐 |
| **Worker Data Transfer** | `dynamo_component_request_bytes_total` 与 `dynamo_component_response_bytes_total` | `rate(...[5m])` | 请求（IN）与响应（OUT）的每秒字节数。展示 worker pod 上的数据吞吐 |

## 指标标签过滤

### 组件名过滤
- **Prefill worker**：`dynamo_component="prefill"`
- **Decode worker**：`dynamo_component="backend"`
- **所有 worker**：用 `dynamo_endpoint="generate"` 过滤，排除 `clear_kv_blocks` 等维护操作

### 重要标签
- `pod`：具体 pod 名（如 `llama3-70b-disagg-sn-0-vllmprefillworker-hrnt5`）
- `namespace`：Kubernetes 命名空间（如 `robert`）
- `dynamo_component`：组件类型（`prefill`、`backend`、`frontend`）
- `dynamo_endpoint`：endpoint 名（`generate`、`clear_kv_blocks`）
- `gpu`：GPU 索引（DCGM 指标中为 0-7）
- `Hostname`：节点主机名（DCGM 指标）

## 指标采集架构

```text
┌─────────────────┐
│  Frontend Pod   │ ──► dynamo_frontend_* metrics (HTTP port)
└─────────────────┘

┌─────────────────┐
│ Prefill Worker  │ ──► dynamo_component_* metrics (system port 9090)
│     Pods        │     ├─ dynamo_component_request_* (request stats)
└─────────────────┘     └─ dynamo_component_*_bytes_total (data transfer)
                        └─ container_cpu_* metrics (cAdvisor)

┌─────────────────┐
│ Decode Worker   │ ──► dynamo_component_* metrics (system port 9090)
│     Pods        │     └─ dynamo_component_request_* (component stats)
└─────────────────┘     └─ container_cpu_* metrics (cAdvisor)

┌─────────────────┐
│ DCGM Exporter   │ ──► DCGM_FI_DEV_* metrics (GPU metrics port)
│   DaemonSet     │     ├─ GPU compute utilization
│ (gpu-operator)  │     ├─ GPU memory bandwidth (NIXL indicator)
└─────────────────┘     ├─ GPU memory usage
                        └─ GPU temperature

┌─────────────────┐
│ Node Exporter   │ ──► node_* metrics (node metrics port)
│   DaemonSet     │     ├─ node_cpu_seconds_total (CPU by mode)
│   (monitoring)  │     ├─ node_load1/5/15 (load average)
└─────────────────┘     └─ node_memory_* (memory stats)

┌─────────────────┐
│    cAdvisor     │ ──► container_* metrics (built into kubelet)
│   (kubelet)     │     ├─ container_cpu_usage_seconds_total
└─────────────────┘     ├─ container_cpu_cfs_throttled_periods_total
                        └─ container_memory_*

           ▼
   ┌─────────────────┐
   │  Prometheus     │ ◄─── ServiceMonitor for DCGM & Node Exporter
   │  (monitoring)   │ ◄─── PodMonitor for Dynamo workers
   └─────────────────┘ ◄─── Scrapes cAdvisor from kubelet
           ▼
   ┌─────────────────┐
   │    Grafana      │
   │  (monitoring)   │
   └─────────────────┘
```

## PodMonitor 配置

Dynamo operator 会自动为开启了指标的部署创建 PodMonitor：
- **标签**：所有 worker pod 上加 `nvidia.com/metrics-enabled: "true"`
- **Endpoint**：系统端口（9090），路径 `/metrics`
- **命名空间**：可发现任意命名空间下的 pod（通过 `podMonitorNamespaceSelector={}`）

要让某个部署退出（opt-out）：
```yaml
apiVersion: nvidia.com/v1
kind: DynamoGraphDeployment
metadata:
  annotations:
    nvidia.com/enable-metrics: "false"
```

## DCGM ServiceMonitor 配置

DCGM ServiceMonitor 必须手动创建（参见 `dcgm-servicemonitor.yaml`）：
- **命名空间**：`gpu-operator`（DCGM exporter 所在）
- **标签**：`release: prometheus`（被 Prometheus 发现所必需）
- **选择器**：`app: nvidia-dcgm-exporter`
- **Endpoint**：`gpu-metrics` 端口，路径 `/metrics`

## 故障排查

### 看不到任何指标：
1. 确认 PodMonitor 存在：`kubectl get podmonitor -A`
2. 确认 pod 带有指标标签：`kubectl get pods -n <namespace> -l nvidia.com/metrics-enabled=true`
3. 检查 Prometheus targets：访问 Prometheus UI → Status → Targets

### DCGM 指标缺失：
1. 确认 DCGM exporter 在运行：`kubectl get daemonset -A | grep dcgm-exporter`
2. 确认 ServiceMonitor 存在：`kubectl get servicemonitor -n gpu-operator`
3. 检查 ServiceMonitor 上的 `release: prometheus` 标签

### Prefill 队列指标全为零：
- 这些指标只有在处理**远程 prefill** 请求时才会有值
- 在仅本地（local-only）模式下，decode worker 自己做 prefill（没有队列）
- 检查部署模式与请求路由配置

### KV Cache 指标只显示 decode worker：
**重要限制**：在解耦模式下，prefill worker（`--disaggregation-mode prefill`）**不会**暴露 `dynamo_component_total_blocks` 或 `dynamo_component_gpu_cache_usage_percent`。只有 decode worker 暴露这些指标。

**为什么会这样：**
- prefill worker 通过 NIXL 把 KV 缓存传给 decode worker
- 它们不维护长期的 KV 缓存状态
- 只有 decode worker 跟踪 KV 缓存利用率指标

**如何诊断 prefill worker 容量瓶颈：**
1. **检查 worker 日志**中启动时打印的 KV 缓存大小：
   ```bash
   kubectl logs -n <namespace> <prefill-worker-pod> | grep "GPU KV cache size"
   # Example output: GPU KV cache size: 254,336 tokens
   ```

2. **计算最大并发**：
   ```text
   Max Concurrency = KV Cache Size ÷ Tokens Per Request
   # For ISL=8192: 254,336 ÷ 131,072 = 1.94 requests per prefill worker
   ```

3. **观察仪表板上的间接指标**：
   - **Prefill Worker Processing Time**：高均值（>5s）或 P99（>10s）意味着饱和
   - **Frontend Avg TTFT**：若远高于 Prefill Processing Time，说明请求在排队
   - **Gap = TTFT - Prefill Processing Time** = 排队等待时间

4. **prefill KV 缓存瓶颈的性能特征**：
   - 最小 TTFT 很低（1-3s）—— 证明系统**能**很快
   - 平均/最大 TTFT 非常高（>30s）—— 证明请求在排队
   - 方差很大（Max ÷ Min > 20×）—— 排队行为的特征
   - 如果是计算受限的问题不可能出现这种方差模式 —— 这就是 KV 缓存容量瓶颈的数学特征

### CPU 占用过高：
1. **Worker CPU Usage 显示很高（>30 核）**：
   - 检查 worker 是否配置了足够的 CPU 限额
   - 可能意味着 CPU 受限的操作（tokenize、调度）
   - 与 GPU 利用率对比 —— 不应该让 CPU 成为瓶颈

2. **多少算正常？**：
   - vLLM worker 的 CPU 用于：
     - 请求调度与 batching
     - tokenize（输入/输出处理）
     - KV 缓存管理
     - TCP/gRPC 通信（request plane）
   - 期望中等水平的 CPU 占用（每个 worker 5-20 核）
   - GPU 计算应当占主导，而不是 CPU

## 仪表板变量

仪表板使用两个模板变量以提高灵活性：

### Datasource 变量
- **变量**：`${datasource}`
- **类型**：`datasource`
- **查询**：`prometheus`
- **自动选择**：默认 Prometheus 实例
- **目的**：让仪表板自动连接到你的 Prometheus 实例，无需硬编码 UID

### Namespace 变量
- **变量**：`${namespace}`
- **类型**：`query`
- **查询**：`label_values(dynamo_frontend_requests_total, namespace)`
- **目的**：按 Kubernetes 命名空间过滤指标（例如 "robert"、"default"）
- **自动填充**：从 frontend pod 动态发现命名空间

**用法**：所有仪表板查询都通过 `namespace="$namespace"` 过滤，从而只显示所选部署的指标。可以通过仪表板顶部的命名空间下拉框，在不同命名空间下的不同 Dynamo 部署之间切换。

---

## Intel XPU-SMI 指标（来自 XPU-SMI Exporter）

这些指标来自 Intel XPU-SMI Prometheus exporter（`xpu-smi-exporter` 任务）。XPU-SMI 收集硬件级 Intel GPU 指标，对应 NVIDIA 的 DCGM。

### 安装

先在主机上启动 XPU-SMI Prometheus exporter，然后在可观测性（observability）栈上叠加 XPU overlay 启动：
```bash
# Install Intel XPU-SMI (xpumanager): https://github.com/intel/xpumanager
# Start the exporter (serves Prometheus metrics on port 9966)
python deploy/observability/xpu_smi_exporter.py --port 9966 &

# Start base services
docker compose -f deploy/docker-compose.yml up -d

# Start observability with XPU overlay (uses prometheus-xpu.yml + xpu-alert-rules.yml)
docker compose -f deploy/docker-observability.yml -f deploy/docker-observability-xpu.yml up -d
```

### XPU-SMI 仪表板面板（`xpu-smi-metrics.json`）

| 面板 | 指标 | 公式 | 描述 |
|-------|--------|---------|-------------|
| **XPU Compute Utilization** | `xpu_engine_group_compute_engine_util` | 原值（0-100%） | 每个 XPU 设备的计算引擎利用率。等同于 `DCGM_FI_DEV_GPU_UTIL` |
| **XPU Memory Usage** | `xpu_memory_used_bytes`、`xpu_memory_free_bytes` | 原值 | 每个 XPU 设备的 HBM/VRAM 已用与空闲字节数。等同于 `DCGM_FI_DEV_FB_USED/FB_FREE` |
| **XPU Temperature** | `xpu_temperature_celsius` | 原值（°C） | GPU die 与显存温度。标签：`location="gpu"` 或 `location="memory"`。阈值：黄色@70°C，红色@85°C |
| **XPU Power Usage** | `xpu_power_watts` | 原值（W） | 每个 XPU 设备的瞬时功耗。等同于 `DCGM_FI_DEV_POWER_USAGE` |
| **XPU Engine Utilization** | `xpu_engine_group_compute_engine_util`、`xpu_engine_group_copy_engine_util`、`xpu_engine_group_render_engine_util` | 原值（0-100%） | 按 engine 组拆分的利用率 |
| **XPU Memory Bandwidth** | `xpu_memory_read_bytes_per_second`、`xpu_memory_write_bytes_per_second` | 原值（bytes/sec） | HBM 读写带宽（字节/秒） |
| **XPU PCIe Bandwidth** | `xpu_pcie_read_bytes_per_second`、`xpu_pcie_write_bytes_per_second` | 原值（bytes/sec） | PCIe 读写带宽。等同于 `DCGM_FI_PROF_PCIE_RX/TX_BYTES` |
| **Avg XPU Utilization** | `xpu_engine_group_compute_engine_util` | `avg(...)` | 所有 XPU 设备的平均利用率仪表 |
| **Max XPU Temperature** | `xpu_temperature_celsius{location="gpu"}` | `max(...)` | 所有 XPU 设备中的最高温度仪表 |

### XPU 与 NVIDIA DCGM 指标映射

| NVIDIA DCGM 指标 | Intel XPU-SMI 指标 | 描述 |
|---|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | `xpu_engine_group_compute_engine_util` | 计算利用率 % |
| `DCGM_FI_DEV_FB_USED` | `xpu_memory_used_bytes` | 已用显存 |
| `DCGM_FI_DEV_FB_FREE` | `xpu_memory_free_bytes` | 空闲显存 |
| `DCGM_FI_DEV_GPU_TEMP` | `xpu_temperature_celsius{location="gpu"}` | GPU 温度 |
| `DCGM_FI_DEV_MEMORY_TEMP` | `xpu_temperature_celsius{location="memory"}` | 显存温度 |
| `DCGM_FI_DEV_POWER_USAGE` | `xpu_power_watts` | 功耗（W） |
| `DCGM_FI_PROF_PCIE_RX_BYTES` | `xpu_pcie_read_bytes_per_second` | PCIe RX 字节/秒 |
| `DCGM_FI_PROF_PCIE_TX_BYTES` | `xpu_pcie_write_bytes_per_second` | PCIe TX 字节/秒 |

### 指标架构（XPU）

```text
┌─────────────────┐
│  Intel XPU-SMI  │ ──► xpu_* metrics (Prometheus port :9966)
│    Exporter     │     ├─ xpu_engine_group_compute_engine_util
│  (host process) │     ├─ xpu_memory_used_bytes / free_bytes
└─────────────────┘     ├─ xpu_temperature_celsius
                        ├─ xpu_power_watts
                        └─ xpu_pcie_*_bytes_total

           ▼
   ┌─────────────────┐
   │  Prometheus     │ ◄─── scrape job: xpu-smi-exporter (port 9966)
   │  (monitoring)   │
   └─────────────────┘
           ▼
   ┌─────────────────┐
   │    Grafana      │ ◄─── xpu-smi-metrics.json dashboard
   │  (monitoring)   │
   └─────────────────┘
```

### XPU 告警规则（`xpu-alert-rules.yml`）

| 告警 | 条件 | 严重级别 | 描述 |
|-------|-----------|----------|-------------|
| `XPUHighTemperature` | temp > 85°C 持续 2m | warning | XPU GPU die 过热 |
| `XPUCriticalTemperature` | temp > 95°C 持续 30s | critical | 即将触发热降频/关机 |
| `XPUMemoryAlmostFull` | mem > 90% 持续 1m | warning | KV 缓存分配可能失败 |
| `XPUMemoryCritical` | mem > 98% 持续 30s | critical | 即将 OOM |
| `XPUHighPowerDraw` | power > 400W 持续 5m | warning | 持续高功耗 |
| `XPUExporterDown` | `up{job="xpu-smi-exporter"} == 0` 持续 1m | critical | 监控盲区 |
| `XPULowComputeUtilizationDuringLoad` | 在有流量（`rate()` > 0）期间 util < 10% | warning | 可能存在派发问题 |
| `XPUWorkerLivenessLost` | 没有 XPU 指标 + 有流量（`rate()` > 0） | critical | 怀疑 XPU worker 崩溃 |

### 排查 XPU 指标

#### Prometheus 中看不到 XPU 指标：
1. 确认 XPU-SMI exporter 在运行：`curl http://localhost:9966/metrics | grep xpu_`
2. 确认 Intel GPU 可见：`xpu-smi discovery`
3. 在 Prometheus UI 中确认 scrape 任务：Status → Targets → `xpu-smi-exporter`
4. 检查防火墙：`sudo ufw allow 9966/tcp`

#### XPU 设备未被识别：
```bash
# Check device visibility in container
ls /dev/dri/
# Should show renderD128, card0, etc.

# Verify XPU-SMI can see the device
xpu-smi discovery
# Expected: lists Intel GPU devices with model name, driver version
```
