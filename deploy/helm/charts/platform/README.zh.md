<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# dynamo-platform

NVIDIA Dynamo Platform 的 Helm chart。

![Version: 1.2.0](https://img.shields.io/badge/Version-1.2.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square)

## 🚀 概述

Dynamo Platform Helm chart 在 Kubernetes 上部署完整的 Dynamo Kubernetes Platform 基础设施，包括：

- **Dynamo Operator**：用于管理 Dynamo 部署的 Kubernetes operator
- **NATS**：用于组件间通信的高性能消息系统
- **etcd**：用于服务发现的分布式键值存储（可选，默认关闭）
- **Grove**：多节点推理（inference）编排（可选）
- **Kai Scheduler**：进阶工作负载调度（scheduler）（可选）

## 📋 前置条件

- Kubernetes 集群（cluster）（v1.20+）
- Helm 3.8+
- 满足你部署规模所需的集群资源
- 容器 registry 访问权限（如使用私有镜像）
- 用于 admission webhook 的 TLS 证书基础设施（默认由 operator 内置的 cert-controller 自动生成；也可使用 [cert-manager](https://cert-manager.io/) 或外部托管）

## 🔄 升级说明

### Webhook 现已强制启用（v1.0.0+）

`webhook.enabled` 这个 Helm 值已被移除。Admission webhook 现在是 operator 必需的组件，无法禁用。这一变更与即将引入的 CRD conversion webhook 相一致 —— 后者是多版本 API 支持所必需的。

大多数升级**无需任何操作** —— operator 内置的 cert-controller 会在启动时自动生成并轮换 TLS 证书。如果你使用 cert-manager 或外部托管证书，请在升级前确认现有配置正确。

---

## ⚠️ 重要：集群级 vs 命名空间级部署

### 单一集群级 Operator（推荐）

**默认情况下，Dynamo operator 以集群级权限运行，每个集群只应部署一次。**

- ✅ **推荐**：每个集群部署一个集群级 operator
- ❌ **不推荐**：在同一集群中部署多个集群级 operator

### 多个命名空间级 Operator（进阶）

如果你需要多个 operator 实例（例如多租户场景），请使用命名空间级部署：

```yaml
# values.yaml
dynamo-operator:
  namespaceRestriction:
    enabled: true
    targetNamespace: "my-tenant-namespace"  # Optional, defaults to release namespace
```

### 校验与安全保护

该 chart 内置校验，可避免一切 operator 冲突：

- **自动检测**：安装时扫描已有 operator（包括集群级与命名空间级）
- **阻止多个集群级**：若已有集群级 operator，则安装会失败
- **阻止混合部署（类型 1）**：当已有集群级 operator 时，尝试安装命名空间级 operator 会失败
- **阻止混合部署（类型 2）**：当已有命名空间级 operator 时，尝试安装集群级 operator 会失败
- **安全默认值**：Leader election 使用共享 ID 以保证正确协同

#### 🚫 **被阻止的冲突场景**

| 已有 operator | 新 operator | 状态 | 原因 |
|-------------------|--------------|---------|--------|
| 无 | 集群级 | ✅ **允许** | 无冲突 |
| 无 | 命名空间级 | ✅ **允许** | 无冲突 |
| 集群级 | 集群级 | ❌ **阻止** | 多个集群管理者 |
| 集群级 | 命名空间级 | ❌ **阻止** | 集群级已经管理目标命名空间 |
| 命名空间级 | 集群级 | ❌ **阻止** | 会与已有命名空间级 operator 冲突 |
| 命名空间级 A | 命名空间级 B（不同命名空间） | ✅ **允许** | 作用域不同 |

## 🔧 配置

## Requirements

| Repository | Name | Version |
|------------|------|---------|
| file://components/operator | dynamo-operator | 1.2.0 |
| https://charts.bitnami.com/bitnami | etcd | 12.0.18 |
| https://nats-io.github.io/k8s/helm/charts/ | nats | 1.3.2 |
| oci://ghcr.io/ai-dynamo/grove | grove(grove-charts) | v0.1.0-alpha.8 |
| oci://ghcr.io/kai-scheduler/kai-scheduler | kai-scheduler | v0.13.4 |

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| global.etcd.install | bool | `false` | 是否由该 chart 安装内置的 etcd 子 chart。设为 true 时部署 etcd 并自动把其地址配置给 operator；设为 false 时不部署 etcd。如果你希望使用外部 etcd，可通过 dynamo-operator.etcdAddr 指向外部实例。 |
| global.nats.install | bool | `true` | 是否由该 chart 安装内置的 NATS 子 chart。设为 true 时部署 NATS 并自动把其地址配置给 operator；设为 false 时不部署 NATS。如果你希望使用外部 NATS，可通过 dynamo-operator.natsAddr 指向外部实例。从 1.1.0 起默认 true，因为 Dynamo runtime 的 event plane（DYN_EVENT_PLANE）在分布式后端（etcd/kubernetes）下默认使用 NATS。 |
| global.kai-scheduler.install | bool | `false` | 是否由该 chart 安装内置的 kai-scheduler 子 chart。设为 true 时部署 kai-scheduler 及其 CRD，集成会被自动启用。注意：生产环境建议单独安装 kai-scheduler。 |
| global.kai-scheduler.enabled | bool | `false` | 是否启用 Kai Scheduler 集成（队列创建、schedulerName 注入）。当 kai-scheduler 已在集群中存在（外部安装）时设为 true。install=true 时会自动启用。Operator 据此决定是否向 pod 模板注入 schedulerName 与队列标签。 |
| global.grove.install | bool | `false` | 是否由该 chart 安装内置的 Grove 子 chart。设为 true 时以集群级方式部署 Grove operator，集成会被自动启用。注意：生产环境建议单独安装 Grove。 |
| global.grove.enabled | bool | `false` | 是否启用 Grove 集成（通过 PodCliqueSets 进行多节点编排）。当 Grove 已在集群中存在（外部安装）时设为 true。install=true 时会自动为 true。Operator 据此决定是否为多节点部署创建 PodCliqueSets。 |
| dynamo-operator.enabled | bool | `true` | 是否部署 Dynamo Kubernetes operator |
| dynamo-operator.upgradeCRD | bool | `true` | 是否通过 pre-install/pre-upgrade Hook Job 管理 CRD。该 Job 运行 operator 镜像中的 crd-apply 工具，通过 server-side apply 应用 CRD。 |
| dynamo-operator.natsAddr | string | `""` | Operator 通信使用的 NATS 服务器地址（留空则使用内置 NATS chart）。格式：`"nats://hostname:port"` |
| dynamo-operator.etcdAddr | string | `""` | 外部 etcd 实例的地址。仅当不使用内置 etcd 子 chart 时才需要。格式：`"http://hostname:port"` 或 `"https://hostname:port"` |
| dynamo-operator.modelExpressURL | string | `""` | 当 Model Express server 不由该 helm chart 部署时使用的 URL。如果由该 chart 安装（global.model-express.enabled 为 true），此字段会被忽略。 |
| dynamo-operator.namespaceRestriction | object | `{"enabled":false,"lease":{"duration":"30s","renewInterval":"10s"},"targetNamespace":null}` | DEPRECATED：命名空间级模式已弃用，将于将来版本移除。请改用集群级模式（默认）。新部署请勿启用。 |
| dynamo-operator.namespaceRestriction.enabled | bool | `false` | DEPRECATED：新部署请勿启用。命名空间级模式已弃用。 |
| dynamo-operator.namespaceRestriction.targetNamespace | string | `nil` | DEPRECATED：仅在已弃用的命名空间级模式下使用。 |
| dynamo-operator.namespaceRestriction.lease | object | `{"duration":"30s","renewInterval":"10s"}` | DEPRECATED：仅在已弃用的命名空间级模式下使用。 |
| dynamo-operator.namespaceRestriction.lease.duration | string | `"30s"` | DEPRECATED：已弃用的命名空间级模式下的 lease duration。 |
| dynamo-operator.namespaceRestriction.lease.renewInterval | string | `"10s"` | DEPRECATED：已弃用的命名空间级模式下的 lease renew 间隔。 |
| dynamo-operator.gpuDiscovery | object | `{"enabled":true}` | DEPRECATED：随命名空间级模式一同弃用的命名空间级 operator GPU 发现功能。 |
| dynamo-operator.gpuDiscovery.enabled | bool | `true` | DEPRECATED：仅当已弃用的 namespaceRestriction 启用时相关。 |
| dynamo-operator.controllerManager.tolerations | list | `[]` | controller manager pod 的节点 tolerations |
| dynamo-operator.controllerManager.affinity | object | `{}` | controller manager pod 的 affinity |
| dynamo-operator.controllerManager.leaderElection.id | string | `""` | 集群级协同的 leader election ID。警告：所有集群级 operator 必须使用**相同**的 ID，否则会出现脑裂。不同 ID 会允许多个 leader 同时存在。 |
| dynamo-operator.controllerManager.leaderElection.namespace | string | `""` | leader election lease 所在命名空间（仅集群级模式下使用）。留空则默认使用 kube-system 进行集群级协同。所有集群级 operator 应使用相同的命名空间，以保证 leader election 正常工作。 |
| dynamo-operator.controllerManager.manager.image.repository | string | `"nvcr.io/nvidia/ai-dynamo/kubernetes-operator"` | NVIDIA Dynamo operator 官方镜像 repository |
| dynamo-operator.controllerManager.manager.image.tag | string | `""` | 镜像 tag（留空则使用 chart 默认值） |
| dynamo-operator.controllerManager.manager.image.pullPolicy | string | `"IfNotPresent"` | 镜像拉取策略 —— 何时拉取镜像 |
| dynamo-operator.controllerManager.manager.args[0] | string | `"--health-probe-bind-address=:8081"` | Kubernetes 健康检查使用的健康探针端点 |
| dynamo-operator.controllerManager.manager.args[1] | string | `"--metrics-bind-address=127.0.0.1:8080"` | 供 Prometheus 抓取的指标（metrics）端点（出于安全仅监听 localhost） |
| dynamo-operator.imagePullSecrets | list | `[]` | 拉取私有容器镜像所用的 Secret |
| dynamo-operator.dynamo.groveTerminationDelay | string | `"4h"` | 强制终止 Grove 实例之前的等待时长 |
| dynamo-operator.dynamo.dockerRegistry.useKubernetesSecret | bool | `false` | 是否使用 Kubernetes Secret 进行 registry 鉴权 |
| dynamo-operator.dynamo.dockerRegistry.server | string | `nil` | Docker registry 服务器 URL |
| dynamo-operator.dynamo.dockerRegistry.username | string | `nil` | registry 用户名 |
| dynamo-operator.dynamo.dockerRegistry.password | string | `nil` | registry 密码（建议改用 existingSecretName） |
| dynamo-operator.dynamo.dockerRegistry.existingSecretName | string | `nil` | 存放 registry 凭证的现有 Kubernetes Secret 名称 |
| dynamo-operator.dynamo.dockerRegistry.secure | bool | `true` | registry 是否使用 HTTPS |
| dynamo-operator.dynamo.ingress.enabled | bool | `false` | 是否创建 ingress 资源 |
| dynamo-operator.dynamo.ingress.className | string | `nil` | Ingress class 名称（如 "nginx"、"traefik"） |
| dynamo-operator.dynamo.ingress.tlsSecretName | string | `"my-tls-secret"` | 存放 TLS 证书的 Secret 名称 |
| dynamo-operator.dynamo.istio.enabled | bool | `false` | 是否启用 Istio 集成 |
| dynamo-operator.dynamo.istio.gateway | string | `nil` | 用于路由的 Istio gateway 名称 |
| dynamo-operator.dynamo.ingressHostSuffix | string | `""` | 生成 ingress 主机名时使用的后缀 |
| dynamo-operator.dynamo.virtualServiceSupportsHTTPS | bool | `false` | VirtualService 是否支持 HTTPS 路由 |
| dynamo-operator.dynamo.serviceMesh.enabled | bool | `false` | 是否为 EPP 生成 service mesh 资源 |
| dynamo-operator.dynamo.serviceMesh.provider | string | `"istio"` | service mesh 提供方。已支持："istio" |
| dynamo-operator.dynamo.serviceMesh.istio | object | `{"insecureSkipVerify":true,"tlsMode":"SIMPLE"}` | Istio 专属设置（仅当 provider 为 "istio" 时生效） |
| dynamo-operator.dynamo.serviceMesh.istio.tlsMode | string | `"SIMPLE"` | DestinationRule 的 TLS 模式："SIMPLE"、"DISABLE"、"ISTIO_MUTUAL"、"MUTUAL" |
| dynamo-operator.dynamo.serviceMesh.istio.insecureSkipVerify | bool | `true` | 是否跳过 TLS 证书校验（用于自签名 EPP 证书） |
| dynamo-operator.dynamo.metrics.prometheusEndpoint | string | `""` | 服务用于获取指标的端点。如果设置，dynamo operator 会自动向其管理的服务注入 PROMETHEUS_ENDPOINT 环境变量。用户也可以通过修改对应 deployment 的环境变量来覆盖该值。 |
| dynamo-operator.dynamo.mpiRun.secretName | string | `"mpi-run-ssh-secret"` | 存放 MPI Run SSH key 的 Secret 名称 |
| dynamo-operator.webhook.certificateSecret.name | string | `"webhook-server-cert"` | 存放 webhook TLS 证书的 Kubernetes Secret 名称。该 Secret 必须包含三个键：tls.crt（服务端证书）、tls.key（服务端私钥）、ca.crt（CA 证书）。 |
| dynamo-operator.webhook.certificateSecret.external | bool | `false` | 是否由外部管理证书 Secret。设为 false（默认）时，operator 内置的 cert-controller 自动生成并轮换证书；设为 true 时，你必须在安装 chart 之前手动创建该 Secret。 |
| dynamo-operator.webhook.caBundle | string | `""` | 用于 webhook 校验的 CA bundle（base64 编码）。仅在 certificateSecret.external=true 时使用。若使用自动证书生成或 cert-manager 集成，请留空，CA bundle 会被自动注入。 |
| dynamo-operator.webhook.failurePolicy | string | `"Fail"` | webhook 失败策略：当 webhook 不可用时 Kubernetes 如何处理请求。`Fail`（生产推荐）会在 webhook 不可达时拒绝请求，确保校验严格；`Ignore` 会在 webhook 不可用时放行请求，以可用性换取校验保证。 |
| dynamo-operator.webhook.timeoutSeconds | int | `10` | webhook 校验调用的超时（秒）。若 webhook 在该时间内未响应，将按 failurePolicy 处理请求。 |
| dynamo-operator.webhook.namespaceSelector | object | `{}` | 自定义 webhook 校验的命名空间选择器。可用于把特定命名空间纳入或排除在 webhook 校验之外。对**集群级**operator，可使用：matchExpressions: [{ key: "dynamo-operator", operator: "NotIn", values: ["namespace-restricted"] }] 排除被命名空间级 operator 管理的命名空间。对**命名空间级**operator 留空即可，会自动配置为只匹配该 operator 自己的命名空间。 |
| dynamo-operator.webhook.certManager.enabled | bool | `false` | 是否使用 cert-manager 自动管理证书。要求集群已安装 cert-manager。启用后由 cert-manager 颁发并轮换证书，而非 operator 内置的 cert-controller。 |
| dynamo-operator.webhook.certManager.certificate.duration | string | `"8760h"` | 由 cert-manager 管理的 webhook 证书的有效期（如 1 年使用 "8760h"）。cert-manager 会在到期前自动续期。 |
| dynamo-operator.webhook.certManager.certificate.renewBefore | string | `"360h"` | 距离到期多久触发续期（如 15 天用 "360h"）。达到该阈值时 cert-manager 会尝试续期。 |
| dynamo-operator.webhook.certManager.certificate.rootCA.duration | string | `"87600h"` | 根 CA 证书的有效期（如 10 年用 "87600h"）。根 CA 通常比其签发的叶子证书寿命长得多。 |
| dynamo-operator.webhook.certManager.certificate.rootCA.renewBefore | string | `"720h"` | 距离根 CA 到期多久触发续期（如 30 天用 "720h"）。续期 CA 可能造成中断，因为所有由其签发的证书都需要重发。 |
| dynamo-operator.checkpoint.enabled | bool | `false` | 是否启用 checkpoint/restore 功能 |
| dynamo-operator.checkpoint.storage | object | `{}` | 当 snapshot-agent 安装在工作负载命名空间之外（snapshot.storage.accessMode=podMount）时，可选的 PVC 存储。create=true 时由 operator 在该命名空间中托管 PVC；省略或 create=false 时则要求已存在指定名称的 PVC。podMount 下使用 ReadWriteOnce 适用于在合适的存储后端上做顺序 checkpoint/restore；并发的多节点访问请使用 ReadWriteMany。 |
| grove.tolerations | list | `[]` | Grove pod 的节点 tolerations |
| grove.affinity | object | `{}` | Grove pod 的 affinity |
| kai-scheduler.global.tolerations | list | `[]` | kai-scheduler pod 的节点 tolerations |
| kai-scheduler.global.affinity | object | `{}` | kai-scheduler pod 的 affinity |
| etcd.image.repository | string | `"bitnamilegacy/etcd"` | 由于 bitnami 公告 brownout - https://github.com/bitnami/charts/tree/main/bitnami/etcd#%EF%B8%8F-important-notice-upcoming-changes-to-the-bitnami-catalog ，在我们迁移到新的 "secure" repository 之前需要使用 legacy repository。 |

### NATS 配置

除 `global.nats.install` 之外更详细的 NATS 配置项，请参阅 NATS Helm chart 官方文档：
**[NATS Helm Chart 文档](https://github.com/nats-io/k8s/tree/main/helm/charts/nats)**

### etcd 配置

Dynamo 平台**已不再依赖** etcd。Operator 默认使用 Kubernetes 原生的服务发现，内置的 etcd 子 chart **默认关闭**。

启用内置的 etcd 子 chart（例如用于基于 etcd 的服务发现）：

```yaml
global:
  etcd:
    install: true
```

或者使用外部 etcd 实例：

```yaml
dynamo-operator:
  etcdAddr: "http://my-external-etcd:2379"
```

更详细的 etcd 配置项请参阅 Bitnami etcd Helm chart 官方文档：
**[etcd Helm Chart 文档](https://github.com/bitnami/charts/tree/main/bitnami/etcd)**

### Kai Scheduler 与 Grove 配置

在**生产环境**中，Kai Scheduler 与 Grove 应**与本 chart 独立安装**，以便单独管理生命周期、固定版本与升级。

**兼容性矩阵：**

| dynamo-platform | kai-scheduler | Grove |
|-----------------|---------------|-------|
| 1.0.x           | >= v0.13.0    | >= v0.1.0-alpha.6 |
| 1.1.x           | >= v0.13.4    | >= v0.1.0-alpha.8 |

独立安装它们之后，启用 Dynamo 集成：

```yaml
global:
  kai-scheduler:
    enabled: true   # Enables queue creation and schedulerName injection
  grove:
    enabled: true   # Enables multinode orchestration via PodCliqueSets
```

**仅用于开发/测试**时，可以把它们作为内置子 chart 一起部署：

```yaml
global:
  kai-scheduler:
    install: true   # Deploys the bundled kai-scheduler subchart (integration auto-enabled)
  grove:
    install: true   # Deploys the bundled Grove subchart (integration auto-enabled)
```

注意：`global.*.install` 控制是否部署内置子 chart，开启后集成会自动启用；`global.*.enabled` 在使用外部安装时可独立设置。

## 📚 更多资源

- [Dynamo Cloud 部署安装指南](../../../../docs/kubernetes/installation-guide.md)
- [NATS 文档](https://docs.nats.io/)
- [etcd 文档](https://etcd.io/docs/)
- [Kubernetes Operator 模式](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)

----------------------------------------------
由 chart metadata 经 [helm-docs v1.14.2](https://github.com/norwoodj/helm-docs/releases/v1.14.2) 自动生成
