<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Global Planner 示例

演示 **GlobalPlanner**——一个集中式的扩缩执行层，跨多个 DGD 强制执行共享的扩缩策略。

## 示例 manifest

| 文件 | 模式 | 后端 | 描述 |
|------|---------|---------|-------------|
| `global-planner-gpu-budget.yaml` | 多模型，GPU 预算 | vLLM | 2 个独立模型 DGD + 1 个使用 `--max-total-gpus` 的控制 DGD |
| `global-planner-vllm-test.yaml` | 单端点、多池 | vLLM | 1 个 Frontend + GlobalRouter + GlobalPlanner，2 个预填充池（TP1、TP2）+ 1 个解码池 |
| `global-planner-mocker-test.yaml` | 单端点、多池 | Mocker | 同上，但使用 Mocker worker；GlobalPlanner 处于 `--no-operation` 模式 |

## 部署模式

### 模式 1：多模型 + GPU 预算（`global-planner-gpu-budget.yaml`）

多个独立 DGD，每个为一个不同的模型提供服务，并各自带前端。一个共享的 GlobalPlanner 强制执行集群级 GPU 上限。

```
DGD gp-ctrl:    GlobalPlanner (--max-total-gpus)
DGD model-a:    Frontend + VllmPrefillWorker + VllmDecodeWorker + Planner  (MODEL_A)
DGD model-b:    Frontend + VllmPrefillWorker + VllmDecodeWorker + Planner  (MODEL_B)
```

- 不需要 GlobalRouter——每个模型有自己独立的端点。
- 每个 DGD 的本地 planner 通过 `environment: "global-planner"` 把扩缩委派给 GlobalPlanner。
- GlobalPlanner 会拒绝任何会让总 GPU 数超过上限的扩缩请求。

### 模式 2：单端点、多池（`global-planner-vllm-test.yaml`）

针对单个模型的一个公共端点，背后由多个特化的池支撑。GlobalRouter 为每个请求选择最佳池。

```
DGD gp-ctrl:      Frontend + GlobalRouter + GlobalPlanner
DGD gp-prefill-0: LocalRouter + VllmPrefillWorker (TP1) + Planner
DGD gp-prefill-1: LocalRouter + VllmPrefillWorker (TP2) + Planner
DGD gp-decode-0:  LocalRouter + VllmDecodeWorker  (TP1) + Planner
```

- GlobalRouter 通过 (ISL, TTFT 目标) 路由 prefill 请求；通过 (上下文长度, ITL 目标) 路由 decode 请求。
- 每个池的 planner 把扩缩委派给 GlobalPlanner。

## 前置条件

- 已安装 Dynamo Kubernetes Platform（参见 [Kubernetes Quickstart](../../docs/kubernetes/README.md)）
- 集群 Prometheus 通过 PodMonitor 抓取 router 指标
- HuggingFace token secret：
  ```bash
  kubectl create secret generic hf-token-secret \
    --from-literal=HF_TOKEN=<your-token> -n ${K8S_NAMESPACE}
  ```
- 一个 ReadWriteMany StorageClass，用于共享模型缓存 PVC

## 部署

所有 manifest 使用 `envsubst` 进行配置。设置必需变量并 apply：

### GPU Budget 示例

```bash
export K8S_NAMESPACE=my-ns
export DYNAMO_IMAGE=<dynamo-image>
export DYNAMO_VLLM_IMAGE=<vllm-image>
export STORAGE_CLASS_NAME=<rwx-storage-class>
export MODEL_A=meta-llama/Llama-3.1-8B-Instruct
export MODEL_B=Qwen/Qwen3-8B
export MAX_TOTAL_GPUS=8

envsubst < global-planner-gpu-budget.yaml | kubectl apply -n ${K8S_NAMESPACE} -f -
```

### 单端点 vLLM 示例

```bash
export K8S_NAMESPACE=my-ns
export DYNAMO_IMAGE=<dynamo-image>
export DYNAMO_VLLM_IMAGE=<vllm-image>
export STORAGE_CLASS_NAME=<rwx-storage-class>
export MODEL_NAME=meta-llama/Llama-3.1-8B-Instruct

envsubst < global-planner-vllm-test.yaml | kubectl apply -n ${K8S_NAMESPACE} -f -
```

### Mocker 示例（无 GPU）

```bash
export K8S_NAMESPACE=my-ns
export DYNAMO_IMAGE=<dynamo-image>

envsubst < global-planner-mocker-test.yaml | kubectl apply -n ${K8S_NAMESPACE} -f -
```

## 验证

```bash
# 查看 DGD 状态
kubectl get dgd -n ${K8S_NAMESPACE}

# 查看 pod
kubectl get pods -n ${K8S_NAMESPACE}

# 跟踪 GlobalPlanner 日志查看扩缩请求
kubectl logs -n ${K8S_NAMESPACE} \
  -l nvidia.com/dynamo-component=GlobalPlanner -f
```

## 清理

```bash
envsubst < <manifest>.yaml | kubectl delete -n ${K8S_NAMESPACE} -f -
```

## SLA Planner 配置

每个池的本地 planner 通过传入 `--config` 的 JSON blob 配置。对 GlobalPlanner 委派的关键字段：

| 字段 | 描述 |
|-------|-------------|
| `environment` | `"global-planner"`——把扩缩委派给 GlobalPlanner |
| `global_planner_namespace` | 控制 DGD 的 Dynamo namespace（例如 `${K8S_NAMESPACE}-gp-ctrl`） |
| `mode` | `"disagg"`、`"prefill"` 或 `"decode"` |
| `throughput_metrics_source` | 多 DGD 设置使用 `"router"`（从 Prometheus 读取 `dynamo_component_router_*`） |
| `max_gpu_budget` | 每池的 GPU 上限（`-1` = 无限，交由 GlobalPlanner） |

## GlobalPlanner Flag

| Flag | 描述 |
|------|-------------|
| `--max-total-gpus N` | 拒绝任何会使所有受管 DGD 总 GPU 数超过 N 的请求。`0` = 不允许 GPU 扩缩，`-1`（默认）= 无限 |
| `--managed-namespaces NS...` | 仅接受来自所列 Dynamo namespace 的扩缩请求（默认：全部接受）。参见下文的 *管理模式* |
| `--no-operation` | 仅记录扩缩请求而不执行（用于 dry-run 测试） |

### 管理模式

GlobalPlanner 根据是否设置了 `--managed-namespaces` 运行在以下两种模式之一：

- **显式模式**（提供了 `--managed-namespaces`）：仅所列 Dynamo namespace 被授权发送扩缩请求，且只有它们对应的 DGD 计入 GPU 预算。DGD 名按 operator 约定 `DYN_NAMESPACE = {k8s_namespace}-{dgd_name}` 由 Dynamo namespace 派生。
- **隐式模式**（无 `--managed-namespaces`）：接受任意调用方，且 Kubernetes namespace 中所有 DGD 都计入 GPU 预算。

## Namespace 约定

Dynamo operator 由 Kubernetes namespace 与 DGD 名构造每个 Dynamo namespace：
- K8s namespace：`my-ns`，DGD 名：`gp-ctrl`
- Dynamo namespace：`my-ns-gp-ctrl`

这就是为什么 planner 配置和 router 端点使用完整的 `${K8S_NAMESPACE}-<dgd-name>` 路径。

## 延伸阅读

- [Global Planner 部署指南](../../docs/components/planner/global-planner.md)
- [Global Planner README](../../components/src/dynamo/global_planner/README.md)
- [Planner 配置指南](../../docs/components/planner/planner-guide.md)
- [Global Router README](../../components/src/dynamo/global_router/README.md)
