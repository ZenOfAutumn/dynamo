---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Disaggregated Serving
subtitle: Find optimal prefill/decode configuration for disaggregated serving deployments
---

[AIConfigurator](https://github.com/ai-dynamo/aiconfigurator/tree/main) 是一款性能优化工具，能够帮助你为 Dynamo 上部署的 LLM 找到最优配置。它会自动确定 prefill（预填充）与 decode（解码）worker 数量、并行设置以及部署参数，以在满足 SLA 目标的同时最大化吞吐。

## 为什么使用 AIConfigurator？

在使用 Dynamo 部署 LLM 时，你需要做出若干关键决策：
- **聚合 vs 解耦**：哪种架构能在你的工作负载下提供更好的性能？
- **Worker 配置**：部署多少 prefill 与 decode worker？
- **并行设置**：使用哪种张量 / 流水线并行配置？
- **SLA 合规**：如何达成 TTFT 与 TPOT 目标？

AIConfigurator 在数秒内回答上述问题，给出：
- 满足 SLA 要求的推荐配置
- 可立即部署的 Dynamo 配置文件（包括 Kubernetes 清单）
- 不同部署策略之间的性能比较
- 相对手工配置最高 1.7 倍的吞吐改进

### 端到端工作流

![AIConfigurator end-to-end workflow](../../assets/img/e2e-workflow.svg)

### 聚合 vs 解耦架构

AIConfigurator 评估两种部署架构，并为你的负载推荐最佳方案：

![Aggregated vs Disaggregated architecture comparison](../../assets/img/arch-comparison.svg)

### 何时使用哪种架构

![Decision flowchart for choosing aggregated vs disaggregated](../../assets/img/decision-flowchart.svg)

## 快速上手

```bash
# Install
pip3 install aiconfigurator

# Find optimal configuration for vLLM backend
aiconfigurator cli default \
  --model Qwen/Qwen3-32B-FP8 \
  --total-gpus 8 \
  --system h200_sxm \
  --backend vllm \
  --backend-version 0.12.0 \
  --isl 4000 \
  --osl 500 \
  --ttft 600 \
  --tpot 16.67 \
  --save-dir ./results_vllm

# Deploy on Kubernetes
kubectl apply -f ./results_vllm/agg/top1/agg/k8s_deploy.yaml
```

## 完整流程：vLLM on H200

本节走通一个已验证示例：在 8× H200 GPU 上使用 vLLM 部署 Qwen3-32B-FP8。

### 步骤 1：运行 AIConfigurator

```bash
aiconfigurator cli default \
  --model Qwen/Qwen3-32B-FP8 \
  --system h200_sxm \
  --total-gpus 8 \
  --isl 4000 \
  --osl 500 \
  --ttft 600 \
  --tpot 25 \
  --backend vllm \
  --backend-version 0.12.0 \
  --generator-dynamo-version 1.0.0 \
  --generator-set K8sConfig.k8s_namespace=$YOUR_NAMESPACE \
  --generator-set K8sConfig.k8s_pvc_name=$YOUR_PVC \
  --save-dir ./results_vllm
```

**参数说明：**
- `--model`：HuggingFace 模型 ID 或本地路径（例如 `Qwen/Qwen3-32B-FP8`）
- `--system`：GPU 系统类型（`h200_sxm`、`h100_sxm`、`a100_sxm`）
- `--total-gpus`：可用于部署的 GPU 数
- `--isl` / `--osl`：输入 / 输出序列长度（token）
- `--ttft` / `--tpot`：SLA 目标 —— Time To First Token（毫秒）与 Time Per Output Token（毫秒）
- `--backend`：推理后端（`vllm`、`trtllm` 或 `sglang`）
- `--backend-version`：后端版本（例如 vLLM 的 `0.12.0`）
- `--save-dir`：保存生成的部署配置的目录

### 步骤 2：查看结果

AIConfigurator 输出聚合与解耦部署策略的对比：

```text
********************************************************************************
*                     Dynamo aiconfigurator Final Results                      *
********************************************************************************
  ----------------------------------------------------------------------------
  Input Configuration & SLA Target:
    Model: Qwen/Qwen3-32B-FP8 (is_moe: False)
    Total GPUs: 8
    Best Experiment Chosen: disagg at 446.85 tokens/s/gpu (disagg 1.38x better)
  ----------------------------------------------------------------------------
  Overall Best Configuration:
    - Best Throughput: 3,574.80 tokens/s
    - Per-GPU Throughput: 446.85 tokens/s/gpu
    - Per-User Throughput: 53.58 tokens/s/user
    - TTFT: 453.18ms
    - TPOT: 18.66ms
    - Request Latency: 9766.51ms
  ----------------------------------------------------------------------------
  Pareto Frontier:
      Qwen/Qwen3-32B-FP8 Pareto Frontier: tokens/s/gpu_cluster vs tokens/s/user
     ┌─────────────────────────────────────────────────────────────────────────┐
850.0┤ •• agg                                                                  │
     │ ff disagg                                                               │
     │ xx disagg best                                                          │
     │                                                                         │
708.3┤                                                                         │
     │         f                                                               │
     │         f                                                               │
     │          fff                                                            │
566.7┤             f                                                           │
     │             f                                                           │
     │              f                                                          │
     │    ••         fffffffffffffffffx                                        │
425.0┤     ••••                        ff                                      │
     │        •••                       f                                      │
     │           •••••                  f                                      │
     │                ••••••••••        f                                      │
283.3┤                          •••     f                                      │
     │                             ••    f                                     │
     │                               ••  f                                     │
     │                                ••••f                                    │
141.7┤                                   •f•                                   │
     │                                     f•••••                              │
     │                                      f    •••••••                       │
     │                                       fffff      ••••                   │
  0.0┤                                                      ••••               │
     └┬─────────────────┬─────────────────┬─────────────────┬─────────────────┬┘
      0                30                60                90               120
tokens/s/gpu_cluster                tokens/s/user

  ----------------------------------------------------------------------------
  Deployment Details:
    (p) stands for prefill, (d) stands for decode, bs stands for batch size, a replica stands for the smallest scalable unit xPyD of the disagg system
    Some math: total gpus used = replicas * gpus/replica
               gpus/replica = (p)gpus/worker * (p)workers + (d)gpus/worker * (d)workers; for Agg, gpus/replica = gpus/worker
               gpus/worker = tp * pp * dp = etp * ep * pp for MoE models; tp * pp for dense models (underlined numbers are the actual values in math)

agg Top Configurations: (Sorted by tokens/s/gpu)
+------+---------+--------------+---------------+--------+-----------------+-------------+-------------------+----------+--------------+-------------+----------+----+
| Rank | backend | tokens/s/gpu | tokens/s/user |  TTFT  | request_latency | concurrency | total_gpus (used) | replicas | gpus/replica | gpus/worker | parallel | bs |
+------+---------+--------------+---------------+--------+-----------------+-------------+-------------------+----------+--------------+-------------+----------+----+
|  1   |   vllm  |    322.69    |     41.78     | 546.92 |     12490.03    |  64 (=32x2) |     8 (8=2x4)     |    2     |      4       |  4 (=4x1x1) |  tp4pp1  | 32 |
|  2   |   vllm  |    293.94    |     44.43     | 593.10 |     11823.67    |  56 (=14x4) |     8 (8=4x2)     |    4     |      2       |  2 (=2x1x1) |  tp2pp1  | 14 |
|  3   |   vllm  |    208.87    |     42.90     | 460.58 |     12093.52    |  40 (=40x1) |     8 (8=1x8)     |    1     |      8       |  8 (=8x1x1) |  tp8pp1  | 40 |
+------+---------+--------------+---------------+--------+-----------------+-------------+-------------------+----------+--------------+-------------+----------+----+

disagg Top Configurations: (Sorted by tokens/s/gpu)
+------+---------+--------------+---------------+--------+-----------------+-------------+-------------------+----------+--------------+------------+----------------+-------------+-------+------------+----------------+-------------+-------+
| Rank | backend | tokens/s/gpu | tokens/s/user |  TTFT  | request_latency | concurrency | total_gpus (used) | replicas | gpus/replica | (p)workers | (p)gpus/worker | (p)parallel | (p)bs | (d)workers | (d)gpus/worker | (d)parallel | (d)bs |
+------+---------+--------------+---------------+--------+-----------------+-------------+-------------------+----------+--------------+------------+----------------+-------------+-------+------------+----------------+-------------+-------+
|  1   |   vllm  |    446.85    |     53.58     | 453.18 |     9766.51     |  76 (=76x1) |     8 (8=1x8)     |    1     | 8 (=2x2+1x4) |     2      |    2 (=2x1)    |    tp2pp1   |   1   |     1      |    4 (=4x1)    |    tp4pp1   |   76  |
|  2   |   vllm  |    446.85    |     41.14     | 453.18 |     12581.87    | 144 (=72x2) |     8 (8=2x4)     |    2     | 4 (=1x2+1x2) |     1      |    2 (=2x1)    |    tp2pp1   |   1   |     1      |    2 (=2x1)    |    tp2pp1   |   72  |
|  3   |   vllm  |    333.73    |     40.22     | 453.18 |     12860.32    |  72 (=36x2) |     8 (8=2x4)     |    2     | 4 (=1x2+2x1) |     1      |    2 (=2x1)    |    tp2pp1   |   1   |     2      |    1 (=1x1)    |    tp1pp1   |   18  |
+------+---------+--------------+---------------+--------+-----------------+-------------+-------------------+----------+--------------+------------+----------------+-------------+-------+------------+----------------+-------------+-------+
```

**输出解读：**
- **tokens/s/gpu**：整体吞吐效率 —— 越高越好
- **tokens/s/user**：每请求生成速度（TPOT 的反数）
- **TTFT**：预测的首字延迟
- **concurrency**：所有副本上并发请求的总数（例如 `56 (=14x4)` 表示批大小 14 × 4 副本）
- **agg Rank 1** 推荐 TP4 + 2 副本 —— 部署更简单
- **disagg Rank 1** 推荐 2 个 prefill worker（TP2）+ 1 个 decode worker（TP4） —— 吞吐更高，但需要 RDMA

### 步骤 3：在 Kubernetes 上部署

`--save-dir` 会生成可直接使用的 Kubernetes 清单：

```
├── agg
│   ├── best_config_topn.csv
│   ├── exp_config.yaml
│   ├── pareto.csv
│   ├── top1
│   │   ├── agg_config.yaml
│   │   ├── bench_run.sh          # aiperf benchmark sweep script (bare-metal)
│   │   ├── generator_config.yaml
│   │   ├── k8s_bench.yaml        # aiperf benchmark sweep Job (Kubernetes)
│   │   ├── k8s_deploy.yaml       # Kubernetes DynamoGraphDeployment
│   │   └── run_0.sh
│   ...
├── disagg
│   ├── best_config_topn.csv
│   ├── exp_config.yaml
│   ├── pareto.csv
│   ├── top1
│   │   ├── bench_run.sh          # aiperf benchmark sweep script (bare-metal)
│   │   ├── decode_config.yaml
│   │   ├── generator_config.yaml
│   │   ├── k8s_bench.yaml        # aiperf benchmark sweep Job (Kubernetes)
│   │   ├── k8s_deploy.yaml       # Kubernetes DynamoGraphDeployment
│   │   ├── prefill_config.yaml
│   │   ├── run_0.sh
│   │   └── run_1.sh  (for multi-node setups)
│   ...
└── pareto_frontier.png
```

#### 前置条件

部署前请确认：

1. **HuggingFace Token Secret**（受限模型需要）：
   ```bash
   kubectl create secret generic hf-token-secret \
     -n your-namespace \
     --from-literal=HF_TOKEN="your-huggingface-token"
   ```

2. **Model Cache PVC**（推荐，用于更快重启）：
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: model-cache
     namespace: your-namespace
   spec:
     accessModes:
       - ReadWriteMany
     resources:
       requests:
         storage: 100Gi
   ```

#### 部署该配置

生成的 `k8s_deploy.yaml` 提供起点。通常需要按你的环境定制：

```bash
kubectl apply -f ./results_vllm/agg/top1/agg/k8s_deploy.yaml
```

**完整部署示例**（启用模型缓存与生产化设置）：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: dynamo-agg
  namespace: your-namespace
spec:
  backendFramework: vllm
  pvcs:
    - name: model-cache
      create: false           # Use existing PVC
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      volumeMounts:
        - name: model-cache
          mountPoint: /opt/models
      envs:
        - name: HF_HOME
          value: /opt/models
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          imagePullPolicy: IfNotPresent

    VLLMWorker:
      envFromSecret: hf-token-secret
      componentType: worker
      replicas: 4
      resources:
        limits:
          gpu: "2"
      sharedMemory:
        size: 16Gi            # Required for vLLM
      volumeMounts:
        - name: model-cache
          mountPoint: /opt/models
      envs:
        - name: HF_HOME
          value: /opt/models
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          workingDir: /workspace
          imagePullPolicy: IfNotPresent
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - "Qwen/Qwen3-32B-FP8"
            - "--no-enable-prefix-caching"
            - "--tensor-parallel-size"
            - "2"
            - "--pipeline-parallel-size"
            - "1"
            - "--data-parallel-size"
            - "1"
            - "--kv-cache-dtype"
            - "fp8"
            - "--max-model-len"
            - "6000"
            - "--max-num-seqs"
            - "1024"
```

**关键部署设置：**

| 设置 | 用途 | 备注 |
|---------|---------|-------|
| `backendFramework: vllm` | 告诉 Dynamo 使用哪个 runtime | spec 顶层必填 |
| `pvcs` + `volumeMounts` | 跨重启缓存模型权重 | 挂在 `/opt/models`（不要在 `/root/`） |
| `HF_HOME` 环境变量 | 让 HuggingFace 指向缓存位置 | 必须与 `mountPoint` 一致 |
| `sharedMemory.size: 16Gi` | vLLM 的 IPC 内存 | vLLM 用 16Gi，TRT-LLM 用 80Gi |
| `envFromSecret` | 注入 HF_TOKEN | 受限模型必需 |

### 步骤 4：用 AIPerf 验证

部署完成后，使用 [AIPerf](https://github.com/ai-dynamo/aiperf) 将预测与实际性能对比。

> ℹ️ 在 **集群内部** 运行 AIPerf，避免网络时延影响测量。

AIC 在生成 Dynamo 配置时会一并输出 AIPerf 脚本（当指定 `--save-dir ...` 时）。Kubernetes 部署使用 `k8s_bench.yaml`；裸机系统使用 `bench_run.sh` 脚本。这些脚本会以一组并发度执行 AIPerf：默认集合（`1 2 8 16 32 64 128`），并加上 `BenchConfig.estimated_concurrency` 及其 ±5% 范围。你也可以按需自定义该并发度列表。

默认情况下，AIPerf 结果保存在容器的 `/tmp/bench_artifacts`。如果通过 `--generator-set K8sConfig.k8s_pvc_name=$YOUR_PVC` 指定了 PVC 名称，则结果会保存到 PVC 挂载点。

![AIC-to-AIPerf parameter mapping](../../assets/img/param-mapping.svg)

| AIC 输出 | AIPerf 参数 | 备注 |
|------------|-----------------|-------|
| `concurrency: 56 (=14x4)` | `--concurrency 56` | 通过 frontend 基准时使用总并发度 |
| ISL/OSL 目标 | `--isl 4000 --osl 500` | 与 AIC 输入一致 |
| - | `--num-requests 800` | 至少使用 `concurrency × 40` 以获得统计稳定性 |
| - | `--extra-inputs "ignore_eos:true"` | 确保生成精确的 OSL token 数 |

> **关于并发度**：AIC 报告的并发度形如 `total (=bs × replicas)`。通过 frontend（其会路由到所有副本）做基准时使用总值。如直接对单副本基准，则使用每副本 `bs` 值。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: aiperf-benchmark
  namespace: your-namespace
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: aiperf
        image: python:3.10
        command:
        - /bin/bash
        - -c
        - |
          pip install aiperf
          aiperf profile \
            -m Qwen/Qwen3-32B-FP8 \
            --endpoint-type chat \
            -u http://dynamo-agg-frontend:8000 \
            --isl 4000 --isl-stddev 0 \
            --osl 500 --osl-stddev 0 \
            --num-requests 800 \
            --concurrency 56 \
            --streaming \
            --extra-inputs "ignore_eos:true" \
            --num-warmup-requests 40 \
            --ui-type simple
```

```bash
kubectl apply -f k8s_bench.yaml
kubectl logs -f -l job-name=aiperf-benchmark
```

**已验证的结果**（Qwen3-32B-FP8，8× H200，TP2×4 副本，聚合）：

| 指标 | AIC 预测 | 实测（avg） | 状态 |
|--------|---------------|--------------|--------|
| TTFT (ms) | 509 | 209 | 优于目标 |
| ITL/TPOT (ms) | 16.49 | 15.06 | 误差 10% 以内 |
| 吞吐 (req/s) | ~6.3 | 6.9 | 误差 10% 以内 |
| 总输出 TPS | ~3,178 | 3,462 | 误差 10% 以内 |

<Note>
实际吞吐通常达到 AIC 预测的约 85-90%，其中 ITL/TPOT 是最准确的指标。多次运行之间会有些方差，建议多次执行。对于重复 prompt，启用 prefix caching（`--enable-prefix-caching`）可以进一步改善 TTFT。
</Note>

## 微调你的部署

AIConfigurator 给出了一个稳健的起点。下面给出向生产迭代的方式：

### 按真实负载调整

如果你的真实负载与基准参数不同：

```bash
# For longer outputs (chat/code generation):
# increase OSL, relax TTFT target
aiconfigurator cli default \
  --model Qwen/Qwen3-32B-FP8 \
  --total-gpus 8 \
  --system h200_sxm \
  --backend vllm \
  --backend-version 0.12.0 \
  --isl 2000 \
  --osl 2000 \
  --ttft 1000 \
  --tpot 10 \
  --save-dir ./results_long_output
```

### 探索备选配置

使用 `exp` 模式比较自定义配置：

```yaml
# custom_exp.yaml
exps:
  - exp_tp2
  - exp_tp4

exp_tp2:
  mode: "patch"
  serving_mode: "agg"
  model_path: "Qwen/Qwen3-32B-FP8"
  total_gpus: 8
  system_name: "h200_sxm"
  backend_name: "vllm"
  backend_version: "0.12.0"
  isl: 4000
  osl: 500
  ttft: 600
  tpot: 16.67
  config:
    agg_worker_config:
      tp_list: [2]

exp_tp4:
  mode: "patch"
  serving_mode: "agg"
  model_path: "Qwen/Qwen3-32B-FP8"
  total_gpus: 8
  system_name: "h200_sxm"
  backend_name: "vllm"
  backend_version: "0.12.0"
  isl: 4000
  osl: 500
  ttft: 600
  tpot: 16.67
  config:
    agg_worker_config:
      tp_list: [4]
```

```bash
aiconfigurator cli exp --yaml-path custom_exp.yaml --save-dir ./results_custom
```

> **关键**：解耦部署**需要 RDMA**才能传输 KV 缓存。没有 RDMA 时，性能会下降 **40 倍**（TTFT 从 355ms 增加到 10 秒以上）。详见下文 "解耦部署" 一节。

### 部署解耦（需要 RDMA）

解耦部署在 prefill 与 decode worker 之间传输 KV 缓存。**没有 RDMA 时，该传输将成为严重瓶颈**，引发 40 倍的性能下降。

#### 解耦部署的前置条件

1. **支持 RDMA 的网络**（InfiniBand 或 RoCE）
2. 集群上**已安装 RDMA 设备插件**（提供 `rdma/ib` 资源）
3. 已部署 **ETCD 与 NATS**（用于协调）

#### 带 RDMA 的解耦 DGD

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: dynamo-disagg
  namespace: your-namespace
spec:
  backendFramework: vllm
  pvcs:
    - name: model-cache
      create: false
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      volumeMounts:
        - name: model-cache
          mountPoint: /opt/models
      envs:
        - name: HF_HOME
          value: /opt/models
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          imagePullPolicy: IfNotPresent

    VLLMPrefillWorker:
      envFromSecret: hf-token-secret
      componentType: worker
      subComponentType: prefill
      replicas: 2
      resources:
        limits:
          gpu: "2"
      sharedMemory:
        size: 16Gi
      volumeMounts:
        - name: model-cache
          mountPoint: /opt/models
      envs:
        - name: HF_HOME
          value: /opt/models
        - name: UCX_TLS
          value: "rc_x,rc,dc_x,dc,cuda_copy,cuda_ipc"  # Enable RDMA transports
        - name: UCX_RNDV_SCHEME
          value: "get_zcopy"
        - name: UCX_RNDV_THRESH
          value: "0"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          workingDir: /workspace
          imagePullPolicy: IfNotPresent
          securityContext:
            capabilities:
              add: ["IPC_LOCK"]  # Required for RDMA memory registration
          resources:
            limits:
              rdma/ib: "2"      # Request RDMA resources
            requests:
              rdma/ib: "2"
          command: ["python3", "-m", "dynamo.vllm"]
          args:
            - --model
            - "Qwen/Qwen3-32B-FP8"
            - "--tensor-parallel-size"
            - "2"
            - "--kv-cache-dtype"
            - "fp8"
            - "--max-num-seqs"
            - "1"               # Prefill workers use batch size 1
            - --disaggregation-mode
            - prefill

    VLLMDecodeWorker:
      envFromSecret: hf-token-secret
      componentType: worker
      subComponentType: decode
      replicas: 1
      resources:
        limits:
          gpu: "4"
      sharedMemory:
        size: 16Gi
      volumeMounts:
        - name: model-cache
          mountPoint: /opt/models
      envs:
        - name: HF_HOME
          value: /opt/models
        - name: UCX_TLS
          value: "rc_x,rc,dc_x,dc,cuda_copy,cuda_ipc"
        - name: UCX_RNDV_SCHEME
          value: "get_zcopy"
        - name: UCX_RNDV_THRESH
          value: "0"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          workingDir: /workspace
          imagePullPolicy: IfNotPresent
          securityContext:
            capabilities:
              add: ["IPC_LOCK"]
          resources:
            limits:
              rdma/ib: "4"
            requests:
              rdma/ib: "4"
          command: ["python3", "-m", "dynamo.vllm"]
          args:
            - --model
            - "Qwen/Qwen3-32B-FP8"
            - "--tensor-parallel-size"
            - "4"
            - "--kv-cache-dtype"
            - "fp8"
            - "--max-num-seqs"
            - "1024"            # Decode workers handle high concurrency
            - --disaggregation-mode
            - decode
```

**关键 RDMA 设置：**

| 设置 | 用途 |
|---------|---------|
| `rdma/ib: "N"` | 申请 N 个 RDMA 资源（与 TP 大小匹配） |
| `IPC_LOCK` capability | RDMA 内存注册所需 |
| `UCX_TLS` 环境变量 | 启用 RDMA 传输（rc_x、dc_x） |
| `UCX_RNDV_SCHEME=get_zcopy` | 零拷贝 RDMA 传输 |

#### 验证 RDMA 已生效

部署后，检查 worker 日志中的 UCX 初始化：

```bash
kubectl logs <prefill-worker-pod> | grep -i "UCX\|NIXL"
```

你应当看到：
```
NIXL INFO Backend UCX was instantiated
```

如果只看到 TCP 传输，则 RDMA 未生效 —— 检查 RDMA 设备插件与资源请求。

### 调整 vLLM 特有参数

通过 `--generator-set` 覆盖 vLLM 引擎参数：

```bash
aiconfigurator cli default \
  --model Qwen/Qwen3-32B-FP8 \
  --total-gpus 8 \
  --system h200_sxm \
  --backend vllm \
  --backend-version 0.12.0 \
  --isl 4000 --osl 500 \
  --ttft 600 --tpot 16.67 \
  --save-dir ./results_tuned \
  --generator-set Workers.agg.kv_cache_free_gpu_memory_fraction=0.85 \
  --generator-set Workers.agg.max_num_seqs=2048
```

运行 `aiconfigurator cli default --generator-help` 可查看所有可用参数。

### 关于 Prefix Caching 的考量

对于含重复前缀（例如系统提示）的负载：

- 当你有较高的前缀命中率时，**启用 prefix caching**
- 对于多样化的 prompt，**禁用 prefix caching**（`--no-enable-prefix-caching`）

AIConfigurator 的默认预测假定不启用 prefix caching。如果你的负载从中受益，部署后再启用即可。

## 支持的配置

### 后端与版本

模型 / 系统 / 后端 / 版本组合在聚合与解耦模式下的支持详情，参见 [**支持矩阵 CSV**](https://github.com/ai-dynamo/aiconfigurator/blob/main/src/aiconfigurator/systems/support_matrix.csv)。该文件由自动生成与测试确保所有支持组合的准确性。

也可以通过 `aiconfigurator cli support` 命令检查某个系统 / 框架版本是否支持。例如：
```bash
aiconfigurator cli support --model Qwen/Qwen3-32B-FP8 --system h100_sxm --backend-version 1.2.0rc5
```


## 常见用例

```bash
# Strict latency SLAs (real-time chat)
aiconfigurator cli default \
  --model meta-llama/Llama-3.1-70B \
  --total-gpus 16 \
  --system h200_sxm \
  --backend vllm \
  --backend-version 0.12.0 \
  --ttft 200 --tpot 8

# High throughput (batch processing)
aiconfigurator cli default \
  --model Qwen/Qwen3-32B-FP8 \
  --total-gpus 32 \
  --system h200_sxm \
  --backend trtllm \
  --ttft 2000 --tpot 50

# Request latency constraint (end-to-end SLA)
aiconfigurator cli default \
  --model Qwen/Qwen3-32B-FP8 \
  --total-gpus 16 \
  --system h200_sxm \
  --backend vllm \
  --backend-version 0.12.0 \
  --request-latency 12000 \
  --isl 4000 --osl 500
```

## 其他选项

```bash
# Web interface for interactive exploration
pip3 install aiconfigurator[webapp]
aiconfigurator webapp  # Visit http://127.0.0.1:7860

# Quick config generation (no parameter sweep)
aiconfigurator cli generate \
  --model Qwen/Qwen3-32B-FP8 \
  --total-gpus 8 \
  --system h200_sxm \
  --backend vllm

# Check model/system support
aiconfigurator cli support \
  --model Qwen/Qwen3-32B-FP8 \
  --system h200_sxm \
  --backend vllm
```

## 故障排查

### AIConfigurator 问题

**Model not found**：使用完整 HuggingFace 路径（例如 `Qwen/Qwen3-32B-FP8` 而不是 `QWEN3_32B`）

**Backend 版本不匹配**：通过 `aiconfigurator cli support --model <model> --system <system> --backend <backend>` 查看支持版本

### 部署问题

**Pod 因 cache 目录的 "Permission denied" 而崩溃**：
- 将 PVC 挂载到 `/opt/models`，而不是 `/root/.cache/huggingface`
- 设置 `HF_HOME=/opt/models` 环境变量
- 确认 PVC 使用 `ReadWriteMany` 访问模式

**Worker 卡在 CrashLoopBackOff**：
- 查看日志：`kubectl logs <pod-name> --previous`
- 确认设置了 `sharedMemory.size`（vLLM 用 16Gi，TRT-LLM 用 80Gi）
- 确认 HuggingFace token Secret 存在且名称正确

**每次重启模型都重新下载**：
- 加上 PVC 做模型缓存（参见上面的部署示例）
- 确认 worker 上正确配置了 `volumeMounts` 与 `HF_HOME`

**"Context stopped or killed" 错误（仅解耦）**：
- 部署 ETCD 与 NATS 基础设施（KV 缓存传输所需）
- 平台搭建参见 [Dynamo Kubernetes Guide](../../kubernetes/README.md)

### 性能问题

**OOM**：减小 `--max-num-seqs` 或增加张量并行度

**性能低于预测**：
- 确认预热请求数量足够（建议 40+）
- 检查集群上是否有竞争负载
- 确认 KV 缓存内存比例已优化
- 在集群内部进行基准测试以排除网络时延

**解耦 TTFT 极高（10+ 秒）**：
几乎总是源于**未配置 RDMA**。没有 RDMA 时，KV 缓存传输回退到 TCP 并成为严重瓶颈。

诊断方法：
```bash
# Check if RDMA resources are allocated
kubectl get pod <worker-pod> -o yaml | grep -A5 "resources:"

# Check UCX transport in logs
kubectl logs <worker-pod> | grep -i "UCX\|transport"
```

修复方法：
1. 确保集群已安装 RDMA 设备插件
2. 在 worker pod 上添加 `rdma/ib` 资源请求
3. 在 securityContext 中加入 `IPC_LOCK` capability
4. 加入 UCX 环境变量（参见解耦部署一节）

**解耦能用但吞吐低于聚合**：
对均衡的负载（ISL/OSL 比在 2:1 到 10:1 之间），聚合通常更好。解耦更适合：
- 输入很长（ISL > 8000）且输出较短的场景
- 需要 prefill / decode 独立扩缩的场景

## 进一步阅读

- [AIConfigurator CLI Guide](https://github.com/ai-dynamo/aiconfigurator/blob/main/docs/cli_user_guide.md)
- [Dynamo Deployment Guide](https://github.com/ai-dynamo/aiconfigurator/blob/main/docs/dynamo_deployment_guide.md)
- [Dynamo Installation Guide](../../kubernetes/installation-guide.md)
- [Benchmarking Guide](../../benchmarks/benchmarking.md)
