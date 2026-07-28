---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Developing the Operator with Tilt
subtitle: Fast, live-reload development loop for the Dynamo Kubernetes operator
---

## 概述

[Tilt](https://tilt.dev) 为 Dynamo Kubernetes operator 提供一个具备热加载的开发环境。你不必再为每次改动手动构建镜像、推送到 registry、再重新部署，Tilt 会监听源文件变化并自动重新编译 Go 二进制、把它同步到运行中的容器中并重启进程 —— 整个过程仅需数秒。

在底层，Tiltfile 会：

1. 在本地**编译** Go manager 二进制（`CGO_ENABLED=0`）。
2. **构建**只包含该二进制的极小 Docker 镜像。
3. 通过 `helm template` **渲染**生产 Helm chart（`deploy/helm/charts/platform`），用 `kubectl` 应用 CRD，并部署所有渲染出的资源。
4. 在每次代码变化时**实时更新**容器内的二进制 —— 无需完整重建镜像。

这样你就拥有一个完整可用的集群（cluster），可以在其中应用 `DynamoGraphDeployment` 与 `DynamoGraphDeploymentRequest` 资源，并真正调谐成实际工作负载，同时以亚秒级的反馈速度迭代控制器逻辑。

## 前置条件

| 工具 | 版本 | 用途 |
|------|---------|---------|
| [Tilt](https://docs.tilt.dev/install.html) | v0.33+ | 开发编排 |
| [Helm](https://helm.sh/docs/intro/install/) | v3 | Chart 渲染 |
| [Go](https://go.dev/dl/) | 1.25+ | 编译 operator |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | — | 集群访问 |
| 一个 Kubernetes 集群 | — | kind、minikube 或远端集群 |

你还需要一个**容器 registry**，集群节点可以从中拉取 Tilt 构建的 operator 镜像。如果使用本地 kind 集群配合本地 registry，Tilt 可以直接推送到那里。

## 快速上手

```bash
cd deploy/operator

# Create your personal settings file (gitignored)
cat > tilt-settings.yaml <<EOF
allowed_contexts:
  - my-cluster-context
registry: docker.io/myuser
skip_codegen: true
EOF

# Launch
tilt up
```

Tilt 会打开一个终端 UI，并在 [http://localhost:10350](http://localhost:10350) 启动一个 Web 仪表盘。
仪表盘会展示资源状态、构建日志以及端口转发情况。

在终端按 **Space** 即可打开 Web UI。按 **Ctrl-C** 关闭一切（资源会保留；运行 `tilt down` 才会拆除它们）。

![Tilt Web UI 展示了 operator、CRD 与基础设施资源](../assets/img/tilt-ui.png)

## 配置

所有配置项都是可选的。Tiltfile 为每个设置都提供了合理的默认值，并且 `tilt-settings.yaml` 会被 gitignore，所以你的个人配置（集群上下文、registry 等）不会泄漏到仓库中。

在 `deploy/operator/tilt-settings.yaml` 中可填写以下任意设置：

```yaml
# Kubernetes contexts Tilt is allowed to connect to.
# Safety guard: prevents accidental deployments to production clusters.
allowed_contexts:
  - my-cluster-context

# Container registry for the operator image.
# Can also be set via the REGISTRY env var (env var takes precedence).
registry: docker.io/myuser

# Skip running `make generate && make manifests` before applying CRDs.
# Set to true when you haven't changed API types (faster iteration).
skip_codegen: true

# Target namespace for the operator and related resources.
# namespace: dynamo-system

# Subchart toggles
# enable_nats: true            # Required for DGD/DGDR workloads (default: true)
# enable_etcd: false           # Only if discoveryBackend is "etcd"
# enable_kai_scheduler: false  # GPU-aware scheduling for multi-node
# enable_grove: false          # PodClique-based multi-node orchestration

# Extra Helm value overrides (applied on top of subchart toggles)
# helm_values:
#   dynamo-operator.discoveryBackend: kubernetes
#   dynamo-operator.natsAddr: "nats://external-nats:4222"
```

### 配置项参考

| Key | 类型 | 默认值 | 描述 |
|-----|------|---------|-------------|
| `allowed_contexts` | list | *(none)* | Tilt 允许连接的 Kubernetes 上下文。可避免误部署到生产集群。 |
| `registry` | string | `""` | 容器 registry 前缀（例如 `docker.io/myuser`）。也可通过环境变量 `REGISTRY` 设置，环境变量优先。 |
| `namespace` | string | `dynamo-system` | Operator Deployment 与相关资源所在的命名空间。 |
| `skip_codegen` | bool | `false` | 在应用 CRD 前跳过 `make generate && make manifests`。当你没有改动 API 类型时设为 `true` 可加速迭代。 |
| `enable_nats` | bool | `true` | 部署 NATS 子 chart。DGD/DGDR 工作负载需要它（worker 之间通过 NATS 通信）。 |
| `enable_etcd` | bool | `false` | 部署 etcd 子 chart。仅当 `discoveryBackend` 为 `etcd` 时需要。 |
| `enable_kai_scheduler` | bool | `false` | 在多节点部署中部署 kai-scheduler，用于 GPU 感知的调度（scheduler）。 |
| `enable_grove` | bool | `false` | 部署 Grove，用于基于 PodClique 的多节点编排。 |
| `image_pull_secret` | string | `""` | 用于从私有 registry 拉取镜像的 `docker-registry` Secret 名称。 |
| `helm_values` | map | `{}` | 任意 `--set` 形式的覆盖值，会传给 `helm template`。 |
| `operator_version` | string | *(from Chart.yaml)* | Operator 的 `--operator-version` 标志。默认取 operator 子 chart 中的 `appVersion`。 |

### Registry 配置

集群节点必须能拉取到 operator 镜像。registry 的解析顺序为：

1. **`REGISTRY` 环境变量** —— `REGISTRY=docker.io/myuser tilt up`
2. **`tilt-settings.yaml` 中的 `registry`**

镜像会被推送为 `{registry}/controller:tilt-dev`。

<Warning>
如果未配置 registry，镜像将仅存在于本地。这种方式在 kind + 本地 registry 下可用，但在远端集群上会失败。
</Warning>

## 工作原理

运行 `tilt up` 后，会按以下顺序创建资源：

```
manager-build     Compile Go binary locally
        │
        ├───── crds       Apply CRDs via server-side apply
        │
    operator              Deploy operator pod (live-updated)
```

webhook 证书生成、CA bundle 注入与 MPI SSH key 生成都由 operator 在运行时自行处理 —— 无需任何额外的外部组件。

### 各资源的作用

**manager-build** —— 运行 `CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build` 编译 operator 二进制。当 `api/`、`cmd/`、`internal/`、`go.mod` 或 `go.sum` 变化时会重新执行。

**crds** —— 通过 `kubectl apply --server-side` 把 Helm chart 中的 CRD 应用到集群。当 `skip_codegen` 为 `false` 时，会先执行 `make generate && make manifests`。

**operator** —— operator 自身的 Deployment。Tilt 监听编译后的二进制，并通过 `live_update` 把它同步进运行中的容器并重启进程 —— 无需重建镜像。启动时，operator 内置的 cert controller 会生成自签名 TLS 证书，将 CA bundle 注入到 webhook 配置中，并创建 MPI SSH secret —— 与生产环境行为完全一致。

### 实时更新流程

内层开发循环大致如下：

1. 你编辑 `deploy/operator/` 下的 Go 源文件。
2. Tilt 检测到改动并重新编译二进制（约 2-5 秒）。
3. 新二进制通过 `live_update` 同步到运行中的容器。
4. 进程自动重启。
5. 你的控制器改动立即生效 —— 通过应用 DGD/DGDR 来测试。

无需 `docker build`、无需 `docker push`、也无需 `kubectl rollout restart`。

## Webhook 证书

Operator 在运行时通过内置的 cert controller（基于 OPA cert-controller）自动管理 webhook TLS 证书。启动时它会：

1. 创建一个自签名 CA 与 webhook 服务证书。
2. 将其存入 `webhook-server-cert` Secret。
3. 将 CA bundle 注入 `ValidatingWebhookConfiguration` 和 `MutatingWebhookConfiguration` 资源。

这与生产环境行为一致，且不依赖任何外部工具。如需使用 cert-manager 或外部证书等替代方案，请参阅 [webhook 文档](../kubernetes/webhooks.md)，并通过 `tilt-settings.yaml` 中的 `helm_values` 进行配置。

## 常见工作流

### 迭代控制器逻辑

最常见的工作流 —— 你在修改调谐逻辑并希望快速反馈：

```yaml
# tilt-settings.yaml
allowed_contexts: [my-cluster]
registry: docker.io/myuser
skip_codegen: true
```

```bash
tilt up
# Edit files under internal/controller/
# Tilt auto-recompiles and live-updates
# Apply test resources:
kubectl apply -f examples/backends/vllm/deploy/agg.yaml
```

### 修改 API 类型（CRD）

当你修改 `api/` 下的文件时，需要执行 codegen：

```yaml
# tilt-settings.yaml
skip_codegen: false   # or omit — false is the default
```

`api/` 文件变更时，Tilt 会自动运行 `make generate && make manifests` 并重新应用 CRD。

### 测试多节点功能

启用所需的子 chart：

```yaml
# tilt-settings.yaml
enable_grove: true
enable_kai_scheduler: true
```

### 使用环境变量

可以不修改设置文件直接覆盖 registry：

```bash
REGISTRY=ghcr.io/myorg tilt up
```

## Tilt UI

[http://localhost:10350](http://localhost:10350) 上的 Web UI 会显示：

- **资源状态** —— 各资源的绿/红/挂起状态
- **构建日志** —— 编译输出与错误
- **运行日志** —— 实时流式 operator 日志
- **端口转发** —— health endpoint 被转发到 `localhost:8081`

资源会按标签（`operator` 与 `infrastructure`）分组，便于在 UI 中查看。

## 清理

```bash
# Stop Tilt and leave resources deployed
# (Ctrl-C in the terminal)

# Stop Tilt and tear down all resources
tilt down
```

## 故障排查

### 镜像拉取失败

如果 pod 出现 `ImagePullBackOff`：
- 确认 `tilt-settings.yaml` 中或环境变量 `REGISTRY` 设置了 `registry`。
- 确认集群节点能从该 registry 拉取镜像。
- 对于 kind + 本地 registry，请参考 [kind 本地 registry 指南](https://kind.sigs.k8s.io/docs/user/local-registry/)。

### Webhook TLS 错误

如果在应用 DGD/DGDR 时报 `x509: certificate signed by unknown authority`：
- 在 Tilt UI 中查看 operator 日志 —— cert controller 会在启动时记录其执行进度。
- 确认 `webhook-server-cert` Secret 存在且已被填充：
  ```bash
  kubectl -n dynamo-system get secret webhook-server-cert
  ```
- Operator 在启动后可能需要几秒才能生成证书并注入 CA bundle。请等到看到 `cert-controller` 日志后再去应用资源。

### CRD codegen 失败

如果 `crds` 因 codegen 错误而失败：
- 确保已安装 `controller-gen`：`make controller-gen`
- 尝试手动运行 codegen：`make generate && make manifests`
- 如果你没有改动 API 类型，可临时设置 `skip_codegen: true` 绕过该步骤。

### 上下文安全保护

如果 Tilt 因上下文错误而拒绝启动，请在 `tilt-settings.yaml` 中将你的集群上下文加入 `allowed_contexts`：

```yaml
allowed_contexts:
  - my-cluster-context
```
