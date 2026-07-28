---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: KV Router A/B Testing
---

本指南介绍如何在 Kubernetes 集群上搭建并运行 A/B 基准测试，对比 Dynamo KV Smart Router 与标准轮询路由的表现。

## 概述

Dynamo 的 KV Smart Router 会根据 KV 缓存（KV cache）亲和性智能路由请求，对带共享 prompt 前缀的工作负载有显著提升。本指南帮助你：

1. 部署两份完全相同的 Dynamo 配置：
   a. 一份针对 Qwen3-32B、8 个 worker（聚合）的 vllm 服务，**未启用** KV Smart Router
   b. 一份针对 Qwen3-32B、8 个 worker（聚合）的 vllm 服务，**启用** KV Smart Router
2. 使用 AIPerf 运行受控基准测试
3. 对比性能指标，评估 KV router 的有效性

**前置条件：** 带 GPU 的 Kubernetes 集群、kubectl、helm

---

## 前置条件

### 必备工具

- `kubectl`（已配置集群访问）
- `helm`（v3+）
- HuggingFace 账号与 token（如模型下载受限需要）
- Kubernetes 集群：
  - GPU 节点（H100、H200 或类似）
  - 足够的 GPU 容量（本示例建议 8+ GPU）
  - 已全局安装 Dynamo platform，或可按 namespace 安装

### 知识储备

- 基本 Kubernetes 概念（namespace、pod、service）
- 熟悉 LLM 推理（inference）概念
- 熟练使用命令行

---

## 架构

本指南使用单一 namespace。先部署一个配置（如 router-ON），跑 benchmark，拆掉，再部署另一个（router-OFF），跑同样的 benchmark。

```text
┌──────────────────────────────────────────────┐
│ Namespace: dynamo-bench                       │
│ (one of A or B active at a time)              │
│                                              │
│  Deployment A: Router OFF                     │
│    ├─ Frontend (Standard Routing)              │
│    └─ 8x Decode Workers (1 GPU each)          │
│                                              │
│  Deployment B: Router ON                      │
│    ├─ Frontend (KV Smart Router)               │
│    └─ 8x Decode Workers (1 GPU each)          │
│                                              │
│  Benchmark Pod (AIPerf + Dataset)             │
└──────────────────────────────────────────────┘
```

**关键差异：** Deployment B 在前端设置 `DYN_ROUTER_MODE=kv` 以启用 KV 缓存感知路由。

---

## 阶段 1：Namespace 与基础设施搭建

### 步骤 1.1：创建 namespace

```bash
kubectl create namespace dynamo-bench
```

### 步骤 1.2：创建 HuggingFace token secret（可选）

如果要部署的模型需要 HF token 下载（Llama 系列需要），把 `YOUR_HF_TOKEN` 替换成你的真实 HuggingFace token：

```bash
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="YOUR_HF_TOKEN" \
  -n dynamo-bench
```

### 步骤 1.3：安装 Dynamo Platform

按照 [Dynamo Kubernetes 安装指南](https://github.com/ai-dynamo/dynamo/blob/main/docs/kubernetes/installation-guide.md)，把 platform 安装到 `dynamo-bench`。

> **注意：** namespace 受限模式（`namespaceRestriction.enabled=true`）已弃用，将在未来版本移除。新部署请用 cluster-wide 模式。

**关键配置注意事项：**
- 调整版本 tag 以匹配你集群可用的 Dynamo 版本
- 若遇到 operator 兼容性问题（如 MPI 参数不被支持），请咨询集群管理员或参考 Dynamo 故障排查文档

### 步骤 1.4：验证基础设施

```bash
kubectl get pods -n dynamo-bench
```

在部署 graph 之前，应能看到 operator、etcd、nats 的 pod 处于 Running 状态。

---

## 阶段 2：部署模型服务

### 步骤 2.1：创建 deployment YAML

创建 `router-off-deployment.yaml`（基线）：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-agg-no-router
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          env:
            - name: POD_UID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.uid
    VllmDecodeWorker:
      envFromSecret: hf-token-secret
      componentType: worker
      replicas: 8
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
                - matchExpressions:
                    - key: node.kubernetes.io/instance-type
                      operator: In
                      values:
                        - gpu-h100-sxm  # Adjust to your GPU node type
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          workingDir: /workspace
          command:
            - /bin/sh
            - -c
          args:
            - >-
              python3 -m dynamo.vllm
              --model Qwen/Qwen3-32B
              --quantization fp8
              --kv-cache-dtype fp8
              --max-model-len 131072
              --hf-overrides '{"rope_scaling":{"rope_type":"yarn","factor":4.0,"original_max_position_embeddings":32768},"max_position_embeddings":131072}'
              --gpu-memory-utilization 0.90
              --block-size 64
              --async-scheduling
              --no-enable-log-requests
          env:
            - name: DYN_HEALTH_CHECK_ENABLED
              value: "false"
            - name: POD_UID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.uid
          startupProbe:
            httpGet:
              path: /health
              port: 9090
            initialDelaySeconds: 120
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 60
          livenessProbe:
            httpGet:
              path: /live
              port: 9090
            initialDelaySeconds: 300
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 10
          readinessProbe:
            httpGet:
              path: /live
              port: 9090
            initialDelaySeconds: 300
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 10
      subComponentType: decode
```

创建 `router-on-deployment.yaml`（启用 KV router）：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-agg-router
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          env:
            - name: POD_UID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.uid
      envs:
        - name: DYN_ROUTER_MODE
          value: kv  # KEY DIFFERENCE: Enable KV Smart Router
    VllmDecodeWorker:
      envFromSecret: hf-token-secret
      componentType: worker
      replicas: 8
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
                - matchExpressions:
                    - key: node.kubernetes.io/instance-type
                      operator: In
                      values:
                        - gpu-h100-sxm  # Adjust to your GPU node type
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          workingDir: /workspace
          command:
            - /bin/sh
            - -c
          args:
            - >-
              python3 -m dynamo.vllm
              --model Qwen/Qwen3-32B
              --quantization fp8
              --kv-cache-dtype fp8
              --max-model-len 131072
              --hf-overrides '{"rope_scaling":{"rope_type":"yarn","factor":4.0,"original_max_position_embeddings":32768},"max_position_embeddings":131072}'
              --gpu-memory-utilization 0.90
              --block-size 64
              --async-scheduling
              --no-enable-log-requests
          env:
            - name: DYN_HEALTH_CHECK_ENABLED
              value: "false"
            - name: POD_UID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.uid
          startupProbe:
            httpGet:
              path: /health
              port: 9090
            initialDelaySeconds: 120
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 60
          livenessProbe:
            httpGet:
              path: /live
              port: 9090
            initialDelaySeconds: 300
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 10
          readinessProbe:
            httpGet:
              path: /live
              port: 9090
            initialDelaySeconds: 300
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 10
      subComponentType: decode
```

### 步骤 2.2：先部署 Router-ON

```bash
kubectl apply -f router-on-deployment.yaml -n dynamo-bench
```

**优化提示：** 每个 worker 会独立下载模型（约 20 分钟/pod）。要加快初始化，可以挂一个 `ReadWriteMany` 共享 PVC 来缓存模型。

首先在 deployment 所在 namespace（如 `dynamo-bench`）创建该 PVC。要选支持 ReadWriteMany 的 storage class：

```bash
kubectl get storageclass   # choose one with ReadWriteMany (e.g. azurefile-csi-premium, nfs, efs)
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-cache
  namespace: dynamo-bench
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: "azurefile-csi-premium"   # Adjust to your cluster
  resources:
    requests:
      storage: 100Gi
```

应用：`kubectl apply -f pvc-model-cache.yaml`

然后在 DynamoGraphDeployment 的 `spec` 下引用已存在的 PVC（并在 `VllmDecodeWorker` 下添加 `volumeMounts`）：

```yaml
spec:
  pvcs:
    - create: false
      name: model-cache
      size: "0"
  services:
    VllmDecodeWorker:
      volumeMounts:
        - mountPoint: /root/.cache/huggingface
          name: model-cache
          useAsCompilationCache: false
```

这样配置后，首次运行只有一个 worker 在下载，其余从缓存加载。最大的好处在于重新部署时：模型留在 PVC 上，新 pod 从缓存加载，约 5–10 分钟即可起来，不需要重新下载。

### 步骤 2.3：监控部署进度

```bash
kubectl get pods -n dynamo-bench -w
```

等待所有 pod 进入 `Running` 状态并通过就绪探针。

**预期时间：**
- **使用共享 PVC**（ReadWriteMany）：约 5–10 分钟（首个 worker 下载，其余使用缓存）
- **不使用共享 PVC**：每个 worker 20–30 分钟（worker 各自独立下载）
  - 8 个 worker：完整部署预算 **1–2 小时**（worker 并行启动，但受节点调度限制）

deployment 的 startup probe（`initialDelaySeconds: 120`、`periodSeconds: 30`、`failureThreshold: 60`）允许每个 pod 最多 32 分钟用于模型下载和初始化。

### 步骤 2.4：确认 worker 健康

> ⚠️ **关键检查点**：在跑 benchmark 之前，**必须**确认 worker 数量与健康状态相同。worker 数量不一致会让对比结果失效。

```bash
# Quick health check - should show "8/8"
echo "Workers: $(kubectl get pods -n dynamo-bench -l nvidia.com/dynamo-component-type=worker --field-selector=status.phase=Running -o json | jq '[.items[] | select(.status.conditions[] | select(.type=="Ready" and .status=="True"))] | length')/8 ready"

# Detailed view
kubectl get pods -n dynamo-bench -l nvidia.com/dynamo-component-type=worker
```

**全部 8 个必须为 `1/1 Running` 且 Ready。** 在确认这一点之前不要继续。在拆掉 router-ON 部署 router-OFF 之后（阶段 5），再做一次同样的检查。

---

## 阶段 3：准备 benchmark 数据集

### 理解 Mooncake Toolagent Trace

本次 A/B 对比使用 [**Mooncake FAST'25 Toolagent Trace**](https://github.com/kvcache-ai/Mooncake/blob/main/FAST25-release/traces/toolagent_trace.jsonl)，由 [Mooncake AI](https://github.com/kvcache-ai/Mooncake)（USENIX FAST'25 Best Paper）发布。这是一份隐私保护的真实 LLM 推理流量数据集，来自生产中的**工具 Agent 工作负载**——AI Agent 在不断增长的对话上下文中迭代调用工具与 API。trace 含有约 59 分钟实时流量、共 **23,608 个请求**。

**为什么用 toolagent trace？** 工具 Agent 工作负载非常适合评估 KV 缓存路由，因为每个 Agent 会话都会反复调用 LLM，并且每次调用共享一个不断增长的长前缀（系统提示 + 对话历史 + 工具结果），这在请求之间产生天然的高前缀重叠。Mooncake toolagent trace 抓取到了这种现实模式，便于我们展示 router 在真实场景下的性能收益。

**数据集里有什么？** 每条 trace 条目包含：
- **timestamp：** 请求到达时间（用于真实请求时序）
- **input/output 长度：** prompt 与响应的 token 数
- **block hash ID：** 表示 KV 缓存块的加密哈希（不含用户文本，下文解释）

**示例 trace 条目（展示前缀复用）：**
```json
{"timestamp": 0, "input_length": 9013, "output_length": 3, "hash_ids": [46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63]}
{"timestamp": 0, "input_length": 6506, "output_length": 3, "hash_ids": [46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 64]}
```

这两条请求共享 46–57 号块（12 块 × 512 token = 约 6,144 token 的共享前缀）——一个工具 Agent 在同一会话中持续累积上下文。每个 hash ID 代表一个 **512 token 的块**，且 hash 同时考虑当前块与所有先前块，从而保留前缀复用模式同时保护用户隐私。**KV Smart Router** 会把带有相同 hash ID 的请求路由到同一 worker，最大化缓存命中。

如果你用 `python -m dynamo.replay` 复现该 benchmark，请把数据集的事实与回放引擎的配置分开来看：

- 对 Mooncake/toolagent trace 本身使用 `--trace-block-size 512`
- `--extra-engine-args` 中引擎的 `block_size` 应与你想模仿的 runtime 对齐（对应已发布的 vLLM 部署，通常是 `64`）

**数据集关键性质：**
- 真实时序：来自生产工具 Agent 工作负载的请求到达模式
- 高前缀重叠：59% 缓存比（[Mooncake FAST'25 论文](https://github.com/kvcache-ai/Mooncake/blob/main/FAST25-release/Mooncake-FAST25.pdf)）；同会话内迭代式工具调用产生天然前缀复用
- 隐私保护：没有真实文本——只有基于哈希的缓存块标识符
- 可复现：公开数据集允许在不同系统间公平对比

### 下载并准备数据集

```bash
# Download the Mooncake FAST'25 toolagent trace
curl -sL https://raw.githubusercontent.com/kvcache-ai/Mooncake/refs/heads/main/FAST25-release/traces/toolagent_trace.jsonl -o toolagent_trace.jsonl

# Slow down timestamps to 0.80× replay speed (~5.3 req/s instead of ~6.7 req/s)
python3 - <<'PY'
import json

with open("toolagent_trace.jsonl") as src, open("toolagent_trace_080x.jsonl", "w") as dst:
    for line in src:
        rec = json.loads(line)
        rec["timestamp"] = int(rec["timestamp"] / 0.80)
        dst.write(json.dumps(rec) + "\n")
PY

echo "Dataset ready: toolagent_trace_080x.jsonl (23,608 requests, 0.80x speed)"
```

---

## 阶段 4：搭建 benchmark 环境

### 步骤 4.1：部署 benchmark pod

创建 `benchmark-job.yaml`：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: aiperf-benchmark
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: benchmark
        image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
        securityContext:
          runAsUser: 0  # Required: apt-get and pip install need root in ephemeral benchmark pod
        command:
          - /bin/bash
          - -lc
          - |
            apt-get update -qq && apt-get install -y -qq tmux > /dev/null 2>&1
            pip install -q aiperf==0.5.0
            echo "Benchmark pod ready (tmux + aiperf installed)."
            sleep infinity
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            nvidia.com/gpu: 0
```

该 pod 启动时安装 `tmux` 与 `aiperf`，使 benchmark 可以跑在能够在 `kubectl exec` 断开后存活的 tmux 会话中。

部署：

```bash
kubectl apply -f benchmark-job.yaml -n dynamo-bench
```

等待 pod 就绪（初始化大约 1–2 分钟来安装包）：

```bash
kubectl get pods -n dynamo-bench -l job-name=aiperf-benchmark -w
```

### 步骤 4.2：把数据集复制到 benchmark pod

```bash
POD_NAME=$(kubectl get pods -n dynamo-bench -l job-name=aiperf-benchmark -o jsonpath='{.items[0].metadata.name}')
kubectl -n dynamo-bench cp toolagent_trace_080x.jsonl ${POD_NAME}:/tmp/toolagent_trace_080x.jsonl
```

---

## 阶段 5：运行 benchmark

### 步骤 5.1：基准测试 Router-ON

确认前端 service 可达（operator 会创建名为 `{deployment-name}-frontend` 的 service）：

```bash
kubectl get svc -n dynamo-bench | grep frontend
```

在 tmux 会话内启动 benchmark，使其在 `kubectl exec` 断开后仍能继续：

```bash
kubectl -n dynamo-bench exec ${POD_NAME} -- bash -c '
  tmux new-session -d -s benchmark ". /opt/dynamo/venv/bin/activate && \
    AIPERF_HTTP_CONNECTION_LIMIT=200 aiperf profile \
      -m Qwen/Qwen3-32B \
      --tokenizer Qwen/Qwen3-32B \
      --input-file /tmp/toolagent_trace_080x.jsonl \
      --custom-dataset-type mooncake_trace \
      --fixed-schedule \
      --url http://vllm-agg-router-frontend.dynamo-bench.svc.cluster.local:8000 \
      --streaming \
      --random-seed 42 \
      --workers-max 200 \
      --request-timeout-seconds 1000 \
      --profile-export-level records \
      --record-processors 8 \
      --artifact-dir /tmp/aiperf_router_on \
      --goodput \"time_to_first_token:5000 inter_token_latency:100\""
'
```

AIPerf 把运行结果写入 pod 的 `/tmp/aiperf_router_on`（汇总 JSON 与 `profile_export.jsonl`）。

### 监控 benchmark

benchmark 跑在 **tmux 会话**中，能在 `kubectl exec` 断开后存活。

附加到实时 TUI（用 **Ctrl+B 然后 D** 分离）：

```bash
kubectl -n dynamo-bench exec -it ${POD_NAME} -- tmux a -t benchmark
```

### 步骤 5.2：切换到 Router-OFF 并 benchmark

拆掉 router-ON，部署基线：

```bash
kubectl delete dynamographdeployment vllm-agg-router -n dynamo-bench
kubectl apply -f router-off-deployment.yaml -n dynamo-bench
```

等到 8/8 个 worker 重新就绪（再跑一次 [步骤 2.4](#step-24-verify-workers-are-healthy) 的健康检查），然后清理旧的 tmux 会话并启动基线 benchmark：

```bash
kubectl -n dynamo-bench exec ${POD_NAME} -- tmux kill-session -t benchmark 2>/dev/null

kubectl -n dynamo-bench exec ${POD_NAME} -- bash -c '
  tmux new-session -d -s benchmark ". /opt/dynamo/venv/bin/activate && \
    AIPERF_HTTP_CONNECTION_LIMIT=200 aiperf profile \
      -m Qwen/Qwen3-32B \
      --tokenizer Qwen/Qwen3-32B \
      --input-file /tmp/toolagent_trace_080x.jsonl \
      --custom-dataset-type mooncake_trace \
      --fixed-schedule \
      --url http://vllm-agg-no-router-frontend.dynamo-bench.svc.cluster.local:8000 \
      --streaming \
      --random-seed 42 \
      --workers-max 200 \
      --request-timeout-seconds 1000 \
      --profile-export-level records \
      --record-processors 8 \
      --artifact-dir /tmp/aiperf_router_off \
      --goodput \"time_to_first_token:5000 inter_token_latency:100\""
'
```

### 步骤 5.3：收集结果

把工件目录（或其中的 summary/export 文件）复制回本机：

```bash
kubectl -n dynamo-bench cp ${POD_NAME}:/tmp/aiperf_router_on ./aiperf_router_on
kubectl -n dynamo-bench cp ${POD_NAME}:/tmp/aiperf_router_off ./aiperf_router_off
```

每个工件目录包含：
- `profile_export_aiperf.json` —— 含聚合指标的汇总（TTFT、各分位时延、吞吐）
- `profile_export.jsonl` —— 每条请求的记录（每条完成请求一个 JSON 对象）

### 步骤 5.4：快速对比

从两份 summary 文件中抽取并对比关键指标：

```bash
python3 -c "
import json, pathlib

def load(d):
    return json.loads(pathlib.Path(d, 'profile_export_aiperf.json').read_text())

on, off = load('aiperf_router_on'), load('aiperf_router_off')

metrics = [
    ('TTFT avg (ms)',             'time_to_first_token', 'avg'),
    ('TTFT p99 (ms)',             'time_to_first_token', 'p99'),
    ('E2E Latency avg (ms)',      'request_latency',     'avg'),
    ('E2E Latency p99 (ms)',      'request_latency',     'p99'),
    ('Output Throughput (tok/s)', 'output_token_throughput', 'avg'),
]

print(f\"{'Metric':<28} {'Router-OFF':>12} {'Router-ON':>12} {'Speedup':>10}\")
print('-' * 66)
for label, key, stat in metrics:
    v_off = off.get(key, {}).get(stat, 0)
    v_on  = on.get(key, {}).get(stat, 0)
    if 'throughput' in key.lower():
        speedup = v_on / v_off if v_off else 0
    else:
        speedup = v_off / v_on if v_on else 0
    print(f'{label:<28} {v_off:>12.1f} {v_on:>12.1f} {speedup:>9.1f}x')
"
```

---

## 阶段 6：分析结果

### 关键对比指标

| 指标 | 描述 | 关注点 |
|--------|-------------|------------------|
| **Time to First Token (TTFT)** | 首个 token 到达的时延 | 越低越好；前缀复用下 KV router 可显著降低 |
| **Inter Token Latency (ITL)** | token 之间的平均时间 | 越低越好；表示生成速度 |
| **Request Latency** | 端到端总时延 | 越低越好；整体用户体验 |
| **Output Token Throughput** | 系统级每秒生成 token 数 | 越高越好；系统效率 |
| **Request Throughput** | 每秒完成的请求数 | 越高越好；容量 |

### 解读结果

**结果可能因人而异**：KV Smart Router 的提升强烈依赖于工作负载特征：

**会增加 KV router 收益的因素：**
- **高前缀重叠**（共享系统提示、模板、文档上下文）
- **长 prompt**（>2000 token），缓存能省下大量计算
- **多轮对话**带上下文延续
- **批量负载**包含相似查询

**会减少 KV router 收益的因素：**
- **唯一 prompt** 没有前缀复用
- **短 prompt**（少于 1000 token），路由开销超过收益
- **均匀分布的负载**，轮询本身已最优
- **低请求速率**，缓存被驱逐抵消收益

**在以下情况下 KV Smart Router 是有益的：**
- TTFT 提升 > 20%
- 其他指标无显著回退
- 工作负载呈现可衡量的前缀复用模式

**在以下情况下标准路由更好：**
- KV router 提升不到 10%
- 观察到时延方差变大
- 跨 worker 的负载分布比缓存亲和性更重要

### 示例对比

下面是我们用 Dynamo Operator 在完整 toolagent trace、0.80× 回放速率下的 benchmark：

| 指标 | Router-OFF（基线） | Router-ON（KV Router） | 提升 | Speedup |
|--------|----------------------|----------------------|-------------|---------|
| TTFT 平均 | 63,652 ms | 2,586 ms | **快 96%** | 24.6x |
| TTFT p99 | 332,974 ms | 17,871 ms | **快 95%** | 18.6x |
| E2E 时延平均 | 92,856 ms | 19,112 ms | **快 79%** | 4.9x |
| E2E 时延 p99 | 411,252 ms | 88,274 ms | **快 79%** | 4.7x |

在该示例中 8 个 worker 全部健康，**KV router 大幅领先**基线：
- **TTFT 快 96%** —— 用户在约 2.6s 而非约 64s 看到首 token
- **E2E 时延降低 79%** —— 请求在约 19s 而非约 93s 完成
- **TTFT p99 快 95%** —— 长尾时延从约 333s 降到约 18s

toolagent trace 由于工具 Agent 会话反复带相同上下文，前缀重叠重。没有 KV router 时，前缀重叠的请求被分散到多个 worker，造成冗余重算，并在高利用率下产生无界队列增长。启用 KV router 后，匹配前缀的请求被路由到同一 worker，缓存命中最大化，负载下时延也保持稳定。

---

## 阶段 7：清理

```bash
kubectl delete dynamographdeployment --all -n dynamo-bench
kubectl delete job aiperf-benchmark -n dynamo-bench
kubectl delete namespace dynamo-bench
```

---

## 故障排查

### 问题：Pod 卡在 Pending

**原因：** GPU 资源不足

**解决：**
```bash
# Check GPU availability
kubectl describe nodes | grep -A 10 "Allocated resources"

# Reduce worker replicas if needed
kubectl edit dynamographdeployment -n dynamo-bench
```

### 问题：ImagePullBackOff

**原因：** 版本不匹配或缺少凭证

**解决：**
```bash
# Check available versions
kubectl get pods -n dynamo-bench -o yaml | grep image:

# Update deployment YAML to match cluster version
```

### 问题：Operator 不处理 deployment

**原因：** namespace 限制

**解决：**
- 确保 Dynamo platform 已通过 Helm 安装到该 namespace
- 验证 operator 带 `--restrictedNamespace=dynamo-bench` 参数
- 查看 operator 日志：`kubectl logs -n dynamo-bench deployment/dynamo-platform-dynamo-operator-controller-manager`

### 问题：worker 无法变为 Ready

**原因：** 模型下载失败或探针配置问题

**解决：**
```bash
# Check worker logs
kubectl logs -n dynamo-bench <worker-pod-name>

# Common issues:
# - Invalid HuggingFace token
# - Network connectivity
# - Insufficient disk space for model
```

### 问题：worker 反复 CrashLoopBackOff

**原因：** Startup probe 超时——worker 在初始化完成前被杀

**症状：**
- pod 显示 "Container main failed startup probe, will be restarted"
- 日志显示 pod 被杀时模型还在下载或加载

**解决：**
本指南中的 deployment YAML 设置了 `failureThreshold: 60`，允许最多 32 分钟（`120s + 60×30s`）。如果你把它调小，或使用了更大的模型需要更多时间，请调高：

```bash
kubectl patch dynamographdeployment <deployment-name> -n dynamo-bench --type='json' \
  -p='[{"op": "replace", "path": "/spec/services/VllmDecodeWorker/extraPodSpec/mainContainer/startupProbe/failureThreshold", "value": 80}]'
```

相关 startup probe 字段：
```yaml
startupProbe:
  httpGet:
    path: /health
    port: 9090
  initialDelaySeconds: 120
  periodSeconds: 30
  timeoutSeconds: 10
  failureThreshold: 60  # 32 minutes total (120s + 60*30s); increase for larger models
```

**模型加载时间（粗略估计）：**
- Qwen3-32B：约 20–25 分钟（首次下载）
- 模型已缓存在节点上：约 2–5 分钟

### 问题：worker 健康状态不一致

**原因：** 资源紧张、镜像拉取问题或配置错误

**解决：**
```bash
# Check all worker status
kubectl get pods -n dynamo-bench -l nvidia.com/dynamo-component-type=worker

# Describe problematic pods
kubectl describe pod <pod-name> -n dynamo-bench

# Fix issues before benchmarking or results will be skewed
```

---

## 高级配置

### 测试不同模型

替换以下位置中的 `Qwen/Qwen3-32B`：
- deployment YAML 的 `args`
- AIPerf 的 `--model` 与 `--tokenizer`

### 调整 worker 数量

修改 deployment YAML 中的 `replicas: 8`。两份 deployment 必须使用相同副本数以保证公平对比。

### 使用自定义数据集

把 Mooncake trace 替换为你自己的 JSONL 文件：
- 格式：每行一个请求，需含 `timestamp` 字段
- AIPerf 通过 `--custom-dataset-type` 支持多种格式

### 解耦预填充/解码（Disaggregated Prefill/Decode）

要做更高级的测试，可以加单独的 prefill worker：

```yaml
VllmPrefillWorker:
  componentType: worker
  replicas: 2
  # ... configuration
```

---

## 最佳实践

1. **条件相同：** 在 benchmark 之前确保两份 deployment 拥有相同的 worker 数量与健康状态
2. **预热：** 在完整 benchmark 之前先跑一次小规模测试（100 个请求）来预热缓存
3. **多次运行：** benchmark 跑 3 次以上并取平均，以保证统计显著性
4. **监控 worker：** 在 benchmark 期间留意是否有 pod 重启或异常
5. **记录条件：** 记录集群状态、worker 健康情况以及任何异常
6. **配置一致：** 两次运行使用相同的 trace 文件与 AIPerf 选项

---

## 结语

本指南为 Dynamo KV Smart Router 的 A/B 测试提供了完整方法论。KV router 的有效性强烈依赖于工作负载特征——前缀重叠高的数据集收益最显著。关于 KV router 的更多调优细节，请参阅 [Tuning Guidelines](../components/router/router-configuration.md#tuning-guidelines)。

如有疑问或问题，参阅 [Dynamo 文档](https://github.com/ai-dynamo/dynamo) 或在 GitHub 提 issue。

---

## 附录：文件清单

- `router-off-deployment.yaml`：标准路由部署
- `router-on-deployment.yaml`：启用 KV router 的部署
- `benchmark-job.yaml`：AIPerf benchmark pod
- AIPerf 工件目录：每次运行的汇总 JSON 与 `profile_export.jsonl`

**仓库：** [https://github.com/ai-dynamo/dynamo](https://github.com/ai-dynamo/dynamo)
