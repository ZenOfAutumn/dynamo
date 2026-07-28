---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Dynamo 基准测试
subtitle: 在不同 Dynamo 部署配置之间进行基准测试与性能对比
---

本指南介绍如何使用 [AIPerf](https://github.com/ai-dynamo/aiperf) 对 Dynamo 部署进行基准测试。AIPerf 是一款用于测量生成式 AI 推理性能的综合工具，提供详细指标、实时仪表盘以及自动可视化 —— 你可直接对端点调用它。

你可以对以下任意组合进行基准测试：
- **DynamoGraphDeployment**
- **外部 HTTP 端点**（vLLM、llm-d、AIBrix 等）

## 选择你的基准测试方式

**客户端侧**通过端口转发在本地机器运行基准测试。**服务端侧**直接在 Kubernetes 集群中、通过内部服务 URL 运行基准测试。

**简言之：**
需要高性能/负载测试？选服务端侧。
只需快速测试/对比？选客户端侧。

### 何时使用客户端侧基准测试：
- 你想快速测试部署
- 你希望立即在本地机器看到结果
- 你在比较外部服务或部署（不一定限于 Dynamo 部署）
- 你需要从笔记本/工作站运行基准测试

→ **[前往 客户端侧基准测试（本地）](#客户端侧基准测试-本地)**

### 何时使用服务端侧基准测试：
- 你拥有具备 kubectl 访问权限的开发环境
- 你在做要求高负载/高速度的性能验证
- 你在客户端侧基准测试中遇到超时或性能问题
- 你想要最佳网络性能（无端口转发开销）
- 你在运行自动化 CI/CD 流水线
- 你需要隔离的执行环境
- 你希望在集群中持久化保存结果

→ **[前往 服务端侧基准测试（集群内）](#服务端侧基准测试-集群内)**

### 快速对比

| 特性 | 客户端侧 | 服务端侧 |
|---------|-------------|-------------|
| **位置** | 本地机器 | Kubernetes 集群 |
| **网络** | 需要端口转发 | 直接服务 DNS |
| **搭建** | 快速简单 | 需要集群资源 |
| **性能** | 受本地资源限制，高负载下可能超时 | 集群最佳性能，可承受高负载 |
| **隔离性** | 共享环境 | 隔离的 Job 执行 |
| **结果** | 本地文件系统 | 持久化卷 |
| **最适用** | 轻负载 | 高负载 |

## AIPerf 概览

[AIPerf](https://github.com/ai-dynamo/aiperf) 是一个独立基准测试工具，发布在 [PyPI](https://pypi.org/project/aiperf/)。它已预装在 Dynamo 容器镜像中。关键特性：

- 测量延迟、吞吐量、TTFT、token 间延迟（inter-token latency）等
- 多种负载模式：并发数、请求速率、trace 回放
- 通过 `aiperf plot` 自动可视化（Pareto 曲线、时间序列、GPU 遥测）
- 支持实时探索的交互式仪表盘模式
- 到达模式（Poisson、constant、gamma）模拟真实流量
- 预热阶段、渐进 ramp-up、多 URL 负载均衡

**重要**：`--model` 参数必须与端点上部署的模型一致。

完整文档参见 [AIPerf 文档](https://github.com/ai-dynamo/aiperf/tree/main/docs)。

---

# 客户端侧基准测试（本地）

客户端侧基准测试在本地机器运行，并通过端口转发连接到 Kubernetes 部署。

## 前置条件

1. **Dynamo 容器环境** - 你必须运行在已预装 AIPerf 的 Dynamo 容器中，或在本地安装：
   ```bash
   pip install aiperf
   ```

2. **HTTP 端点** - 确保你有可基准测试的 HTTP 端点，可以是：
   - 通过 HTTP 端点暴露的 DynamoGraphDeployment
   - 外部服务（vLLM、llm-d、AIBrix 等）
   - 任何提供 OpenAI 兼容模型的 HTTP 端点

## 用户工作流

### 步骤 1：搭建集群并部署

按 [安装指南](../kubernetes/installation-guide.md) 搭建带 NVIDIA GPU 的 Kubernetes 集群并安装 Dynamo Kubernetes Platform。然后按 [部署文档](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends) 部署你的 DynamoGraphDeployment。

### 步骤 2：端口转发并运行单次基准测试

> **等待模型就绪。** 在基准测试前，确保你的部署已完全加载模型。检查 pod 日志或访问健康端点（`curl http://localhost:8000/health`） —— 应返回 `200 OK` 后再继续。

```bash
# 端口转发前端服务
kubectl port-forward -n <namespace> svc/<frontend-service-name> 8000:8000 > /dev/null 2>&1 &

# 运行单次基准测试
aiperf profile \
    --model <your-model-name> \
    --url http://localhost:8000 \
    --endpoint-type chat \
    --streaming \
    --concurrency 10 \
    --request-count 100 \
    --synthetic-input-tokens-mean 2000 \
    --output-tokens-mean 256
```

这会在 `artifacts/` 中产出结果，并向控制台打印汇总表：

```text
                                NVIDIA AIPerf | LLM Metrics
┏━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┓
┃              Metric ┃     avg ┃     min ┃     max ┃     p99 ┃     p90 ┃     p50 ┃     std ┃
┡━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━┩
│ Time to First Token │  234.56 │  189.23 │  298.45 │  289.34 │  267.12 │  231.12 │   28.45 │
│                (ms) │         │         │         │         │         │         │         │
│     Request Latency │ 1234.56 │  987.34 │ 1567.89 │ 1534.23 │ 1456.78 │ 1223.45 │  156.78 │
│                (ms) │         │         │         │         │         │         │         │
│ Inter Token Latency │   15.67 │   12.34 │   19.45 │   19.01 │   18.23 │   15.45 │    1.89 │
│                (ms) │         │         │         │         │         │         │         │
│  Request Throughput │   31.45 │     N/A │     N/A │     N/A │     N/A │     N/A │     N/A │
│      (requests/sec) │         │         │         │         │         │         │         │
└─────────────────────┴─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
```

*实际数字会随模型大小、硬件、batch size 与网络条件而异。客户端侧基准测试包含端口转发开销 —— 精确性能测量请使用 [服务端侧基准测试](#服务端侧基准测试-集群内)。*

完成后停止端口转发：`kill %1`（或 `kill <PID>`）。

### 步骤 3：用于 Pareto 分析的并发扫描

要理解部署在不同负载下的行为，运行并发扫描。每个并发级别发送足够的请求以获得稳定测量（`max(c*3, 10)`）：

```bash
MODEL="<your-model-name>"
URL="http://localhost:8000"

for c in 1 2 5 10 50 100; do
    aiperf profile \
        --model "$MODEL" \
        --url "$URL" \
        --endpoint-type chat \
        --streaming \
        --concurrency $c \
        --request-count $(( c * 3 > 10 ? c * 3 : 10 )) \
        --synthetic-input-tokens-mean 2000 \
        --output-tokens-mean 256 \
        --artifact-dir "artifacts/deployment-a/c$c"
done
```

**注意**：调整并发级别以匹配部署容量。在小规模部署上设置过高并发（例如单 GPU 上 c250）会导致服务器错误。从较低值起步并逐步增加直到饱和点。

### 步骤 4：[若需对比] 对第二个部署做基准测试

拆除部署 A，部署具有不同配置的部署 B。杀掉之前的端口转发（`kill %1`），然后重复：

```bash
kubectl port-forward -n <namespace> svc/<frontend-service-b> 8000:8000 > /dev/null 2>&1 &

for c in 1 2 5 10 50 100; do
    aiperf profile \
        --model "$MODEL" \
        --url "$URL" \
        --endpoint-type chat \
        --streaming \
        --concurrency $c \
        --request-count $(( c * 3 > 10 ? c * 3 : 10 )) \
        --synthetic-input-tokens-mean 2000 \
        --output-tokens-mean 256 \
        --artifact-dir "artifacts/deployment-b/c$c"
done
```

### 步骤 5：生成可视化

```bash
# 比较所有运行 —— 自动检测多 run 目录
aiperf plot artifacts/deployment-a artifacts/deployment-b

# 或比较父目录下所有子目录
aiperf plot artifacts/

# 启动交互式仪表盘进行探索
aiperf plot artifacts/ --dashboard
```

AIPerf 会基于可用数据自动生成图表：
- **TTFT vs 吞吐量** —— 在响应速度与容量之间寻找甜蜜点（多 run 比较时始终生成）
- **Pareto 曲线** —— 每 GPU 吞吐量与延迟、交互性的权衡（仅当有 GPU 遥测数据时生成 —— 在 profile 时加 `--gpu-telemetry`，前提是 DCGM 在运行）
- **时间序列** —— 每请求的 TTFT、ITL、延迟随时间变化（用于单 run 分析）

下图是 8x H200 上 vLLM 服务 Qwen3-0.6B 的并发扫描得到的 Pareto 前沿示例，展示了用户体验（每用户 tokens/sec）与资源效率（每 GPU tokens/sec）之间的权衡：

![AIPerf Pareto Frontier](../assets/img/aiperf-pareto-frontier.png)

完整的图表自定义、实验分类与主题详情参见
[AIPerf 可视化指南](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/plot.md)。

## 用例

- **比较 DynamoGraphDeployment**（如聚合 vs 解耦配置）
- **比较不同后端**（如 SGLang vs TensorRT-LLM vs vLLM）
- **比较 Dynamo 与其他平台**（如 Dynamo vs llm-d vs AIBrix）
- **比较不同模型**（如 Llama-3-8B vs Llama-3-70B vs Qwen-3-0.6B）
- **比较不同硬件配置**（如 H100 vs A100 vs H200）
- **比较不同并行策略**（如不同 GPU 数或显存配置）

## AIPerf 快速参考

### 常用选项

```text
aiperf profile [OPTIONS]

REQUIRED:
  --model MODEL               Model name (must match the deployed model)
  --url URL                   Endpoint URL (e.g., http://localhost:8000)

COMMON OPTIONS:
  --endpoint-type TYPE        Endpoint type: chat, completions, embeddings (default: chat)
  --streaming                 Enable streaming responses
  --concurrency N             Number of concurrent requests
  --request-rate N            Target requests per second (alternative to --concurrency)
  --request-count N           Total number of requests to send
  --benchmark-duration N      Run for N seconds instead of a fixed request count
  --synthetic-input-tokens-mean N   Average input sequence length in tokens
  --output-tokens-mean N      Average output sequence length in tokens
  --artifact-dir DIR          Output directory for results (default: artifacts/)
  --warmup-request-count N    Warmup requests before measurement
  --ui TYPE                   UI mode: dashboard, simple, none (default: dashboard)
```

完整 CLI 参考见 `aiperf profile --help` 或 [CLI 文档](https://github.com/ai-dynamo/aiperf/blob/main/docs/cli-options.md)。

### 输出序列长度

如需强制特定输出长度，通过 `--extra-inputs` 传入 `ignore_eos` 与 `min_tokens`：

```bash
aiperf profile \
    --model <model> \
    --url http://localhost:8000 \
    --endpoint-type chat \
    --streaming \
    --concurrency 10 \
    --output-tokens-mean 256 \
    --extra-inputs max_tokens:256 \
    --extra-inputs min_tokens:256 \
    --extra-inputs ignore_eos:true
```

### 理解结果

每次 `aiperf profile` 运行会产出一个 artifact 目录，包含：
- **`profile_export_aiperf.json`** —— 结构化指标（延迟、吞吐量、TTFT、ITL 等）
- **`profile_export.jsonl`** —— 每请求原始数据
- **`profile_export_aiperf.csv`** —— CSV 格式指标

结果按你指定的 `--artifact-dir` 组织。并发扫描的常见模式是：

```text
artifacts/
├── deployment-a/
│   ├── c1/
│   │   ├── profile_export_aiperf.json
│   │   └── profile_export.jsonl
│   ├── c10/
│   ├── c50/
│   └── c100/
├── deployment-b/
│   ├── c1/
│   ├── c10/
│   ├── c50/
│   └── c100/
└── plots/                    # 由 aiperf plot 生成
    ├── ttft_vs_throughput.png
    ├── pareto_curve_throughput_per_gpu_vs_latency.png      # 当有 GPU 遥测时
    └── pareto_curve_throughput_per_gpu_vs_interactivity.png # 当有 GPU 遥测时
```

---

# 服务端侧基准测试（集群内）

服务端侧基准测试直接在 Kubernetes 集群内运行，消除端口转发开销，并支持高负载测试。

## 前置条件

1. **带 NVIDIA GPU 的 Kubernetes 集群**，并已搭建 Dynamo 命名空间（参见 [Dynamo Kubernetes Platform 文档](../kubernetes/README.md)）
2. **存储**：配置好相应权限的 PersistentVolumeClaim（参见 [deploy/utils README](https://github.com/ai-dynamo/dynamo/blob/main/deploy/utils/README.md)）
3. **包含 AIPerf 的 Docker 镜像**（Dynamo 运行时镜像已包含）

## 快速开始

### 步骤 1：部署你的 DynamoGraphDeployment
按 [部署文档](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends) 部署。确保前端服务已暴露且模型已完全加载，再运行基准测试 —— 检查 pod 日志或确认健康端点返回 `200 OK`。

### 步骤 2：配置并运行基准测试 Job

首先编辑 `benchmarks/incluster/benchmark_job.yaml` 以匹配你的部署：

- **模型名**：更新 `MODEL` 变量
- **服务 URL**：更新 `URL` 变量（跨命名空间访问使用 `<svc_name>.<namespace>.svc.cluster.local:port`）
- **并发级别**：调整 `for c in ...` 循环
- **Docker 镜像**：必要时更新 `image` 字段

然后部署：

```bash
export NAMESPACE=benchmarking

# 部署基准测试 Job
kubectl apply -f benchmarks/incluster/benchmark_job.yaml -n $NAMESPACE

# 监控 Job
kubectl logs -f job/dynamo-benchmark -n $NAMESPACE
```

### 步骤 3：取回结果
```bash
# 创建访问 Pod（如已运行可跳过）
kubectl apply -f deploy/utils/manifests/pvc-access-pod.yaml -n $NAMESPACE
kubectl wait --for=condition=Ready pod/pvc-access-pod -n $NAMESPACE --timeout=60s

# 下载结果
kubectl cp $NAMESPACE/pvc-access-pod:/data/results ./results

# 清理
kubectl delete pod pvc-access-pod -n $NAMESPACE
```

### 步骤 4：生成图表
```bash
aiperf plot ./results
```

## 跨命名空间服务访问

引用其他命名空间中的服务时，使用完整的 Kubernetes DNS：

```bash
# 同命名空间
--url http://vllm-agg-frontend:8000

# 不同命名空间
--url http://vllm-agg-frontend.production.svc.cluster.local:8000
```

## 监控与调试

```bash
# 查看 Job 状态
kubectl describe job dynamo-benchmark -n $NAMESPACE

# 跟踪日志
kubectl logs -f job/dynamo-benchmark -n $NAMESPACE

# 查看 Pod 状态
kubectl get pods -n $NAMESPACE -l job-name=dynamo-benchmark

# 调试失败 Pod
kubectl describe pod <pod-name> -n $NAMESPACE
```

### 故障排查

1. **服务未找到**：确保 DynamoGraphDeployment 前端服务已运行
2. **PVC 访问**：检查 `dynamo-pvc` 配置正确且可访问
3. **镜像拉取问题**：确保 Docker 镜像可被集群访问
4. **资源约束**：若 Job 被驱逐，调整资源上限

```bash
# 检查 PVC 状态
kubectl get pvc dynamo-pvc -n $NAMESPACE

# 校验服务存在并有 endpoints
kubectl get svc -n $NAMESPACE
kubectl get endpoints <service-name> -n $NAMESPACE
```

---

## 使用 Mocker 后端进行测试

为开发和测试目的，Dynamo 提供了一个 [mocker 后端](https://github.com/ai-dynamo/dynamo/blob/main/components/src/dynamo/mocker)，它在不需要真实 GPU 资源的情况下模拟 LLM 推理。该后端适用于：

- **测试部署**而不需昂贵的 GPU 基础设施
- **开发与调试**路由、planner 或前端逻辑
- 需要在不执行模型的情况下验证基础设施的 **CI/CD 流水线**
- **基准测试框架验证**，确保在使用真实后端前你的搭建可用

mocker 后端模仿真实后端（SGLang、TensorRT-LLM、vLLM）的 API 与行为，但生成 mock 响应而非运行真实推理。

用法示例与配置选项见
[mocker 目录](https://github.com/ai-dynamo/dynamo/blob/main/components/src/dynamo/mocker)。

---

## AIPerf 进阶特性

AIPerf 在基础 profiling 之外还有许多能力。下面是一些对 Dynamo 基准测试尤为有用的功能：

| 特性 | 说明 | 文档 |
|---------|-------------|------|
| Trace Replay | 回放生产 trace 以进行确定性基准测试 | [Trace Replay](https://github.com/ai-dynamo/aiperf/blob/main/docs/benchmark-modes/trace-replay.md) |
| 到达模式 | Poisson、constant、gamma 流量分布 | [Arrival Patterns](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/arrival-patterns.md) |
| 渐进 Ramp | 平滑提升并发数与请求速率 | [Ramping](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/ramping.md) |
| 预热阶段 | 排除冷启动对测量的影响 | [Warmup](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/warmup.md) |
| 多 URL 负载均衡 | 在多个端点之间分发请求 | [Multi-URL](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/multi-url-load-balancing.md) |
| GPU 遥测 | 在基准测试期间收集 DCGM 指标 | [GPU Telemetry](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/gpu-telemetry.md) |
| Goodput 分析 | 基于 SLO 的吞吐量测量 | [Goodput](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/goodput.md) |
| 时间切片分析 | 按时间切片的性能拆解 | [Timeslices](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/timeslices.md) |
| 多轮对话 | 对多轮 chat 工作负载做基准测试 | [Multi-Turn](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/multi-turn.md) |
| 实验分类 | 在图表中以基线 vs 处理组语义着色 | [Plotting](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/plot.md) |
