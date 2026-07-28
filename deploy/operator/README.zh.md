# Dynamo Kubernetes Operator

用于通过自定义资源管理所有 Dynamo 流水线（pipelines）的 Kubernetes Operator。


## 概览

该 operator 自动化处理 Kubernetes 集群中 Dynamo 资源的部署与生命周期管理：

- **DynamoGraphDeploymentRequest（DGDR）** - 简化的、SLA 驱动的部署接口
- **DynamoGraphDeployment（DGD）** - 直接的部署配置

它基于 [Kubebuilder](https://book.kubebuilder.io/) 构建，遵循 Kubernetes 最佳实践，并通过 CustomResourceDefinitions（CRD）支持声明式配置。

### 自定义资源

- **DynamoGraphDeploymentRequest**：用于 SLA 驱动配置生成的高层接口。它会自动处理性能剖析（profiling），并基于你的性能需求生成优化后的 DGD 规约。
- **DynamoGraphDeployment**：用于直接部署配置的低层接口，可对所有参数进行完全控制。


## 开发者指南

### 前置条件

- [Go](https://go.dev/doc/install) >= 1.25
- [Kubebuilder](https://book.kubebuilder.io/quick-start.html)

### 构建

```
make
```

### 使用 Tilt 进行本地开发

[Tilt](https://docs.tilt.dev/install.html) 为 operator 提供了实时刷新（live-reload）开发循环。它会在本地编译 Go 二进制、构建一个最小化的 Docker 镜像、渲染生产环境的 Helm chart，并将所有内容部署到你的集群。代码变更时，Tilt 会重新编译并实时更新二进制文件，无需完整重建镜像 — 因而可以在真实集群上快速迭代控制器逻辑。

#### 前置条件

运行 `tilt up` 之前，需要在 `PATH` 中安装并可用以下工具：

| 工具 | 版本 | 用途 | 安装 |
|------|---------|---------|---------|
| [Go](https://go.dev/doc/install) | ≥ 1.25 | 在本地编译 manager 二进制 | [go.dev/doc/install](https://go.dev/doc/install) |
| [Tilt](https://docs.tilt.dev/install.html) | latest | 实时刷新开发循环编排器 | [docs.tilt.dev/install](https://docs.tilt.dev/install.html) |
| [Helm](https://helm.sh/docs/intro/install/) | v3 | 渲染 platform Helm chart | [helm.sh/docs/intro/install](https://helm.sh/docs/intro/install/) |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | ≥ 1.29 | 应用 CRD 并创建命名空间 | [kubernetes.io/docs/tasks/tools](https://kubernetes.io/docs/tasks/tools/) |
| [Docker](https://docs.docker.com/get-docker/) | latest | 构建实时更新的容器镜像 | [docs.docker.com/get-docker](https://docs.docker.com/get-docker/) |

**条件性前置条件**（仅在 `skip_codegen: false`（默认值）时需要）：

| 工具 | 版本 | 用途 | 安装 |
|------|---------|---------|---------|
| [yq](https://github.com/mikefarah/yq) | v4+ | 对生成的 CRD YAML 进行后处理 | `make ensure-yq` 或 [github.com/mikefarah/yq](https://github.com/mikefarah/yq) |
| [Python 3](https://www.python.org/) + [pydantic](https://docs.pydantic.dev/) | 3.x | 从 Go 类型生成 Pydantic 模型（`make generate`） | `pip install pydantic` |

> **提示：** 在 `tilt-settings.yaml` 中设置 `skip_codegen: true`，可以在每次重新加载时跳过 CRD/代码生成。这会移除 yq/Python 依赖，并在你未修改 API 类型时加快迭代速度。

**集群：** 你需要一个 Tilt 可访问其 kubeconfig context 的 Kubernetes 集群（kind、minikube、GKE、EKS、裸金属等）。如果集群带 GPU 且你希望端到端测试 DGD/DGDR 工作负载，应在集群上安装 [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html)。

#### 搭建

1. **在 `deploy/operator/` 中创建 `tilt-settings.yaml`**，最小化配置如下：
   ```yaml
   allowed_contexts:
     - h100                 # Change to your Kubernetes context

   registry: docker.io/myuser  # Change to your Docker registry
   ```

2. **运行 Tilt**：
   ```bash
   cd deploy/operator
   tilt up
   ```
   Tilt UI 会在 http://localhost:10350 打开，显示资源状态与日志。

#### 特性

- **快速迭代**：代码变更时，Tilt 会重新编译 manager 二进制并实时更新到运行中的容器 — 无需完整重建镜像
- **真实集群测试**：在你真实的 Kubernetes 集群（kind、minikube、GKE、AKS 等）上进行调谐（reconcile）
- **CRD + Helm 渲染**：自动应用 CRD，并按你的配置渲染 platform Helm chart
- **基础设施开关**：通过 `tilt-settings.yaml` 控制 NATS、etcd、KAI scheduler 与 Grove

#### 可选配置

`tilt-settings.yaml` 中的额外可选配置：

```yaml
# Infrastructure toggles (control which components are deployed)
enable_nats: true              # Enable NATS messaging (default: true, required for DGD/DGDR)
enable_etcd: false             # Enable etcd service discovery (default: false)
enable_kai_scheduler: false    # Enable KAI GPU-aware scheduler (default: false)
enable_grove: false            # Enable Grove orchestrator (default: false)

# Other settings
namespace: dynamo-system       # Kubernetes namespace for operator deployment
skip_codegen: false            # Skip code generation for faster reloads if API unchanged
image_pull_secret: ""          # Name of Secret for private Docker registries
helm_values: {}                # Extra Helm value overrides for platform chart
operator_version: "0.0.0-dev"  # Override operator version (default: from Chart.yaml)
```

### 安装

请参见 [Dynamo Kubernetes Platform 安装指南](/docs/kubernetes/installation-guide.md) 获取安装说明。
