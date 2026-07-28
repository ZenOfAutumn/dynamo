# Dynamo Snapshot Helm Chart

> 实验性功能。`snapshot-agent` 以特权 DaemonSet 形式运行，
> 用于执行 CRIU 的 checkpoint 与 restore 操作。

本 chart 安装 Dynamo 使用的、命名空间作用域的 snapshot 基础设施：

- 在符合条件的 GPU 节点上部署 `snapshot-agent` DaemonSet
- 创建 `snapshot-pvc`，或者接入已有 PVC
- 命名空间作用域的 RBAC
- CRIU 所需的 seccomp profile

默认情况下，请在每个需要 checkpoint 与 restore 的命名空间中安装本 chart；
此模式下 chart 可以创建或复用该命名空间的 PVC。

也可以选择在某个基础设施命名空间中只安装一个集群作用域的 `snapshot-agent`，
方法是设置 `storage.accessMode=podMount` 与 `rbac.namespaceRestricted=false`。
此模式下 DaemonSet 不直接挂载 checkpoint PVC。Dynamo operator 会把命名空间本地的
checkpoint PVC 挂载进 checkpoint/restore 工作负载 pod，agent 通过目标 pod 的
mount namespace 访问该 PVC。这些命名空间本地工作负载 PVC 既可由 operator 创建，
也可要求其预先存在。

## 前置条件

- 包含 x86_64 GPU 节点的 Kubernetes 集群
- NVIDIA 驱动 580.xx 或更新
- **containerd** 或 **CRI-O**（chart 默认使用 containerd；CRI-O / OpenShift 见下方）
- 已经安装 Dynamo Platform 并设置 `dynamo-operator.checkpoint.enabled=true`
- 集群允许带有 `hostPID`、`hostIPC`、`hostNetwork` 的特权 DaemonSet

在默认的 `agentMount` 模式下，snapshot-agent DaemonSet 会直接挂载 checkpoint PVC。
在多节点 GPU 集群中，这意味着多个节点上的 agent pod 可能挂载同一个 PVC，
因此该 PVC 通常需要 `ReadWriteMany`。chart 默认使用该模式。

`podMount` 取消了 agent 侧的 RWX 要求。snapshot-agent 不再直接挂载 PVC；
只有正在 checkpoint/restore 的工作负载 pod 挂载它，agent 通过该 pod 的
mount namespace 访问 PVC。这样，对于按顺序进行 checkpoint/restore 的工作流，
可以使用合适的 `ReadWriteOnce` 存储类，只要存储后端能把卷重新挂载到承载该
工作负载 pod 的节点上即可。

由于 `podMount` 通过 `/host/proc/<pid>/root` 访问存储，agent 启动 checkpoint
或 restore 时，目标容器必须仍然存活，并能通过 host proc 看到。容器重启、
退出的占位容器、运行时 PID 查找失败，或者节点安全策略阻止访问 host proc，
都会导致该次尝试失败或被推迟，直到 reconcile 看到一个新的容器。

## CRI-O 与 OpenShift

对于 CRI-O 节点，请设置 `runtime.type=crio`。仅当 CRI socket 不是该 type
的默认值时，才需要设置 `runtime.socketPath`（参见 `values.yaml`）。
在 OpenShift 上，请设置 `openshift.enabled=true`，让 chart 输出 agent 所需的
额外 RBAC 与 pod annotation。示例：

```bash
helm upgrade --install snapshot ./deploy/helm/charts/snapshot \
  --namespace "${NAMESPACE}" --create-namespace \
  --set storage.pvc.create=true \
  --set runtime.type=crio \
  --set openshift.enabled=true
```

## 最小化安装

创建 checkpoint PVC 与 agent：

```bash
helm upgrade --install snapshot ./deploy/helm/charts/snapshot \
  --namespace ${NAMESPACE} \
  --create-namespace \
  --set storage.pvc.create=true
```

如果集群没有默认的 storage class，还需设置 `storage.pvc.storageClass`。

复用已有 PVC：

```bash
helm upgrade --install snapshot ./deploy/helm/charts/snapshot \
  --namespace ${NAMESPACE} \
  --create-namespace \
  --set storage.pvc.create=false \
  --set storage.pvc.name=my-snapshot-pvc
```

## 验证

```bash
kubectl get pvc snapshot-pvc -n ${NAMESPACE}
kubectl rollout status daemonset/snapshot-agent -n ${NAMESPACE}
kubectl get pods -n ${NAMESPACE} -l app.kubernetes.io/name=snapshot -o wide
```

## 重要 values

| 参数 | 含义 | 默认值 |
|-----------|---------|---------|
| `storage.type` | snapshot 自有的存储后端 | `pvc` |
| `storage.accessMode` | `agentMount` 表示由命名空间本地的 agent 挂载 PVC，`podMount` 表示用单个集群级 agent 访问由工作负载挂载的 PVC | `agentMount` |
| `storage.pvc.create` | 在 `agentMount` 模式下创建 `snapshot-pvc` 而非使用已有 PVC | `true` |
| `storage.pvc.name` | checkpoint PVC 名称。在 `agentMount` 模式下由 agent 挂载，在 `podMount` 模式下由工作负载 pod 挂载 | `snapshot-pvc` |
| `storage.pvc.size` | 申请的 PVC 大小 | `1Ti` |
| `storage.pvc.storageClass` | storage class 名称 | `""` |
| `storage.pvc.accessMode` | checkpoint PVC 的访问模式。`ReadWriteMany` 最稳妥；在合适的存储后端上，`ReadWriteOnce` 可与 `podMount` 搭配用于顺序 checkpoint/restore | `ReadWriteMany` |
| `storage.pvc.basePath` | checkpoint 存储的挂载路径：`agentMount` 模式在 `snapshot-agent` pod 内部，`podMount` 模式在 checkpoint/restore 工作负载 pod 内部 | `/checkpoints` |
| `seccomp.deploy` | 部署 CRIU seccomp profile 的 ConfigMap 与 init container。请使用此字段名；`seccomp.enabled` 不是 chart value | `true` |
| `daemonset.image.repository` | snapshot-agent 镜像仓库 | `nvcr.io/nvidia/ai-dynamo/snapshot-agent` |
| `daemonset.image.tag` | snapshot-agent 镜像 tag | `1.0.0` |
| `daemonset.imagePullSecrets` | agent 的 image pull secrets | `[{name: ngc-secret}]` |
| `runtime.type` | CRI 后端：`containerd` 或 `crio` | `containerd` |
| `runtime.socketPath` | CRI socket（空表示使用 `runtime.type` 的默认值） | `""` |
| `openshift.enabled` | OpenShift 相关的 RBAC / SCC chart 片段 | `false` |

`s3` 与 `oci` 是预留的 chart 名称，未来用于其他 snapshot 后端，
当前仅实现了 `pvc`。

当使用 `storage.accessMode=podMount` 时，请在 Dynamo operator 中配置相同的
工作负载 PVC 名称与挂载路径。snapshot chart 中的字段是 `storage.pvc.name`；
operator 配置中的字段名是 `pvcName`，因为它们是不同的 API/配置对象。
如果希望 operator 在 reconcile checkpoint 或 restore 工作负载时创建命名空间
本地 PVC，请使用 `create: true`；如果要求 PVC 必须预先存在，则省略或设为 `false`：

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

对于没有可用或负担不起 RWX storage class 的集群，可以将 `podMount` 与
`accessMode: ReadWriteOnce` 搭配使用，因为 DaemonSet 不再需要在每个 GPU 节点
上挂载 PVC。这适用于 checkpoint/restore pod 一次只用一个的工作流。
对于并发 restore、多节点同时访问，或者无法把 RWO 卷重新挂载到所选 restore 节点
的存储后端，请继续使用 `ReadWriteMany`。

完整配置参见 [values.yaml](./values.yaml)。

## 后续步骤

chart 安装完成后，请参照 snapshot 指南创建 checkpoint，或体验底层的
`snapshotctl` 流程：

- [Snapshot 指南](../../../../docs/kubernetes/snapshot.md)

## 卸载

```bash
helm uninstall snapshot -n ${NAMESPACE}
```

chart 不会自动删除 checkpoint 数据。如需清理已存的 checkpoint，请自行删除 PVC：

```bash
kubectl delete pvc snapshot-pvc -n ${NAMESPACE}
```
