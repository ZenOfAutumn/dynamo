

# SLA Planner 负载测试

本目录包含用于验证 SLA planner 扩缩容行为的完整测试工具。
SLA planner 每 60 秒（默认调整间隔）观测一次指标，并基于 TTFT、ITL 与请求模式
对预填充（prefill）/解码（decode）worker 进行扩缩容。

要搭建环境，可以直接使用任一后端的官方 docker 镜像，或参照 `./examples/backends/<vllm/sglang/trtllm>/README.md` 自行构建 docker 镜像，
或参考 [README.md](../../../../../../README.md) 中的 `Developing Locally` 一节在本地搭建环境。
若使用本地环境，请运行 `UV_GIT_LFS=1 uv pip install --no-cache -r container/deps/requirements.common.txt -r container/deps/requirements.planner.txt` 安装依赖。

## 前置条件：预部署 Profiling 数据

获取预部署 profiling 数据有两种方式：

### 方式 A：使用测试配置（快速开始）

使用预先配置好、附带样例 profiling 数据的测试部署。我们为如下模型 × 硬件
组合提供了结果与部署配置：

- `nvidia/Llama-3.1-8B-Instruct-FP8` 在 H200 上、最大上下文 16384、TP1 Prefill、TP1 Decode。在 ISL/OSL 3000/150 时，可达到 prefill 40k tokens/s/gpu（TTFT 80ms）和 decode 10k tokens/s/gpu（ITL 10ms）。详见 `../tests/data/profiling_results/H200_TP1P_TP1D/`。

### 方式 B：使用你自己的 profiling 结果

1. 针对你的具体配置运行预部署 profiling。详细步骤参见
   [预部署 profiling 文档](../../../../../../docs/components/profiler/profiler-guide.md)。

## 生成负载数据集

我们提供了一个工具用于生成具有变化请求速率的负载数据集。详情参见
[sin_load_generator](../../../../../../benchmarks/sin_load_generator/README.md)。

根据先前 interpolator 测试，ISL 3000、OSL 300 时 prefill 与 decode 都能承载约 15 request/s/gpu。
为测试 planner 在不同请求速率下的表现，可生成请求速率在 12 到 36 request/s 之间变化的负载数据集。
对于 TP1 H200 引擎，planner 应在 1P1D 与 3P3D 之间扩缩。

```bash
python benchmarks/sin_load_generator/sin_synth.py \
  --time-duration 1800 \
  --request-rate-min 5 \
  --request-rate-max 45 \
  --request-rate-period 600 \
  --isl1 3000 \
  --osl1 300 \
  --isl2 3000 \
  --osl2 300 \
  --output-file rr-5-45_i3000o300.jsonl
```

数据集从 5 requests/s 起步，t=300s 时增至 45 requests/s，t=600s 时回到 5 requests/s，并循环往复。
总时长为 30 分钟（1800 秒）。

## 扩缩容测试

本目录包含完整的 SLA planner 扩缩容验证测试。这些测试既验证副本数计算逻辑，
也验证端到端的扩缩容行为。扩缩容测试采用渐进式负载方式而不是数据集文件，
因为这种方式在指标生成与扩缩容触发上更可靠。

### 测试类型

1. **单元测试**（`components/src/dynamo/planner/tests/unit/test_replica_calculation.py`）—— 单独测试 prefill/decode 副本计算的数学公式
2. **端到端测试**（`scaling/run_scaling_test.sh`）—— 测试完整工作流，包括 Kubernetes 部署、负载生成与 pod 扩缩容验证
3. **端到端性能测试**（说明见下文）—— 比较有/无 SLA planner 的部署在性能（goodput 和 goodput/GPU）上的差异

### 单元测试与端到端测试的快速开始

#### 仅运行单元测试

无需 Kubernetes 即可测试副本计算逻辑：

```bash
PYTHONPATH=components/src python -m pytest components/src/dynamo/planner/tests/unit/test_replica_calculation.py -v
```

**注意**：单元测试会自动 mock 外部依赖（prometheus_client、runtime 模块），
以确保它们能在不依赖完整 Dynamo 环境的情况下独立运行。

#### 运行完整端到端测试

测试完整的扩缩容行为，包括 Kubernetes 部署与负载生成。

**前置条件：**

- **已安装并运行 [kube-prometheus-stack](../../../../../../docs/kubernetes/observability/metrics.md)**。SLA planner 需要 Prometheus 来观测指标并做出扩缩容决策。
- 确保 Dynamo operator 在安装时已配置好 Prometheus 端点（详情参见 [SLA Planner 快速开始指南](../../../../../../docs/components/planner/planner-guide.md#prerequisites)）。

**测试场景**

主测试场景验证 H200 在 1P1D -> 2P1D 配置下的 prefill 扩缩容：

- **阶段 1**：8 req/s 持续 90s（基线 - 维持 1P1D）
- **阶段 2**：18 req/s 持续 120s（触发扩容 - 扩至 2P1D）
- **ISL/OSL**：4000/150 tokens（针对 prefill 瓶颈优化）
- **过渡延迟**：阶段间 30s
- **总测试时长**：约 7 分钟 + 扩缩容观察
- **智能清理**：仅当测试自身创建了部署时才清理（保留已存在的部署）

运行测试：

```bash
components/src/dynamo/planner/tests/manual/scaling/run_scaling_test.sh --namespace <namespace>
```

将结果保存到 `components/src/dynamo/planner/tests/e2e_scaling_results` 而不是 `/tmp`：

```bash
components/src/dynamo/planner/tests/manual/scaling/run_scaling_test.sh --namespace <namespace> --save-results
```

### 端到端性能测试说明

在该测试中，我们使用上述 8b FP8 模型在 H200 上，使用 dryrun 中所用数据集，
比较以下四种部署的性能（goodput 和 goodput/GPU）：

- 配置 1：低效 P/D 比 — 3xTP1P_1xTP1D_4GPU
 `./perf_test_configs/disagg_8b_3p1d.yaml`
- 配置 2：最优静态部署 — 2xTP1P_2xTP1D_4GPU
 `./perf_test_configs/disagg_8b_2p2d.yaml`
- 配置 3：低效并行映射 — 1xTP2P_1xTP2D_4GPU
 `./perf_test_configs/disagg_8b_tp2.yaml`
- 配置 4：使用 SLA planner — `./perf_test_configs/disagg_8b_planner.yaml`

针对每种配置运行测试时，先部署对应的 DynamoGraphDeployment：

```bash
kubectl apply -f ./perf_test_configs/<config_file_name> -n <namespace>
```

部署带 sla-planner 的版本时，为减少镜像拉取耗时，可提前部署一个 `DaemonSet` 来缓存镜像：

```bash
kubectl apply -f ./perf_test_configs/image_cache_daemonset.yaml -n <namespace>
```

然后对前端 pod 进行 port-forward 或登录其 shell，运行 AIPerf 获取 goodput：

```bash
aiperf profile \
  --model nvidia/Llama-3.1-8B-Instruct-FP8 \
  --tokenizer nvidia/Llama-3.1-8B-Instruct-FP8 \
  --endpoint-type chat \
  --url localhost:8000 \
  --streaming \
  --input-file /workspace/rr-5-45_i3000o300.jsonl \
  --custom-dataset-type mooncake_trace \
  --goodput "time_to_first_token:200 inter_token_latency:10"
```

> [!NOTE]
> 有时当 SLA planner 缩容 worker 数量时，少量请求会出错并导致 AIPerf 卡住。我们已知该问题并正在修复。

#### 端到端性能测试结果

结果

下表显示了 SLA planner 在不同部署配置下带来的性能提升：


| 基线                                | Goodput 提升        | Goodput/GPU 提升        |
| ----------------------------------- | ------------------- | ----------------------- |
| 低效 P/D 比                         | 725%                | 600%                    |
| 低效并行映射                        | 311%                | 249%                    |
| 最优静态部署                        | 52%                 | 29%                     |
