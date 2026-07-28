# `snapshotctl`

`snapshotctl` 是面向开发者与运维人员的较低层级 snapshot 工具。
它并不是 Dynamo 主要的用户工作流。常规面向用户的路径是：

```text
DynamoCheckpoint CR -> operator -> snapshot-agent
```

当你希望直接基于 worker pod manifest 触发 checkpoint 或 restore 行为，
而不经过 operator 时，可以使用 `snapshotctl`。

## 前置条件

- 目标命名空间内必须已安装 snapshot Helm chart
- 该命名空间内必须运行有 `snapshot-agent` DaemonSet
- 该命名空间内必须已经挂载好被 agent 使用的 checkpoint PVC

## Manifest 要求

`snapshotctl checkpoint --manifest ...` 与 `snapshotctl restore --manifest ...`
接受的是 Kubernetes `Pod` manifest，而不是 Deployment 或 Job manifest。

该 pod manifest 必须：

- 描述你要 checkpoint 或 restore 的 worker pod
- 在 checkpoint 感知的流程中，使用占位（placeholder）镜像
- 与你想保留的运行时相关 worker 设置保持一致

实际操作中，建议从你日常运行的 worker pod 规范出发，仅保留为了准确重建该
worker 所需的 pod 级字段即可。

## 目标容器

操作作用于哪个（哪些）容器，由 snapshot 层视为强制必填的一个 annotation 控制：

```yaml
metadata:
  annotations:
    nvidia.com/snapshot-target-containers: "main"
    # or "engine-0,engine-1" for a failover-style restore
```

- Checkpoint 必须 **恰好** 针对一个容器。
- Restore 必须 **至少** 针对一个容器；同一份 checkpoint 会被回放到所有命名容器中。

`snapshotctl` 会根据 CLI 参数自动为你写入该 annotation：

- `checkpoint` 子命令使用 `--container <name>`（单一名称）
- `restore` 子命令使用 `--containers <name>[,<name>...]`（一个或多个）

你也可以预先在 manifest 上写好该 annotation，并省略命令行标志。如果两者都
提供，则必须保持一致 —— `snapshotctl` 不会默默选择其一，遇到不一致会直接拒绝。

## 命令

基于 manifest 进行 checkpoint：

```bash
snapshotctl checkpoint \
  --manifest ./worker-pod.yaml \
  --container main \
  --namespace ${NAMESPACE}
```

如果省略 `--checkpoint-id`，`snapshotctl` 会自动生成一个。

通过 manifest 创建一个新 pod 来执行 restore：

```bash
snapshotctl restore \
  --manifest ./worker-pod.yaml \
  --containers main \
  --namespace ${NAMESPACE} \
  --checkpoint-id manual-snapshot-123
```

如果是 pod 内的 failover 类型 restore，需要列出每个 engine 容器：

```bash
snapshotctl restore \
  --manifest ./failover-pod.yaml \
  --containers engine-0,engine-1 \
  --namespace ${NAMESPACE} \
  --checkpoint-id manual-snapshot-123
```

对一个已经存在并且兼容 snapshot restore 的 pod 进行原地 restore：

```bash
snapshotctl restore \
  --pod existing-restore-target \
  --containers main \
  --namespace ${NAMESPACE} \
  --checkpoint-id manual-snapshot-123
```

## 注意事项

- `restore --pod` 期望传入的 pod 已经与 snapshot restore 兼容
- `restore --manifest` 会基于你提供的 manifest 创建一个新的 restore 目标 pod
- `restore` 在 restore 请求提交后即返回，不会等待完成
- 可通过 pod 就绪状态、events/logs，以及每个容器的
  `nvidia.com/snapshot-restore-status.<container>` annotation 观察 restore 进度
- `snapshotctl` 适合用于调试与较低层级的验证，但不能替代由 operator 管理的
  checkpoint 流程
