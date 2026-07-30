---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: KV Router A/B 测试
---

本指南将引导你在 Kubernetes 集群上设置并运行 A/B 基准测试，以比较 Dynamo 的 KV 智能路由器与标准轮询路由。

## 概述

Dynamo 的 KV 智能路由器根据 KV cache 亲和性智能路由请求，从而提升具有共享提示词前缀的工作负载的性能。本指南将帮助你：

1. 部署两套相同的 Dynamo 配置：
   a. 一套用于 Qwen3-32B、包含 8 个 Worker（聚合式）且**未启用** KV 智能路由器的 vLLM 服务器
   b. 一套用于 Qwen3-32B、包含 8 个 Worker（聚合式）且**已启用** KV 智能路由器的 vLLM 服务器
2. 使用 AIPerf 运行受控基准测试
3. 比较性能指标以评估 KV router 的有效性

**前提条件：** 配备 GPU 的 Kubernetes 集群、kubectl、helm

---

## 前提条件

### 所需工具

- `kubectl`（已配置集群访问权限）
- `helm`（v3+）
- Hugging Face 账户和令牌（如果模型下载受限）
- 满足以下条件的 Kubernetes 集群：
  - GPU 节点（H100、H200 或类似型号）
  - 足够的 GPU 容量（本示例建议使用 8 个以上 GPU）
  - 已全局安装 Dynamo 平台，或者能够按命名空间安装

### 知识要求

- 了解 Kubernetes 基本概念（命名空间、Pod、Service）
- 熟悉 LLM 推理概念
- 熟练使用命令行

---

## 架构

本指南使用单个命名空间。先部署一种配置（例如启用 router），运行基准测试并将其拆除；然后部署另一种配置（禁用 router），运行相同的基准测试。

```text
┌──────────────────────────────────────────────┐
│ 命名空间：dynamo-bench                        │
│（同一时间仅启用 A 或 B 之一）                  │
│                                              │
│  部署 A：禁用 Router                          │
│    ├─ Frontend（标准路由）                     │
│    └─ 8 个 Decode Worker（每个使用 1 个 GPU）  │
│                                              │
│  部署 B：启用 Router                          │
│    ├─ Frontend（KV 智能路由器）                │
│    └─ 8 个 Decode Worker（每个使用 1 个 GPU）  │
│                                              │
│  基准测试 Pod（AIPerf + 数据集）               │
└──────────────────────────────────────────────┘
```

**主要区别：** 部署 B 在 Frontend 上设置 `DYN_ROUTER_MODE=kv`，以启用 KV cache 感知路由。

---

## 阶段 1：设置命名空间和基础设施

### 步骤 1.1：创建命名空间

```bash
kubectl create namespace dynamo-bench
```

### 步骤 1.2：创建 Hugging Face 令牌 Secret（可选）

如果要部署的模型需要 HF 令牌才能下载（Llama 系列模型需要），请将 `YOUR_HF_TOKEN` 替换为实际的 Hugging Face 令牌：

```bash
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="YOUR_HF_TOKEN" \
  -n dynamo-bench
```

### 步骤 1.3：安装 Dynamo 平台

按照 [Dynamo Kubernetes 安装指南](../../../../../docs/kubernetes/installation-guide.md)在 `dynamo-bench` 中安装平台。

> [!WARNING]
> 命名空间限制模式（`namespaceRestriction.enabled=true`）仅用于开发和测试，
> 不支持用于生产环境。

**主要配置说明：**
- 调整版本标签，使其与集群中可用的 Dynamo 版本匹配
- 如果遇到 Operator 兼容性问题（例如不支持的 MPI 参数），请咨询集群管理员或查阅 Dynamo 故障排除文档

### 步骤 1.4：验证基础设施

```bash
kubectl get pods -n dynamo-bench
```

部署 Graph 之前，Operator、etcd 和 nats Pod 应处于 Running 状态。

---

## 阶段 2：部署模型服务

### 步骤 2.1：创建部署 YAML

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
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.1
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
                        - gpu-h100-sxm  # 根据你的 GPU 节点类型进行调整
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.1
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
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.1
          env:
            - name: POD_UID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.uid
      envs:
        - name: DYN_ROUTER_MODE
          value: kv  # 主要区别：启用 KV 智能路由器
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
                        - gpu-h100-sxm  # 根据你的 GPU 节点类型进行调整
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.1
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

### 步骤 2.2：先部署启用 Router 的配置

```bash
kubectl apply -f router-on-deployment.yaml -n dynamo-bench
```

**💡 优化提示：** 每个 Worker 都会独立下载模型（每个 Pod 约需 20 分钟）。为了加快初始化速度，可添加一个采用 `ReadWriteMany` 访问模式的共享 PVC 来缓存模型。

首先，在部署所在的同一命名空间（例如 `dynamo-bench`）中创建 PVC。请使用支持 ReadWriteMany 的存储类：

```bash
kubectl get storageclass   # 选择支持 ReadWriteMany 的存储类（例如 azurefile-csi-premium、nfs、efs）
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
  storageClassName: "azurefile-csi-premium"   # 根据你的集群进行调整
  resources:
    requests:
      storage: 100Gi
```

应用该配置：`kubectl apply -f pvc-model-cache.yaml`

然后，在 DynamoGraphDeployment 的 `spec` 下添加以下内容以引用现有 PVC（并在 `VllmDecodeWorker` 下添加 `volumeMounts`）：

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

采用此配置后，首次运行时由一个 Worker 下载模型，其余 Worker 从缓存加载。其主要优势体现在重新部署时：模型会保留在 PVC 上，因此新 Pod 可从缓存加载并在约 5–10 分钟内启动，无需再次下载。

### 步骤 2.3：监控部署进度

```bash
kubectl get pods -n dynamo-bench -w
```

等待所有 Pod 进入 `Running` 状态并通过就绪探针。

**预计耗时：**
- **使用共享 PVC**（ReadWriteMany）：总计约 5–10 分钟（第一个 Worker 下载，其他 Worker 复用缓存）
- **不使用共享 PVC**：每个 Worker 需要 20–30 分钟（各 Worker 独立下载）
  - 对于 8 个 Worker：请为完整部署预留 **1–2 小时**（Worker 并行启动，但受节点调度限制）

部署的启动探针（`initialDelaySeconds: 120`、`periodSeconds: 30`、`failureThreshold: 60`）允许每个 Pod 最多用 32 分钟下载并初始化模型。

### 步骤 2.4：验证 Worker 健康状态

> [!IMPORTANT]
> 运行基准测试前，**必须**验证两次测试中的 Worker 健康状态一致。Worker 数量不一致会使比较结果无效。

```bash
# 快速健康检查——应显示“8/8”
echo "Worker：$(kubectl get pods -n dynamo-bench -l nvidia.com/dynamo-component-type=worker --field-selector=status.phase=Running -o json | jq '[.items[] | select(.status.conditions[] | select(.type=="Ready" and .status=="True"))] | length')/8 个已就绪"

# 详细视图
kubectl get pods -n dynamo-bench -l nvidia.com/dynamo-component-type=worker
```

**所有 8 个 Worker 都必须显示 `1/1 Running` 且处于 Ready 状态。** 确认之前不要继续。拆除启用 router 的配置并部署禁用 router 的配置（阶段 5）后，请重复此检查。

---

## 阶段 3：准备基准测试数据集

### 了解 Mooncake Toolagent Trace

本次 A/B 比较使用 [**Mooncake FAST'25 Toolagent Trace**](https://github.com/kvcache-ai/Mooncake/blob/main/FAST25-release/traces/toolagent_trace.jsonl)，该数据集由 [Mooncake AI](https://github.com/kvcache-ai/Mooncake) 发布（USENIX FAST'25 最佳论文）。这是一个保护隐私的真实生产环境 LLM 推理流量数据集，来源于**工具智能体工作负载**，即在保留不断增长的对话上下文的同时，迭代调用工具和 API 的 AI 智能体。该 Trace 包含 **23,608 个请求**，覆盖约 59 分钟的实时流量。

**为什么选择 toolagent Trace？** 工具智能体工作负载非常适合评估 KV cache 路由，因为每个智能体会话都会重复调用 LLM，并共享一个较长且不断增长的前缀（系统提示词 + 对话历史 + 工具结果），从而在请求之间自然形成很高的前缀重叠。Mooncake toolagent Trace 捕获了这些真实模式，可用于展示 router 在实际工作负载中的性能提升。

**数据集包含什么？** 每个 Trace 条目包含：
- **时间戳：** 请求到达的时间（用于还原真实的请求时序）
- **输入/输出长度：** 提示词和响应中的 token 数量
- **Block 哈希 ID：** 表示 KV cache block 的加密哈希（不含用户文本；详见下文）

**Trace 条目示例（展示前缀复用）：**

```text
{"timestamp": 0, "input_length": 9013, "output_length": 3, "hash_ids": [46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63]}
{"timestamp": 0, "input_length": 6506, "output_length": 3, "hash_ids": [46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 64]}
```

这两个请求共享 block 46–57（12 个 block × 512 个 token = 约 6,144 个 token 的共享前缀），表示工具智能体携带累积上下文继续同一会话。每个哈希 ID 表示一个 **512-token block**，并且哈希同时包含当前 block 及其之前的所有 block，在保护用户隐私的同时保留前缀复用模式。**KV 智能路由器**会将具有匹配哈希 ID 的请求路由到同一 Worker，从而最大化 cache 命中率。

如果使用 `python -m dynamo.replay` 复现此基准测试，请将该数据集的特性与
Replay Engine 配置区分开：

- 对 Mooncake/toolagent Trace 本身使用 `--trace-block-size 512`
- `--extra-engine-args` 中 Engine 的 `block_size` 应与要模拟的 Runtime 保持一致
  （对于本文使用的 vLLM 部署，通常为 `64`）

**数据集的主要特性：**
- ✅ **真实时序：** 来自生产环境工具智能体工作负载的请求到达模式
- ✅ **高前缀重叠：** cache 比率为 59%（参见 [Mooncake FAST'25 论文](https://github.com/kvcache-ai/Mooncake/blob/main/FAST25-release/Mooncake-FAST25.pdf)）；会话中的迭代工具调用会自然产生前缀复用
- ✅ **保护隐私：** 不含实际文本，仅包含基于哈希的 cache block 标识符
- ✅ **可复现：** 公开数据集支持不同系统之间的公平比较

### 下载并准备数据集

```bash
# 下载 Mooncake FAST'25 toolagent Trace
curl -sL https://raw.githubusercontent.com/kvcache-ai/Mooncake/refs/heads/main/FAST25-release/traces/toolagent_trace.jsonl -o toolagent_trace.jsonl

# 将时间戳减速至 0.80 倍 Replay 速度（约 5.3 req/s，而非约 6.7 req/s）
python3 - <<'PY'
import json

with open("toolagent_trace.jsonl") as src, open("toolagent_trace_080x.jsonl", "w") as dst:
    for line in src:
        rec = json.loads(line)
        rec["timestamp"] = int(rec["timestamp"] / 0.80)
        dst.write(json.dumps(rec) + "\n")
PY

echo "数据集已就绪：toolagent_trace_080x.jsonl（23,608 个请求，0.80 倍速度）"
```

---

## 阶段 4：设置基准测试环境

### 步骤 4.1：部署基准测试 Pod

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
        image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.1
        securityContext:
          runAsUser: 0  # 必需：临时基准测试 Pod 中的 apt-get 和 pip install 需要 root 权限
        command:
          - /bin/bash
          - -lc
          - |
            apt-get update -qq && apt-get install -y -qq tmux > /dev/null 2>&1
            pip install -q aiperf==0.10.0
            echo "基准测试 Pod 已就绪（已安装 tmux + aiperf）。"
            sleep infinity
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            nvidia.com/gpu: 0
```

此 Pod 启动时会安装 `tmux` 和 `aiperf`，因此基准测试可在 tmux 会话中运行，并在 `kubectl exec` 连接断开后继续执行。

部署：

```bash
kubectl apply -f benchmark-job.yaml -n dynamo-bench
```

等待 Pod 就绪（初始化并安装软件包约需 1–2 分钟）：

```bash
kubectl get pods -n dynamo-bench -l job-name=aiperf-benchmark -w
```

### 步骤 4.2：将数据集复制到基准测试 Pod

```bash
POD_NAME=$(kubectl get pods -n dynamo-bench -l job-name=aiperf-benchmark -o jsonpath='{.items[0].metadata.name}')
kubectl -n dynamo-bench cp toolagent_trace_080x.jsonl ${POD_NAME}:/tmp/toolagent_trace_080x.jsonl
```

---

## 阶段 5：运行基准测试

### 步骤 5.1：对启用 Router 的配置进行基准测试

验证 Frontend Service 是否可访问（Operator 会创建名为 `{deployment-name}-frontend` 的 Service）：

```bash
kubectl get svc -n dynamo-bench | grep frontend
```

在 tmux 会话中启动基准测试，使其在 `kubectl exec` 连接断开后继续运行：

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

AIPerf 将运行结果写入 Pod 上的 `/tmp/aiperf_router_on`（摘要 JSON 和 `profile_export.jsonl`）。

### 监控基准测试

基准测试在 **tmux 会话**中运行，因此会在 `kubectl exec` 连接断开后继续执行。

连接到实时 TUI（按 **Ctrl+B，然后按 D** 断开）：

```bash
kubectl -n dynamo-bench exec -it ${POD_NAME} -- tmux a -t benchmark
```

### 步骤 5.2：切换到禁用 Router 的配置并进行基准测试

拆除启用 router 的配置并部署基线：

```bash
kubectl delete dynamographdeployment vllm-agg-router -n dynamo-bench
kubectl apply -f router-off-deployment.yaml -n dynamo-bench
```

再次等待 8/8 个 Worker 进入 Ready 状态（重新运行[步骤 2.4](#步骤-24验证-worker-健康状态)中的健康检查），然后清理之前的 tmux 会话并启动基线测试：

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

将产物目录（或其中的摘要/导出文件）复制到本机：

```bash
kubectl -n dynamo-bench cp ${POD_NAME}:/tmp/aiperf_router_on ./aiperf_router_on
kubectl -n dynamo-bench cp ${POD_NAME}:/tmp/aiperf_router_off ./aiperf_router_off
```

每个产物目录包含：
- `profile_export_aiperf.json`——包含聚合指标（TTFT、延迟百分位数、吞吐量）的摘要
- `profile_export.jsonl`——每个请求的记录（每个已完成请求对应一个 JSON 对象）

### 步骤 5.4：快速比较

从两个摘要文件中提取并比较关键指标：

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

### 要比较的关键指标

| 指标 | 说明 | 关注点 |
|--------|-------------|------------------|
| **首 token 时间（TTFT）** | 第一个 token 到达前的延迟 | 越低越好；存在前缀复用时，KV router 可能降低此指标 |
| **Token 间延迟（ITL）** | token 之间的平均时间 | 越低越好；表示生成速度 |
| **请求延迟** | 端到端总延迟 | 越低越好；反映整体用户体验 |
| **输出 Token 吞吐量** | 整个系统每秒生成的 token 数 | 越高越好；反映系统效率 |
| **请求吞吐量** | 每秒完成的请求数 | 越高越好；反映系统容量 |

### 解读结果

**实际结果可能有所不同**：KV 智能路由器带来的改进高度依赖工作负载特征：

**可提升 KV router 收益的因素：**
- **高前缀重叠**（共享的系统提示词、模板和文档上下文）
- **长提示词**（超过 2000 个 token），此时缓存可节省大量计算
- **多轮对话**，上下文会延续到后续请求
- **批处理工作负载**，查询内容相似

**会降低 KV router 收益的因素：**
- **独立提示词**，没有前缀复用
- **短提示词**（少于 1000 个 token），路由开销超过收益
- **负载分布均匀**，轮询路由已经达到最优
- **请求速率低**，cache 淘汰抵消了收益

**以下情况下 KV 智能路由器可带来收益：**
- TTFT 改进超过 20%
- 其他指标未明显下降
- 工作负载呈现可度量的前缀复用模式

**以下情况下标准路由更好：**
- KV router 的改进低于 10%
- 观察到延迟方差增大
- Worker 间的负载分布比 cache 亲和性更重要

### 比较示例

以下结果来自使用完整 toolagent Trace、以 0.80 倍 Replay 速度运行的 Dynamo Operator 基准测试：

| 指标 | 禁用 Router（基线） | 启用 Router（KV Router） | 改进 | 加速比 |
|--------|----------------------|----------------------|-------------|---------|
| TTFT 平均值 | 63,652 ms | 2,586 ms | **快 96%** | 24.6x ✅ |
| TTFT p99 | 332,974 ms | 17,871 ms | **快 95%** | 18.6x ✅ |
| 端到端延迟平均值 | 92,856 ms | 19,112 ms | **快 79%** | 4.9x ✅ |
| 端到端延迟 p99 | 411,252 ms | 88,274 ms | **快 79%** | 4.7x ✅ |

在本示例中，所有 8 个 Worker 均处于健康状态，**KV router 的性能显著优于**基线：
- **TTFT 快 96%**——用户看到第一个 token 的时间从约 64 秒缩短至约 2.6 秒
- **端到端延迟降低 79%**——请求完成时间从约 93 秒缩短至约 19 秒
- **TTFT p99 快 95%**——尾延迟从约 333 秒降至约 18 秒

toolagent Trace 中的工具智能体会话包含大量重复上下文，因此具有很高的前缀重叠。未使用 KV router 时，前缀重叠的请求分散到不同 Worker，导致重复计算，并在高利用率下造成队列无限增长。使用 KV router 后，前缀匹配的请求会被路由到同一 Worker，从而最大化 cache 命中率，并使高负载下的延迟保持稳定。

---

## 阶段 7：清理

```bash
kubectl delete dynamographdeployment --all -n dynamo-bench
kubectl delete job aiperf-benchmark -n dynamo-bench
kubectl delete namespace dynamo-bench
```

---

## 故障排除

### 问题：Pod 一直处于 Pending 状态

**原因：** GPU 资源不足

**解决方案：**
```bash
# 检查 GPU 可用情况
kubectl describe nodes | grep -A 10 "Allocated resources"

# 必要时减少 Worker 副本数
kubectl edit dynamographdeployment -n dynamo-bench
```

### 问题：出现 ImagePullBackOff 错误

**原因：** 版本不匹配或缺少凭证

**解决方案：**
```bash
# 检查可用版本
kubectl get pods -n dynamo-bench -o yaml | grep image:

# 更新部署 YAML 以匹配集群版本
```

### 问题：Operator 未处理部署

**原因：** 命名空间限制

**解决方案：**
- 确保已使用 Helm 在该命名空间中安装 Dynamo 平台
- 验证 Operator 是否具有 `--restrictedNamespace=dynamo-bench` 参数
- 检查 Operator 日志：`kubectl logs -n dynamo-bench deployment/dynamo-platform-dynamo-operator-controller-manager`

### 问题：Worker 未进入 Ready 状态

**原因：** 模型下载失败或探针配置问题

**解决方案：**
```bash
# 检查 Worker 日志
kubectl logs -n dynamo-bench <worker-pod-name>

# 常见问题：
# - Hugging Face 令牌无效
# - 网络连接问题
# - 模型所需磁盘空间不足
```

### 问题：Worker 在 CrashLoopBackOff 状态下反复重启

**原因：** 启动探针超时——Worker 在完成初始化前被终止

**症状：**
- Pod 显示“Container main failed startup probe, will be restarted”
- 日志显示 Pod 被终止时模型仍在下载或加载

**解决方案：**
本指南中的部署 YAML 将 `failureThreshold` 设为 `60`，允许最长 32 分钟（`120s + 60×30s`）。如果你降低了此值，或者使用的较大模型需要更多时间，请增大该值：

```bash
kubectl patch dynamographdeployment <deployment-name> -n dynamo-bench --type='json' \
  -p='[{"op": "replace", "path": "/spec/services/VllmDecodeWorker/extraPodSpec/mainContainer/startupProbe/failureThreshold", "value": 80}]'
```

相关启动探针字段如下：
```yaml
startupProbe:
  httpGet:
    path: /health
    port: 9090
  initialDelaySeconds: 120
  periodSeconds: 30
  timeoutSeconds: 10
  failureThreshold: 60  # 总计 32 分钟（120s + 60*30s）；对于较大模型请增大此值
```

**模型加载时间（近似值）：**
- Qwen3-32B：约 20–25 分钟（首次下载）
- 节点上已有缓存模型：约 2–5 分钟

### 问题：Worker 健康状态不一致

**原因：** 资源限制、镜像拉取问题或配置错误

**解决方案：**
```bash
# 检查所有 Worker 的状态
kubectl get pods -n dynamo-bench -l nvidia.com/dynamo-component-type=worker

# 查看存在问题的 Pod 的详细信息
kubectl describe pod <pod-name> -n dynamo-bench

# 在进行基准测试前解决问题，否则结果会出现偏差
```

---

## 高级配置

### 测试不同模型

在以下位置将 `Qwen/Qwen3-32B` 替换为你的模型：
- 部署 YAML 的 `args` 部分
- AIPerf 的 `--model` 和 `--tokenizer` 参数

### 调整 Worker 数量

修改部署 YAML 中的 `replicas: 8`。为确保公平比较，请让两套部署使用相同的数量。

### 使用自定义数据集

将 Mooncake Trace 替换为你自己的 JSONL 文件：
- 格式：每行一个请求，并包含 `timestamp` 字段
- AIPerf 通过 `--custom-dataset-type` 支持多种格式

### 分离式 Prefill/Decode

如需进行高级测试，请添加独立的 Prefill Worker：

```yaml
VllmPrefillWorker:
  componentType: worker
  replicas: 2
  # ... 配置
```

---

## 最佳实践

1. **条件一致：** 进行基准测试前，确保两套部署的 Worker 数量和健康状态相同
2. **预热：** 在完整基准测试前运行一个小型测试（100 个请求）以预热 cache
3. **多次运行：** 运行基准测试至少 3 次并对结果取平均值，以确保统计显著性
4. **监控 Worker：** 基准测试期间留意 Pod 是否重启或出现其他问题
5. **记录条件：** 记录集群状态、Worker 健康状态和所有异常情况
6. **配置一致：** 两次运行使用相同的 Trace 文件和 AIPerf 选项

---

## 总结

本指南提供了一套完整的方法，用于对 Dynamo 的 KV 智能路由器进行 A/B 测试。KV router 的有效性高度依赖工作负载特征；具有高前缀重叠的数据集将获得最大收益。有关 KV router 调优的更多信息，请参阅[调优指南](../../../../../docs/components/router/router-configuration.md#tuning-guidelines)。

如有疑问或遇到问题，请查阅 [Dynamo 文档](https://github.com/ai-dynamo/dynamo)，或在 GitHub 上提交 Issue。

---

## 附录：文件参考

- `router-off-deployment.yaml`：标准路由部署
- `router-on-deployment.yaml`：启用 KV router 的部署
- `benchmark-job.yaml`：AIPerf 基准测试 Pod
- AIPerf 产物目录：每次运行的摘要 JSON 和 `profile_export.jsonl`

**仓库：** [https://github.com/ai-dynamo/dynamo](https://github.com/ai-dynamo/dynamo)

