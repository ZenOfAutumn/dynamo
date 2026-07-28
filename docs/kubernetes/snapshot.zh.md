---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Snapshot
---

> ⚠️ **实验性特性**：Dynamo Snapshot 目前为预览阶段，可能仅在部分集群环境下可用。`snapshot-agent` DaemonSet 以特权模式运行以执行 CRIU 操作。详情见[限制](#限制)。

**Dynamo Snapshot** 是一套基础设施，用于在 Kubernetes 中借助 CRIU（Checkpoint/Restore in Userspace）与 NVIDIA `cuda-checkpoint` 工具实现 GPU 应用的快速启动。常规流程：

1. 启动一次 Worker，并对其初始化完成后的状态做 checkpoint
2. 将该 checkpoint 存放到命名空间本地的 snapshot 卷上
3. 后续 Worker 从该 checkpoint 恢复，而不必再次冷启动

| 启动类型 | 时间 | 发生了什么 |
|--------------|------|--------------|
| **冷启动（Cold Start）** | ~1 分钟 | 下载模型、加载到 GPU、初始化引擎 |
| **温启动（Warm Start，从 checkpoint 恢复）** | ~10 秒 | 从已就绪的 checkpoint 目录恢复 |

> ⚠️ 恢复时间取决于存储带宽、GPU 型号以及恢复是否仍在同一节点上。

## 前置条件

- x86_64（`amd64`）GPU 节点
- 目标 GPU 节点上的 NVIDIA 驱动 580.xx 或更高（如果测试多 GPU snapshot 需 590.xx 或更高）
- 当前仅支持 vLLM 或 SGLang 后端
- Checkpoint 存储。`ReadWriteMany` 是跨节点或并发多节点访问最稳妥的默认值，但 `podMount` 模式也可以使用合适的 `ReadWriteOnce` 存储用于顺序的 checkpoint/restore 流程。
- **CRI-O / OpenShift：** 在 snapshot chart 中设置 `runtime.type=crio`（在 OpenShift 上还需 `openshift.enabled=true`）。默认值面向 containerd；socket 与 Helm 参数详见 chart 的 README。

## 通过 `DynamoCheckpoint` CR 快速开始

1. 构建占位（placeholder）镜像
2. 安装 snapshot chart
3. 创建 `DynamoCheckpoint` 并等待其就绪
4. 部署一个使用对应 `checkpointRef` 进行恢复的 `DynamoGraphDeployment`

### 1. 构建并推送占位镜像

启用 snapshot 的 Worker 必须使用一个占位镜像，它在常规 runtime 镜像之外封装恢复所需的工具。如果还没有，请构建并推送到集群可拉取的镜像仓库：

```bash
export RUNTIME_IMAGE=registry.example.com/dynamo/vllm-runtime:1.0.0
export PLACEHOLDER_IMAGE=registry.example.com/dynamo/vllm-placeholder:1.0.0

cd deploy/snapshot

make docker-build-placeholder \
  PLACEHOLDER_BASE_IMG="${RUNTIME_IMAGE}" \
  PLACEHOLDER_IMG="${PLACEHOLDER_IMAGE}"

make docker-push-placeholder \
  PLACEHOLDER_IMG="${PLACEHOLDER_IMAGE}"
```

占位镜像保留与常规 runtime 相同的入口/命令契约，并补充 `criu`、`cuda-checkpoint`、`nsrestore` 等用于 checkpoint 与恢复的工具。

如需基于自定义 CRIU 分支或 ref 构建任一 snapshot 镜像，可通过 `make` 传入 `CRIU_REPO` 与 `CRIU_REF`。如果未设置，则使用 Dockerfile 默认值。

```bash
make docker-build-agent \
  IMG=registry.example.com/dynamo/snapshot-agent:1.0.0 \
  CRIU_REPO="${YOUR_CRIU_REPO}" \
  CRIU_REF="branch-or-sha"

make docker-build-placeholder \
  PLACEHOLDER_BASE_IMG="${RUNTIME_IMAGE}" \
  PLACEHOLDER_IMG="${PLACEHOLDER_IMAGE}" \
  CRIU_REPO="${YOUR_CRIU_REPO}" \
  CRIU_REF="branch-or-sha"
```

### 2. 在平台中启用 checkpoint 并验证

无论是安装还是升级 `dynamo-platform`，Operator 仅需启用 checkpoint：

```yaml
dynamo-operator:
  checkpoint:
    enabled: true
```

如平台已安装，验证 Operator 配置中包含 checkpoint 块：

```bash
OPERATOR_CONFIG=$(kubectl get deploy -n "${PLATFORM_NAMESPACE}" \
  -l app.kubernetes.io/name=dynamo-operator,app.kubernetes.io/component=manager \
  -o jsonpath='{.items[0].spec.template.spec.volumes[?(@.name=="operator-config")].configMap.name}')

kubectl get configmap "${OPERATOR_CONFIG}" -n "${PLATFORM_NAMESPACE}" \
  -o jsonpath='{.data.config\.yaml}' | sed -n '/^checkpoint:/,/^[^[:space:]]/p'
```

确认渲染出的配置中包含 `enabled: true`。

### 3. 安装 snapshot chart

对默认的命名空间本地模式，请在每个工作负载所在命名空间中安装 snapshot chart。chart 会在该命名空间中创建 PVC 与 agent：

```bash
helm upgrade --install snapshot ./deploy/helm/charts/snapshot \
  --namespace ${NAMESPACE} \
  --create-namespace \
  --set storage.pvc.create=true
```

在默认的 `agentMount` 模式下，snapshot-agent DaemonSet 直接挂载 checkpoint PVC。在多节点 GPU 集群中这意味着多个节点上的 agent Pod 可能挂载同一 PVC，因此 PVC 通常需要 `ReadWriteMany`。chart 默认正是该模式。如果集群没有默认存储类，请同时设置 `storage.pvc.storageClass`。

如果你要复用已有的 checkpoint PVC，不要设置 `storage.pvc.create=true`；而是用 `storage.pvc.create=false` 安装 chart 并设置 `storage.pvc.name`。

CRI-O 或 OpenShift：可追加 `--set runtime.type=crio`，在 OpenShift 上还需 `--set openshift.enabled=true`（详见 `deploy/helm/charts/snapshot/README.md`）。

如果你希望集群只跑一个特权 snapshot agent，而不是为每个工作负载命名空间都跑一份 DaemonSet，请把 chart 安装到一个基础设施命名空间中。该模式下 chart 不会创建工作负载 PVC；Dynamo Operator 会在每个命名空间创建本地 PVC，或验证它已存在：

```bash
helm upgrade --install snapshot ./deploy/helm/charts/snapshot \
  --namespace dynamo-system \
  --create-namespace \
  --set storage.accessMode=podMount \
  --set storage.pvc.create=false \
  --set rbac.namespaceRestricted=false
```

要让 Operator 在每个使用 checkpoint/restore 的命名空间中创建工作负载 PVC，请将 Operator 配置为 `create: true`：

```yaml
dynamo-operator:
  checkpoint:
    enabled: true
    storage:
      type: pvc
      pvc:
        pvcName: snapshot-pvc
        basePath: /checkpoints
        create: true
        size: 1Ti
        storageClassName: ""
        accessMode: ReadWriteMany
```

chart 与 Operator 在此处使用不同的配置接口：snapshot chart 中 PVC 名称为 `storage.pvc.name`，而 Operator 配置字段为 `checkpoint.storage.pvc.pvcName`。

这是 `agentMount` 与 `podMount` 之间的关键差异：`podMount` 不再要求 snapshot-agent DaemonSet 在每个 GPU 节点上挂载 checkpoint PVC。只有当前活跃的 checkpoint/restore 工作负载 Pod 会挂载 PVC，agent 通过该 Pod 的 mount 命名空间访问。`ReadWriteMany` 仍是 Operator 管理下最稳妥的默认值，尤其是当多个 checkpoint/restore Pod 可能并发访问同一 PVC，或恢复调度可能跨节点时。如果后端能够将卷挂载到运行活跃工作负载 Pod 的节点上，合适的 `ReadWriteOnce` 存储类仍可用于顺序的 `podMount` checkpoint/restore 流程。

`podMount` 依赖目标容器在 agent 解析 `/host/proc/<pid>/root/<basePath>` 期间保持存活。如果在 checkpoint/restore 设置过程中容器退出或重启、运行时无法暴露稳定的 host PID、或节点安全策略禁止遍历 host proc，agent 会失败或跳过本次尝试，Kubernetes/Operator 在新容器可用后必须再次进行调谐。

要使用已有 PVC 而非新建，请省略 `create` 或将其设为 `false`。如果指定的 PVC 在工作负载命名空间中不存在，Operator 会带明确错误信息使调谐失败。

确认 DaemonSet 已就绪。在 checkpoint 或 restore 工作负载完成调谐后，验证工作负载命名空间的 PVC：

```bash
kubectl rollout status daemonset/snapshot-agent -n dynamo-system
kubectl get pods -n dynamo-system -l app.kubernetes.io/component=snapshot-agent -o wide
kubectl get pvc snapshot-pvc -n ${NAMESPACE}
```

### 4. 创建 `DynamoCheckpoint`

checkpoint Job 的 Pod 模板应与你想要 checkpoint 的 Worker 容器一致。对于 snapshot 流程，关键点是 checkpoint 身份信息、一个名为 `main` 的容器、以及占位镜像；其余 Pod 模板字段应与正常 Worker 配置保持一致。允许出现额外容器，但只有 `main` 会被 checkpoint。

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoCheckpoint
metadata:
  name: qwen3-06b-bf16
spec:
  identity:
    model: Qwen/Qwen3-0.6B
    backendFramework: vllm
    tensorParallelSize: 1
    dtype: bfloat16
    maxModelLen: 2048

  job:
    activeDeadlineSeconds: 3600
    podTemplateSpec:
      spec:
        ...
        containers:
          - name: main
            image: registry.example.com/dynamo/vllm-placeholder:1.0.0
            ...
```

GMS + Snapshot 支持当前是关闭的。

完整可运行示例参见 [deploy/operator/config/samples/nvidia.com_v1alpha1_dynamocheckpoint.yaml](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/config/samples/nvidia.com_v1alpha1_dynamocheckpoint.yaml)。

apply 它：

```bash
kubectl apply -f qwen3-checkpoint.yaml -n ${NAMESPACE}
```

### 5. 等待 checkpoint 就绪

```bash
kubectl get dckpt -n ${NAMESPACE} \
  -o custom-columns=NAME:.metadata.name,HASH:.status.identityHash,PHASE:.status.phase

kubectl wait \
  --for=jsonpath='{.status.phase}'=Ready \
  dynamocheckpoint/qwen3-06b-bf16 \
  -n ${NAMESPACE} \
  --timeout=30m
```

常用的 status 字段：

- `status.phase`：高层生命周期（`Pending`、`Creating`、`Ready`、`Failed`）
- `status.identityHash`：`spec.identity` 的确定性哈希
- `status.jobName`：checkpoint Job 名称
- `status.createdAt`：checkpoint 就绪时记录的时间戳
- `status.message`：进度或失败的详细信息（如有）

### 6. 部署一个从 `checkpointRef` 恢复的 `DynamoGraphDeployment`

当 checkpoint 进入 `Ready` 后，可显式从中恢复 Worker：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-checkpointref-demo
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: registry.example.com/dynamo/vllm-runtime:1.0.0

    VllmDecodeWorker:
      componentType: worker
      replicas: 1
      checkpoint:
        enabled: true
        checkpointRef: qwen3-06b-bf16
      extraPodSpec:
        mainContainer:
          image: registry.example.com/dynamo/vllm-placeholder:1.0.0
          ...
        ...
```

apply 它：

```bash
kubectl apply -f vllm-checkpointref-demo.yaml -n ${NAMESPACE}
kubectl get pods -n ${NAMESPACE} -w
```

`VllmDecodeWorker` Pod 应从就绪的 checkpoint 恢复，而非新建一个。

## DGD Auto 流程

`checkpointRef` 是最显式的路径。`mode: Auto` 是更高层的路径：Operator 会计算 checkpoint 身份哈希、查找等价的 `DynamoCheckpoint`，仅当不存在匹配 checkpoint 时才创建一个。如果已存在同身份的 `DynamoCheckpoint`，Auto 模式会复用它。如果尚无匹配 checkpoint，第一个 Worker 会冷启动，Operator 在后台创建 checkpoint。

```yaml
checkpoint:
  enabled: true
  mode: Auto
  identity:
    model: Qwen/Qwen3-0.6B
    backendFramework: vllm
    tensorParallelSize: 1
    dtype: bfloat16
    maxModelLen: 2048
```

在 `DynamoGraphDeployment` 中形如：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-auto-demo
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: registry.example.com/dynamo/vllm-runtime:1.0.0

    VllmDecodeWorker:
      componentType: worker
      replicas: 1
      checkpoint:
        enabled: true
        mode: Auto
        identity:
          model: Qwen/Qwen3-0.6B
          backendFramework: vllm
          tensorParallelSize: 1
          dtype: bfloat16
          maxModelLen: 2048
      extraPodSpec:
        mainContainer:
          image: registry.example.com/dynamo/vllm-placeholder:1.0.0
          ...
        ...
```

Auto 模式仅对 `checkpoint.identity` 求哈希。GMS 专属的 checkpoint 行为尚未提供。

实用的检查命令：

```bash
kubectl get dgd vllm-auto-demo -n ${NAMESPACE} \
  -o jsonpath='{.status.checkpoints.VllmDecodeWorker.checkpointName}{"\n"}{.status.checkpoints.VllmDecodeWorker.identityHash}{"\n"}{.status.checkpoints.VllmDecodeWorker.ready}{"\n"}'

kubectl get dckpt -n ${NAMESPACE}
```

如想在 checkpoint 就绪后强制触发一次新的恢复，对 Worker 进行扩容：

```bash
kubectl patch dgd vllm-auto-demo -n ${NAMESPACE} --type=merge \
  -p '{"spec":{"services":{"VllmDecodeWorker":{"replicas":2}}}}'
```

## 故障切换恢复

故障切换（Failover）恢复尚未提供。当前 Snapshot 流程不支持 GMS + Snapshot，因此请勿将故障切换恢复作为受支持的 checkpoint/restore 路径。当前 GMS 与主备故障切换的指引参见 [Shadow Engine Failover](shadow-engine-failover.md)。

## 使用 `snapshotctl` 进行底层测试

可以通过较底层的 `snapshotctl` 工具在不依赖 Dynamo Operator 的情况下对 Pod 做 checkpoint 与 restore。但仍需安装 snapshot helm chart，且命名空间内有正在运行的 `snapshot-agent` DaemonSet 并已挂载 checkpoint PVC。

`snapshotctl` 用于较底层的调试与验证流程，不是面向用户的主要 checkpoint 接口。命令细节与清单要求请参见 [deploy/snapshot/cmd/snapshotctl/README.md](../../deploy/snapshot/cmd/snapshotctl/README.md)。

### 从 Worker Pod 清单做 checkpoint

```bash
snapshotctl checkpoint \
  --manifest ./worker-pod.yaml \
  --container main \
  --namespace ${NAMESPACE}
```

checkpoint 清单必须是一个 Pod 并使用占位镜像。`--container` 指定要 checkpoint 的工作负载容器名。

如果未传入 `--checkpoint-id`，`snapshotctl` 会自动生成并打印：

```text
status=completed
namespace=...
name=...
checkpoint_job=...
checkpoint_id=manual-snapshot-...
checkpoint_location=/checkpoints/...
```

### 从 Worker Pod 清单恢复

```bash
snapshotctl restore \
  --manifest ./worker-pod.yaml \
  --namespace ${NAMESPACE} \
  --checkpoint-id manual-snapshot-... \
  --containers main
```

它会创建一个新的恢复 Pod，并在请求提交后返回。可通过 Kubernetes 就绪状态、events 与日志观察进度。

### 就地恢复一个已有 Pod

```bash
snapshotctl restore \
  --pod existing-restore-target \
  --namespace ${NAMESPACE} \
  --checkpoint-id manual-snapshot-... \
  --containers main
```

它会向已与 snapshot 兼容的现有 Pod 打上恢复元数据，并在 patch 被接受后返回。

## Checkpoint 身份

Checkpoint 由配置中影响运行时状态的字段计算得到的 **16 字符 SHA256 哈希**（64 位）唯一标识：

| 字段 | 必填 | 影响哈希 | 示例 |
|-------|----------|-------------|---------|
| `model` | ✓ | ✓ | `meta-llama/Llama-3-8B` |
| `backendFramework` | ✓ | ✓ | `vllm` |
| `dynamoVersion` | | ✓ | `0.9.0`、`1.0.0` |
| `tensorParallelSize` | | ✓ | `1`、`2`、`4`、`8` |
| `pipelineParallelSize` | | ✓ | `1`、`2` |
| `dtype` | | ✓ | `float16`、`bfloat16`、`fp8` |
| `maxModelLen` | | ✓ | `4096`、`8192` |
| `extraParameters` | | ✓ | 自定义 key-value 对 |

**不会**改变 checkpoint 哈希的字段包括：

- 副本数（replica count）
- 节点放置（`nodeSelector`、`affinity`、`tolerations`）
- 资源请求/限制（resource requests/limits）
- 日志或可观测性配置

## `DynamoCheckpoint` CRD

`DynamoCheckpoint`（短名：`dckpt`）是 Operator 管理 checkpoint 生命周期的资源。

适用场景：

- 在任何 `DynamoGraphDeployment` 创建之前预热 checkpoint
- 独立于 DGD 进行显式生命周期控制
- 提供稳定的、可读名称，让 service 通过 `checkpointRef` 引用

Operator 要求：

- `spec.identity`
- `spec.job.podTemplateSpec`

`spec.job.backoffLimit` 已弃用并被忽略。Checkpoint Job 始终为单次尝试。

查看状态：

```bash
kubectl get dckpt -n ${NAMESPACE}
kubectl describe dckpt qwen3-06b-bf16 -n ${NAMESPACE}
kubectl get dckpt qwen3-06b-bf16 -n ${NAMESPACE} -o yaml
```

`status` 块形如：

```yaml
status:
  phase: Ready
  identityHash: 3bff874d069f0ed5
  jobName: checkpoint-job-3bff874d069f0ed5-1
  createdAt: "2026-01-29T10:05:00Z"
  message: ""
```

## 限制

- **后端支持有限**：checkpoint/restore 当前仅支持 vLLM Worker，且仍处于有限预览阶段。
- **Worker 覆盖范围有限**：尚不支持多模态、embedding、扩散等专业 Worker。
- **多 GPU 仍为预览**：vLLM 的张量并行配置验证有限，跨集群尚未广泛支持。
- **GMS 恢复仍为实验性**：GMS + Snapshot 当前关闭。
- **网络状态敏感**：恢复对 TCP socket 在线状态较敏感。loopback bootstrap/控制 socket 是当前最稳的路径。
- **需要特权 DaemonSet**：`snapshot-agent` 必须以特权方式运行以执行 CRIU 与 `cuda-checkpoint`。工作负载 Pod 不需要特权。

## 故障排查

### Checkpoint Job 完成但 checkpoint 一直没进入 `Ready`

snapshot 仅在 `snapshot-agent` 确认 checkpoint 内容后才进入 `Ready`。Job 完成本身并不足够。

```bash
kubectl get dckpt <checkpoint-name> -n ${NAMESPACE} \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,MESSAGE:.status.message,JOB:.status.jobName

JOB_NAME=$(kubectl get dckpt <checkpoint-name> -n ${NAMESPACE} -o jsonpath='{.status.jobName}')
if [ -n "${JOB_NAME}" ]; then
  kubectl logs job/"${JOB_NAME}" -n ${NAMESPACE}
fi

kubectl logs daemonset/snapshot-agent -n ${NAMESPACE} --all-containers
```

如果 Worker 模板有问题，最常见的原因是使用了原始 runtime 镜像而非占位镜像，或者遗漏了 Worker 启动所需的常规挂载与 Secret。

### Restore 找不到或挂载不上 checkpoint 存储

对于默认的 `agentMount` 安装，恢复会从工作负载命名空间内的 `snapshot-agent` DaemonSet 发现 checkpoint 存储。该 DaemonSet 必须就绪并挂载 checkpoint PVC。

```bash
kubectl rollout status daemonset/snapshot-agent -n ${NAMESPACE}
kubectl get daemonset -n ${NAMESPACE} -l app.kubernetes.io/component=snapshot-agent -o wide
kubectl get pvc -n ${NAMESPACE}
```

对于共享 agent 的 `podMount` 安装，`snapshot-agent` DaemonSet 可改运行在基础设施命名空间中。先检查那里的共享 agent Pod，再确认工作负载命名空间是否有 Operator 创建或验证过的 checkpoint PVC：

```bash
kubectl rollout status daemonset/snapshot-agent -n dynamo-system
kubectl get pods -n dynamo-system -l app.kubernetes.io/component=snapshot-agent -o wide
kubectl get pvc snapshot-pvc -n ${NAMESPACE}
```

`podMount` 模式下，agent 通过工作负载 Pod 的 mount 命名空间访问 checkpoint，而不是自己挂载 PVC。检查工作负载 Pod 的 checkpoint 存储相关 annotation 与 `snapshot-agent` 日志，可看到实际解析出的 checkpoint 路径。`snapshotctl` 使用 chart 的存储解析路径，因此在使用较底层的 `snapshotctl` 调试时，请确保 snapshot chart 配置与你正在测试的访问模式一致。

### `snapshotctl` 清单被拒绝或恢复目标错误

`snapshotctl` 要求是 `Pod` 清单，并提供目标容器列表。多容器清单允许，只要所有通过 `--container` 或 `--containers` 传入的名称都存在于 Pod spec 中。

```bash
snapshotctl checkpoint --manifest ./worker-pod.yaml --container main --namespace ${NAMESPACE}
snapshotctl restore  --manifest ./worker-pod.yaml --containers main --namespace ${NAMESPACE} --checkpoint-id <checkpoint-id>
```

如果清单本身已带 snapshot 目标元数据，则它必须与 CLI 参数一致；`snapshotctl` 不会静默择一，而是直接拒绝不匹配。

## 计划中的特性

- 稳定多 GPU 支持
- 增加更多后端支持
- 替代存储后端

## 相关文档

- [Installation Guide](installation-guide.md)
- [Shadow Engine Failover](shadow-engine-failover.md)
- [API Reference](api-reference.md)
