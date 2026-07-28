<!-- SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# Frontend 扩缩测试：寻找饱和点

本指南介绍如何使用 sweep runner 寻找服务真实 vLLM 后端的 Dynamo 前端的饱和点。饱和点是延迟开始下降的请求速率——prefill 请求开始排队而非立即处理、TTFT p99 飙升、吞吐量进入平台期。

---

## 概述

测试在固定的输入序列长度下扫描递增的请求速率（`--rps`），同时保持后端处于热态（`--reset-strategy frontend`）。每个数据点是 60 秒、按受控 RPS 的 aiperf 运行。在连续失败（`--max-consecutive-fails`）后扫描会自动停止。

**你将获得：**

- 每个 RPS 的吞吐量（实际 req/s vs 目标）、TTFT p50/p99、ITL p50/p99
- 用于按管道阶段拆分的 Prometheus 前/后指标
- 便于对比的 CSV + summary

---

## 前置条件

1. **K8s namespace** 包含：
   - `hf-token-secret`（HuggingFace token）
   - `nvcrimagepullsecret`（镜像拉取凭据）
   - `model-cache` PVC（RWX，足以容纳模型权重）
   - 已下载到 PVC 的模型权重（参见下文“模型下载”）

2. **DGD 已部署**，且使用目标模型与后端。

3. **sweep_runner.py** 可从一台具备 `kubectl` 集群访问权限的机器访问。

---

## 模型下载（gpt-oss-20b 示例）

将模型下载到 PVC，排除非推理用大目录：

```bash
# 创建一个下载 Job（按需调整 image 与 namespace）
kubectl apply -n <namespace> -f - <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: model-download-gpt-oss-20b
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      imagePullSecrets:
        - name: nvcrimagepullsecret
      containers:
        - name: download
          image: nvcr.io/nvidian/dynamo-dev/biswa:vllm-runtime-1a8bce12ea
          command: ["python3", "-c"]
          args:
            - |
              import os, subprocess, sys, pathlib
              model = "openai/gpt-oss-20b"
              os.environ["HF_HOME"] = "/model-store"
              cmd = ["huggingface-cli", "download", model,
                     "--exclude", "metal/*", "--exclude", "original/*",
                     "--local-dir", "/model-store/hub/models--openai--gpt-oss-20b/snapshots/main"]
              sys.exit(subprocess.run(cmd).returncode)
          env:
            - name: HF_HOME
              value: /model-store
            - name: HF_TOKEN
              valueFrom:
                secretKeyRef:
                  name: hf-token-secret
                  key: HF_TOKEN
          volumeMounts:
            - name: model-cache
              mountPath: /model-store
      volumes:
        - name: model-cache
          persistentVolumeClaim:
            claimName: model-cache
EOF

# 监控
kubectl logs -n <namespace> -l job-name=model-download-gpt-oss-20b -f
```

---

## 部署 DGD

使用提供的 gpt-oss-20b TP=2 模板：

```bash
# 模板路径（相对仓库根）
# benchmarks/frontend/dgd/templates/vllm-gpt-oss-20b.yaml
#
# 模板中的关键设置：
#   - tensor-parallel-size 2（每个 worker 2 个 GPU）
#   - max-model-len 65536
#   - gpu-memory-utilization 0.90
#   - 用于调度的 GPU toleration

# 直接部署（按需调整值）：
kubectl apply -n <namespace> -f - <<'EOF'
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: gpt-oss-20b-bench
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        imagePullSecrets:
          - name: nvcrimagepullsecret
        mainContainer:
          image: <your-image>
          command: ["/bin/sh", "-c"]
          args: ["python3 -m dynamo.frontend --router-mode round-robin --http-port 8000"]
          env:
            - name: DYN_TOKENIZER_BACKEND
              value: "default"
            - name: DYN_PERF_DIAG
              value: "1"
            - name: HF_HOME
              value: /model-store
            - name: HF_TOKEN
              valueFrom:
                secretKeyRef:
                  name: hf-token-secret
                  key: HF_TOKEN
          volumeMounts:
            - name: model-cache
              mountPath: /model-store
        volumes:
          - name: model-cache
            persistentVolumeClaim:
              claimName: model-cache

    VllmWorker:
      componentType: worker
      replicas: 4                    # <-- 后端副本数
      extraPodSpec:
        imagePullSecrets:
          - name: nvcrimagepullsecret
        tolerations:
          - effect: NoSchedule
            key: nvidia.com/gpu
            operator: Exists
        mainContainer:
          image: <your-image>
          command: ["/bin/sh", "-c"]
          args:
            - >-
              python3 -m dynamo.vllm
              --model /model-store/hub/models--openai--gpt-oss-20b/snapshots/main
              --served-model-name openai/gpt-oss-20b
              --tensor-parallel-size 2
              --max-model-len 65536
              --gpu-memory-utilization 0.90
          env:
            - name: HF_HOME
              value: /model-store
            - name: HF_TOKEN
              valueFrom:
                secretKeyRef:
                  name: hf-token-secret
                  key: HF_TOKEN
          resources:
            limits:
              nvidia.com/gpu: "2"    # <-- TP=2 需要 2 GPU
          volumeMounts:
            - name: model-cache
              mountPath: /model-store
        volumes:
          - name: model-cache
            persistentVolumeClaim:
              claimName: model-cache
EOF

# 等待所有 pod 就绪
kubectl get pods -n <namespace> -w
```

---

## 运行饱和扫描

### 基线：HF tokenizer 的 RPS 扫描

```bash
cd benchmarks/frontend/scripts

python3 sweep_runner.py --mode k8s \
    --dgd-name gpt-oss-20b-bench \
    --namespace <namespace> \
    --endpoint gpt-oss-20b-bench-frontend:8000 \
    --model openai/gpt-oss-20b \
    --backend vllm \
    --image <your-image> \
    --tokenizers hf \
    --concurrency 200 \
    --rps 10,20,30,40,50,60,70,80,90,100 \
    --isl 6144 \
    --osl 256 \
    --benchmark-duration 60 \
    --reset-strategy frontend \
    --isolation reuse_by_deploy_key \
    --worker-replicas 4 \
    --max-consecutive-fails 2
```

**Flag 说明：**

| Flag | 取值 | 用途 |
|------|-------|---------|
| `--rps 10,20,...,100` | 扫描维度 | 每次运行针对一个固定请求速率。aiperf 用 `--request-rate` 限制提交速率。 |
| `--concurrency 200` | 高上限 | 在飞请求最大数。设得高，使 aiperf 能持续达到目标 RPS 而不被可用连接槽限制。这不是扫描维度。 |
| `--isl 6144` | 固定 ISL | 保持输入长度恒定，以隔离吞吐量扩缩。 |
| `--osl 256` | 固定 OSL | 全部运行使用一致输出长度。 |
| `--benchmark-duration 60` | 每点 60 秒 | 足够让 vLLM 调度趋稳。 |
| `--reset-strategy frontend` | 仅前端 | 在两次运行之间重置 Prometheus 计数器，但保持 vLLM worker 存活，KV 缓存和 CUDA graph 仍温热。避免每点约 90 秒的整 DGD 重启。 |
| `--isolation reuse_by_deploy_key` | 复用部署 | 由于 tokenizer=hf 不变，不需要在两次运行之间重启 DGD。仅重启前端 pod 以获得干净的指标。 |
| `--max-consecutive-fails 2` | 自动停止 | 在某 RPS 下连续失败 2 次后，跳过剩余更高的 RPS 值。 |

### 后续：FastTokens 对比

获得基线后，使用 fastokens 运行同一扫描，看看饱和点是否会发生位移：

```bash
python3 sweep_runner.py --mode k8s \
    --dgd-name gpt-oss-20b-bench \
    --namespace <namespace> \
    --endpoint gpt-oss-20b-bench-frontend:8000 \
    --model openai/gpt-oss-20b \
    --backend vllm \
    --image <your-image> \
    --tokenizers fastokens \
    --concurrency 200 \
    --rps 10,20,30,40,50,60,70,80,90,100 \
    --isl 6144 \
    --osl 256 \
    --benchmark-duration 60 \
    --reset-strategy frontend \
    --isolation reuse_by_deploy_key \
    --worker-replicas 4 \
    --max-consecutive-fails 2
```

### 拐点附近的细粒度扫描

如果基线显示饱和落在 RPS=40 与 RPS=60 之间：

```bash
python3 sweep_runner.py --mode k8s \
    ... \
    --rps 35,40,45,50,55,60 \
    --reset-strategy frontend \
    --isolation reuse_by_deploy_key
```

---

## 阅读结果

扫描在输出目录生成 `results.csv` 与 `summary.md`。

### 识别饱和点

在 CSV 中查找以下信号：

| RPS | 实际 Req/s | TTFT p50 | TTFT p99 | ITL p99 | 状态 |
|----:|-----------:|--------:|--------:|-------:|--------|
| 10 | 10.0 | 800ms | 1200ms | 30ms | ok |
| 20 | 19.8 | 850ms | 1400ms | 32ms | ok |
| 30 | 29.5 | 900ms | 2000ms | 35ms | ok |
| 40 | 38.0 | 1200ms | 5000ms | 45ms | ok —— 拐点 |
| 50 | 42.0 | 3000ms | 15000ms | 80ms | ok —— 已饱和 |
| 60 | 41.5 | 8000ms | 30000ms | 120ms | ok —— 过载 |
| 70 | -- | -- | -- | -- | fail |

**饱和指标：**

1. **实际 req/s < 目标 RPS**：系统无法持续达到要求的速率。
   RPS=50 时只达到 42 req/s。
2. **TTFT p99 飙升**：急剧上升（例如 2x-5x）意味着 prefill 请求互相排队。
3. **ITL p99 退化**：decode 吞吐量下降，因为 vLLM 调度器被并发 prefill 占用。
4. **错误/失败**：超时、OOM 或 vLLM 拒绝请求。

上例中**饱和点**约在 **RPS ~40**——这是最后一个实际吞吐能跟上目标且 TTFT p99 仍合理的速率。

### Prometheus 指标

每次运行会捕获 `frontend_metrics_pre.txt` 与 `frontend_metrics_post.txt`。饱和分析的关键指标：

- `dynamo_frontend_stage_duration_seconds{stage="preprocess"}` —— 分词时间
- `dynamo_frontend_stage_duration_seconds{stage="transport_roundtrip"}` —— 后端延迟
- `dynamo_frontend_queued_requests` —— 在 HTTP 队列中等待的请求（饱和以下应为 0）
- `dynamo_frontend_inflight_requests` —— 并发在飞请求
- `dynamo_frontend_time_to_first_token_seconds` —— TTFT 直方图分桶

---

## DGD 模板参考

`dgd/templates/vllm-gpt-oss-20b.yaml` 模板已为 gpt-oss-20b TP=2 预配置。配合 `--deploy-template` 使用：

```bash
python3 sweep_runner.py --mode k8s \
    --deploy-template benchmarks/frontend/dgd/templates/vllm-gpt-oss-20b.yaml \
    --dgd-name gpt-oss-20b-bench \
    --model /model-store/hub/models--openai--gpt-oss-20b/snapshots/main \
    --image <your-image> \
    --worker-replicas 4 \
    ...
```

模板在部署时替换以下变量：
`${DGD_NAME}`、`${IMAGE}`、`${MODEL}`、`${MODEL_NAME}`、
`${WORKER_REPLICAS}`、`${DYN_TOKENIZER_BACKEND}`、`${FRONTEND_PORT}`、
`${ROUTER_MODE}`。

---

## 调优参数

| 参数 | 推荐范围 | 备注 |
|-----------|-------------------|-------|
| `--benchmark-duration` | 60-120 秒 | 越长平均越稳定但扫描越慢 |
| `--concurrency` | 目标 RPS 上限的 2-4 倍 | 必须高到使 aiperf 能达到目标速率 |
| `--rps` | 从 10 起，连续翻倍直至失败 | 几何级数能快速找到量级 |
| `--worker-replicas` | 1-8 | 副本越多饱和点越高但 GPU 更多 |
| `--reset-strategy` | 饱和测试用 `frontend` | 干净基线 TTFT 测量用 `graph` |
| `--isolation` | 同 tokenizer 扫描用 `reuse_by_deploy_key` | 避免不必要的 DGD 重启 |
| `--max-consecutive-fails` | 2-3 | 越高在失败边界数据点越多 |
