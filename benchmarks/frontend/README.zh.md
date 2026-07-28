<!-- SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# Frontend 性能基准测试套件

一个用于度量 Dynamo 前端（frontend）模型服务性能的可配置 sweep runner。它驱动 [aiperf](https://github.com/ai-dynamo/aiperf) 对前端/mocker（或前端/vLLM）栈施加负载，并在一组参数网格上收集吞吐、延迟以及可观测性（observability）数据。

主要用例是 **HuggingFace tokenizer 与 fastokens 的对比**——在并发等级、输入序列长度（ISL）以及 worker 数量上进行 sweep，以量化 tokenizer 对端到端性能的影响。

---

## 架构

代码库采用三层设计，将纯逻辑与执行、基础设施关注点分离。

| 层 | Package | 职责 |
|-------|---------|----------------|
| **Core** | `scripts/sweep_core/` | 纯数据模型、计划构造、产物写出、报告。无 subprocess 或 kubectl 调用。 |
| **Executors** | `scripts/sweep_executors/` | `SweepExecutor` 协议的两种实现——`LocalExecutor`（委托给 `run_perf.sh`）与 `K8sDgdExecutor`（基于 DynamoGraphDeployment 的 K8s 运行）。 |
| **K8s helpers** | `scripts/sweep_k8s/` | kubectl 包装器、DGD patch、模板渲染、aiperf Job 启动以及 Prometheus 指标采集。 |

入口为 `scripts/sweep_runner.py`，一个轻量 CLI，将三层串联：根据 CLI 参数构造 `SweepPlan`、按 `--mode` 选择 executor，并把计划交给编排器。

**数据流：**

```
CLI args --> SweepConfig --> SweepPlan (Cartesian grid of RunSpecs)
                                |
                          Orchestrator
                                |
                   LocalExecutor  or  K8sDgdExecutor
                        |                   |
                  run_perf.sh        DGD + aiperf Job
                        |                   |
                  artifacts/           artifacts/
```

---

## 快速开始 —— 本地

本地模式在当前机器上启动 mocker 后端与前端进程，运行 aiperf 测试，并在多次运行之间将其拆除。

**前置条件：**

- 已安装 `dynamo.mocker` 与 `dynamo.frontend`（来自 Dynamo 仓库）
- 已安装 `aiperf` 并在 `$PATH` 中
- 本地可访问 HuggingFace 模型（默认：`Qwen/Qwen3-0.6B`）

**冒烟测试（2 次运行，每次约 30 秒）：**

```bash
cd benchmarks/frontend/scripts

python3 sweep_runner.py \
    --tokenizers hf,fastokens \
    --concurrency 32 \
    --isl 512 \
    --benchmark-duration 30 \
    --speedup-ratio 1000000
```

**完整本地 sweep：**

```bash
python3 sweep_runner.py \
    --tokenizers hf,fastokens \
    --concurrency 32,64,128 \
    --isl 512,1024,2048
```

**传输饱和 sweep（高并发，变化 worker 数）：**

```bash
python3 sweep_runner.py \
    --tokenizers hf \
    --concurrency 4096 \
    --num-requests 16384,32768 \
    --workers 1,2,4,8 \
    --speedup-ratio 1000000
```

结果写入 `artifacts/sweep_<timestamp>/`。

---

## 快速开始 —— Kubernetes

K8s 模式在 Kubernetes 命名空间中部署一个 DynamoGraphDeployment（DGD），并以集群内 Job 形式启动 aiperf，目标为前端服务端点。

### 前置条件

1. **命名空间** —— 用于基准测试的专用命名空间（默认：`dynamo-bench`）。
2. **HuggingFace token secret** —— 名为 `hf-token-secret` 的 Kubernetes Secret，
   含你的 HF token（如模型需要鉴权）。
3. **模型缓存 PVC** —— 用于缓存模型权重的 PersistentVolumeClaim（避免多次运行
   重复下载）。
4. **DGD 已部署** —— 你可以预先自行部署 DGD，或使用
   `--deploy --deploy-template` 参数让 sweep runner 创建。
5. **kubectl** 已配置好对目标集群与命名空间的访问。

### 示例：mocker 后端

```bash
python3 sweep_runner.py \
    --mode k8s \
    --dgd-name dynamo-bench-mocker \
    --tokenizers hf,fastokens \
    --concurrency 50,100 \
    --isl 512
```

### 示例：基于模板的部署

提供 `--deploy-template` 时，runner 会用每次运行的变量（tokenizer、workers、model 等）渲染模板，并在每个运行组之前通过 kubectl 应用：

```bash
python3 sweep_runner.py \
    --mode k8s \
    --deploy \
    --deploy-template dgd/templates/mocker.yaml \
    --dgd-name dynamo-bench-mocker \
    --image nvcr.io/.../image:tag \
    --tokenizers hf,fastokens \
    --concurrency 50,100 \
    --isl 512
```

### aiperf 在集群中如何运行

sweep runner 在 DGD 同一命名空间下创建一个短生命周期的 Kubernetes Job。Job pod 通过前端的集群内服务 DNS 名（如 `dynamo-bench-mocker-frontend:8000`）运行 `aiperf`。Job 完成后，产物通过 `kubectl cp` 拷贝回本地。

### 重置策略

运行之间，`--reset-strategy` 参数控制部署栈如何回收：

| 策略 | 行为 |
|----------|----------|
| `none` | 不重置；在同一部署上连续运行。 |
| `frontend` | 仅在两次运行之间重启前端 pod。 |
| `graph`（默认） | 每个运行组之间重新部署整个 DGD 图。 |

---

## CLI 参考

`sweep_runner.py` 的所有参数：

### 通用选项

| 参数 | 默认 | 说明 |
|------|---------|-------------|
| `--mode` | `local` | 执行模式：`local` 或 `k8s`。 |
| `--backend` | `mocker` | 引擎后端：`mocker`（合成）或 `vllm`（真实推理）。 |
| `--model` | `Qwen/Qwen3-0.6B` | HuggingFace 模型路径。 |
| `--model-name` | 同 `--model` | 服务的模型名（多模型部署时使用）。 |
| `--tokenizers` | `hf,fastokens` | 逗号分隔的 tokenizer 后端。 |
| `--concurrency` | `50,100,200` | 逗号分隔的并发等级。 |
| `--isl` | `512,1024,2048` | 逗号分隔的输入序列长度。 |
| `--osl` | `256` | 输出序列长度。 |
| `--workers` | `2` | 每个模型的 worker 数（逗号分隔）。 |
| `--num-models` | `1` | 模型实例数。 |
| `--speedup-ratio` | `1.0` | mocker 加速分母；使用大值（如 1000000）可获得近乎瞬时的 mocker。 |
| `--benchmark-duration` | `60` | aiperf 运行时长（秒）。 |
| `--num-requests` | 无 | 逗号分隔的请求数（覆盖 `--benchmark-duration`）。 |
| `--rps` | 无 | 逗号分隔的目标请求速率（req/s）。 |
| `--output-dir` | 自动加时间戳 | 输出目录。 |
| `--cooldown` | `3` | 运行之间冷却秒数。 |
| `--max-consecutive-fails` | `2` | 连续失败 N 次后中止 sweep。 |
| `--isolation` | `fresh_per_run` | 隔离策略：`fresh_per_run` 或 `reuse_by_deploy_key`。 |
| `--no-report` | 关闭 | 跳过生成单次运行报告。 |

### 执行控制

| 参数 | 说明 |
|------|-------------|
| `--dry-run` | 仅打印 sweep 计划，不执行任何运行。 |
| `--emit-plan` | 以 JSON 形式输出 sweep 计划并退出（适用于 Argo 或 MCP 集成）。 |

### K8s 模式选项

| 参数 | 默认 | 说明 |
|------|---------|-------------|
| `--namespace` | `dynamo-bench` | Kubernetes 命名空间。 |
| `--endpoint` | 自动派生 | 前端端点（`host:port`）。 |
| `--dgd-name` | 无 | DynamoGraphDeployment 名称。 |
| `--image` | 无 | k8s 部署使用的容器镜像。 |
| `--deploy-template` | 无 | DGD YAML 模板路径（启用基于模板的部署）。 |
| `--deploy` | 关闭 | 在 sweep 之前部署基础设施。 |
| `--reset-strategy` | `graph` | 单次运行重置：`none`、`frontend` 或 `graph`。 |
| `--frontend-port` | `8000` | 前端 HTTP 端口。 |
| `--worker-replicas` | `1` | worker pod 副本数。 |
| `--request-plane` | `tcp` | 请求平面传输。 |
| `--event-plane` | `nats` | 事件平面传输。 |
| `--router-mode` | `round-robin` | 前端 router 模式。 |
| `--hf-token` | 无 | k8s 用 HuggingFace token。 |
| `--image-pull-secret` | 无 | image pull secret 名称。 |
| `--export-level` | `summary` | aiperf 导出级别。 |

---

## 产物结构

每次 sweep 都会产生一个带时间戳的输出目录：

```
artifacts/sweep_20260330_143000/
    sweep_config.json        # Full SweepConfig used for this run
    results.csv              # One row per run with key metrics
    summary.md               # Markdown summary table

    mocker_hf_w2_c50_isl512/
        aiperf/              # aiperf JSON output
        prometheus/          # Prometheus metric snapshots
        report.md            # Per-run analysis report (unless --no-report)

    mocker_fastokens_w2_c50_isl512/
        aiperf/
        prometheus/
        report.md
    ...
```

**results.csv 列：**

`run_id`、`backend`、`tokenizer`、`concurrency`、`isl`、`osl`、`workers`、
`speedup_ratio`、`status`、`req_per_sec`、`output_tok_per_sec`、
`ttft_p50_ms`、`ttft_p99_ms`、`itl_p50_ms`、`itl_p99_ms`、`duration_sec`、
`run_dir`

---

## DGD 模板

`dgd/templates/` 目录包含用于 k8s 模式的 DynamoGraphDeployment YAML 模板。模板变量（如 `${DGD_NAME}`、`${IMAGE}`、`${DYN_TOKENIZER_BACKEND}`）会在部署时由 sweep runner 替换。

| 模板 | 后端 | 是否需要 GPU | 说明 |
|----------|---------|-------------|-------------|
| `mocker.yaml` | mocker | 否 | 用于隔离前端/tokenizer 开销的合成后端。 |
| `vllm.yaml` | vLLM | 是 | 端到端基准测试的真实推理后端。 |

---

## 分析

sweep 后的分析脚本位于 `scripts/analysis/`：

| 脚本 | 用途 |
|--------|---------|
| `create_report.py` | 从 aiperf JSON、Prometheus 快照、NVTX trace、syscall profile 与 BPF 数据生成单次运行的可观测性报告。 |
| `frontend_perf_analysis.py` | 生成可扩展性曲线（TTFT/ITL/吞吐 vs. 并发）、ISL 热力图、阶段瀑布图，并支持回归检测。支持单次分析、A/B 对比与热力图生成。 |

**单次运行报告：**

```bash
python3 scripts/analysis/create_report.py analyze artifacts/sweep_*/mocker_hf_w2_c50_isl512/
```

**A/B 对比：**

```bash
python3 scripts/analysis/frontend_perf_analysis.py compare \
    artifacts/sweep_*/mocker_hf_w2_c50_isl512/ \
    artifacts/sweep_*/mocker_fastokens_w2_c50_isl512/
```
