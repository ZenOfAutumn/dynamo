# Operator 默认值注入

当用户在部署中没有显式指定某些字段时，Dynamo operator 会自动为它们注入默认值。
这些默认值包括：

- **健康探针（Health Probes）**：startup、liveness、readiness 探针针对前端
  （frontend）、worker 与 planner 组件配置不同。例如 worker 组件会获得一个
  超时时长 2 小时（720 次失败 × 10 秒）的 startup 探针，以容纳长时间的模型
  加载。

- **安全上下文（Security Context）**：所有组件默认都会获得 `fsGroup: 1000`，
  以保证挂载卷上的文件权限正确。可通过 `extraPodSpec.securityContext` 覆盖。

- **共享内存**：所有组件默认都会获得一份挂载到 `/dev/shm` 的 8Gi 共享内存卷
  （可通过 `sharedMemory` 字段禁用或调整大小）。

- **环境变量**：组件会自动获得 `DYN_NAMESPACE`、`DYN_PARENT_DGD_K8S_NAME`、
  `DYNAMO_PORT` 以及与具体后端（backend）相关的环境变量。

- **Pod 配置**：默认 `terminationGracePeriodSeconds` 为 60 秒，
  `restartPolicy: Always`。

- **自动伸缩**：在启用自动伸缩但未指定指标（metrics）时，默认按 CPU 进行伸缩，
  目标利用率 80%。

- **后端特定行为**：对于多节点部署，会根据后端框架（VLLM、SGLang 或
  TensorRT-LLM）自动修改或移除 worker 节点的探针。

## Pod 规范默认值

除非显式覆盖，所有组件都会获得以下 pod 级默认值：

- **`terminationGracePeriodSeconds`**：`60` 秒
- **`restartPolicy`**：`Always`

## 安全上下文

operator 会自动为所有组件应用默认的安全上下文设置，以保证挂载卷上的文件权限
正确：

- **`fsGroup`**：`1000` —— 设置挂载卷及其中所创建文件的组属主

该默认值确保非 root 容器能够顺利写入挂载卷（如模型缓存或持久化存储）而不会
出现权限问题。`fsGroup` 设置在以下场景下尤为重要：
- 模型下载与缓存
- 编译缓存目录
- 持久化卷声明（PVC）
- 多节点部署中的 SSH key 生成

### 覆盖安全上下文

要覆盖默认的安全上下文，请在组件的 `extraPodSpec` 中指定自己的
`securityContext`：

```yaml
services:
  YourWorker:
    extraPodSpec:
      securityContext:
        fsGroup: 2000  # Custom group ID
        runAsUser: 1000
        runAsGroup: 1000
        runAsNonRoot: true
```

**重要：** 一旦你在 `extraPodSpec` 中提供了**任意**一个 `securityContext`
对象，operator 就不会再注入任何默认值。这让你完全掌控安全上下文，包括允许以
root 运行（通过省略 `runAsNonRoot` 或将其设为 `false`）。

### OpenShift 与安全上下文约束

在启用了 Security Context Constraints（SCC）的 OpenShift 环境中，你可能需要
省略显式的 UID/GID，让 OpenShift 的准入控制器（admission controllers）动态
分配：

```yaml
services:
  YourWorker:
    extraPodSpec:
      securityContext:
        # 省略 fsGroup，让 OpenShift 根据 SCC 自动分配
        # OpenShift 将注入合适的 UID 段
```

或者，如果你希望保留默认的 `fsGroup: 1000` 行为且确认集群允许，那么不需要
做任何额外配置 —— operator 的默认值即可工作。

## 共享内存配置

所有组件默认启用共享内存：

- **启用**：`true`（除非通过 `sharedMemory.disabled` 显式禁用）
- **大小**：`8Gi`
- **挂载路径**：`/dev/shm`
- **卷类型**：`emptyDir`，介质为 `memory`

要禁用共享内存或自定义大小，请在组件规范中使用 `sharedMemory` 字段。

## 各组件类型的健康探针

operator 会针对不同组件类型注入不同的默认健康探针。

### 前端（Frontend）组件

前端组件会获得如下探针配置：

**Liveness Probe（存活探针）：**
- **类型**：HTTP GET
- **路径**：`/health`
- **端口**：`http`（8000）
- **初始延迟**：60 秒
- **周期**：60 秒
- **超时**：30 秒
- **失败阈值**：10

**Readiness Probe（就绪探针）：**
- **类型**：Exec 命令
- **命令**：`curl -s http://localhost:${DYNAMO_PORT}/health | jq -e ".status == \"healthy\""`
- **初始延迟**：60 秒
- **周期**：60 秒
- **超时**：30 秒
- **失败阈值**：10

### Worker 组件

worker 组件会获得如下探针配置：

**Liveness Probe：**
- **类型**：HTTP GET
- **路径**：`/live`
- **端口**：`system`（9090）
- **周期**：5 秒
- **超时**：30 秒
- **失败阈值**：1

**Readiness Probe：**
- **类型**：HTTP GET
- **路径**：`/health`
- **端口**：`system`（9090）
- **周期**：10 秒
- **超时**：30 秒
- **失败阈值**：60

**Startup Probe（启动探针）：**
- **类型**：HTTP GET
- **路径**：`/live`
- **端口**：`system`（9090）
- **周期**：10 秒
- **超时**：5 秒
- **失败阈值**：720（允许最多 2 小时启动：10s × 720 = 7200s）

:::{note}
对于较大的模型（通常 > 70B 参数）或较慢的存储系统，你可能需要增加
`failureThreshold` 以容纳更长的模型加载时间。可按预期启动时间计算所需阈值：
`failureThreshold = (expected_startup_seconds / period)`。如果默认 2 小时
窗口不够，请在组件规范中覆盖 startup 探针。
:::

### 多节点部署中的探针修改

对于多节点部署，operator 会根据后端框架与节点角色调整探针：

#### VLLM 后端

operator 会基于并行配置自动选择两种部署模式之一：

**张量 / 流水并行模式**（当 `world_size > GPUs_per_node`）：
- 使用 Ray 进行分布式执行（`--distributed-executor-backend ray`）
- **Leader 节点**：启动 Ray head 并运行 vLLM；所有探针保持启用
- **Worker 节点**：仅运行 Ray agent；所有探针（liveness、readiness、startup）
  都被移除

**数据并行模式**（当 `world_size × data_parallel_size > GPUs_per_node`）：
- **Worker 节点**：所有探针（liveness、readiness、startup）都被移除
- **Leader 节点**：所有探针保持启用

#### SGLang 后端
- **Worker 节点**：所有探针（liveness、readiness、startup）都被移除

#### TensorRT-LLM 后端
- **Leader 节点**：所有探针保持不变
- **Worker 节点**：
  - liveness 与 startup 探针被移除
  - readiness 探针被替换为对 SSH 端口（2222）的 TCP socket 检查：
    - **初始延迟**：20 秒
    - **周期**：20 秒
    - **超时**：5 秒
    - **失败阈值**：10

## 环境变量

operator 会根据组件类型、后端框架以及 operator 配置，自动向组件容器中注入
环境变量。用户提供的 `envs` 取值始终优先于 operator 默认值。

### 所有组件

无论组件类型如何，以下环境变量都会被注入到所有组件容器：

| 变量 | 用途 | 默认值 | 类型 | 来源 |
| --- | --- | --- | --- | --- |
| `DYN_NAMESPACE` | 用于服务发现与路由的 Dynamo 服务命名空间 | 由 DGD spec 推导 | `string` | 在 checkpoint 恢复的 pod 上通过 Downward API annotation 注入 |
| `DYN_COMPONENT` | 标识组件类型，用于运行时行为 | 取值之一：`frontend`, `worker`, `prefill`, `decode`, `planner`, `epp` | `string` | 由组件 spec 设置 |
| `DYN_PARENT_DGD_K8S_NAME` | 父 DynamoGraphDeployment 资源的 Kubernetes 名称 | — | `string` | 由 DGD metadata 设置 |
| `DYN_PARENT_DGD_K8S_NAMESPACE` | 父 DynamoGraphDeployment 资源的 Kubernetes 命名空间 | — | `string` | 由 DGD metadata 设置 |
| `POD_NAME` | 当前 pod 名称 | — | `string` | Downward API（`metadata.name`） |
| `POD_NAMESPACE` | 当前 pod 命名空间 | — | `string` | Downward API（`metadata.namespace`） |
| `POD_UID` | 当前 pod UID | — | `string` | Downward API（`metadata.uid`） |
| `DYN_DISCOVERY_BACKEND` | 组件间通信使用的服务发现后端 | `kubernetes` | `string` | 取值：`kubernetes`、`etcd` |

### 基础设施（条件注入）

当 operator 的 `OperatorConfiguration` 中配置了相应的基础设施服务时，下列变量
会被注入到所有组件中：

| 变量 | 用途 | 默认值 | 类型 | 注入条件 |
| --- | --- | --- | --- | --- |
| `NATS_SERVER` | NATS 消息服务器地址 | — | `string` | 当配置了 `infrastructure.natsAddress` 时 |
| `ETCD_ENDPOINTS` | 用于分布式状态的 etcd 端点地址 | — | `string` | 当配置了 `infrastructure.etcdAddress` 时 |
| `MODEL_EXPRESS_URL` | 用于模型管理的 Model Express 服务 URL | — | `string` | 当配置了 `infrastructure.modelExpressURL` 时 |
| `PROMETHEUS_ENDPOINT` | 用于指标采集的 Prometheus 端点 | — | `string` | 当配置了 `infrastructure.prometheusEndpoint` 时 |

### 前端组件

| 变量 | 用途 | 默认值 | 类型 |
| --- | --- | --- | --- |
| `DYNAMO_PORT` | 前端监听的 HTTP 端口 | `8000` | `int` |
| `DYN_HTTP_PORT` | 前端服务的 HTTP 端口（别名） | `8000` | `int` |
| `DYN_NAMESPACE_PREFIX` | 用于前端请求路由的命名空间前缀 | 与 `DYN_NAMESPACE` 相同 | `string` |

### Worker 组件

| 变量 | 用途 | 默认值 | 类型 |
| --- | --- | --- | --- |
| `DYN_SYSTEM_ENABLED` | 启用用于健康检查与指标的系统 HTTP server | `true` | `string`（boolean） |
| `DYN_SYSTEM_USE_ENDPOINT_HEALTH_STATUS` | 用作 readiness 依据的端点的健康状态 | `["generate"]` | `string`（JSON 数组） |
| `DYN_SYSTEM_PORT` | 系统 HTTP server 端口（健康、指标） | `9090` | `int` |
| `DYN_HEALTH_CHECK_ENABLED` | 关闭旧的健康检查机制，改用 system server | `false` | `string`（boolean） |
| `NIXL_TELEMETRY_ENABLE` | 启用 / 关闭 NIXL 遥测采集 | `n` | `string` | 取值：`y`、`n` |
| `NIXL_TELEMETRY_EXPORTER` | NIXL 指标的遥测 exporter 格式 | `prometheus` | `string` |
| `NIXL_TELEMETRY_PROMETHEUS_PORT` | NIXL Prometheus 指标端点端口 | `19090` | `int` |
| `DYN_NAMESPACE_WORKER_SUFFIX` | 在滚动更新过渡期间附加到 worker namespace 的哈希后缀 | — | `string` | 仅在滚动更新过渡时设置 |

### Planner 组件

| 变量 | 用途 | 默认值 | 类型 |
| --- | --- | --- | --- |
| `PLANNER_PROMETHEUS_PORT` | planner 的 Prometheus 指标端点端口 | `9085` | `int` |

### EPP（Endpoint Picker Plugin）组件

| 变量 | 用途 | 默认值 | 类型 |
| --- | --- | --- | --- |
| `USE_STREAMING` | 启用推理请求代理的流式模式 | `true` | `string`（boolean） |
| `RUST_LOG` | Rust 日志级别与过滤器配置 | `debug,dynamo_llm::kv_router=trace` | `string` |

### VLLM 后端

| 变量 | 用途 | 默认值 | 类型 | 注入条件 |
| --- | --- | --- | --- | --- |
| `VLLM_CACHE_ROOT` | vLLM 编译缓存产物目录 | — | `string` | 当某个 volume mount 设置了 `useAsCompilationCache: true` 时 |
| `VLLM_NIXL_SIDE_CHANNEL_HOST` | 多进程模式下 NIXL side channel 的 Host IP | Pod IP | `string` | 仅多节点 mp 后端（Downward API：`status.podIP`） |

### TensorRT-LLM 后端

| 变量 | 用途 | 默认值 | 类型 | 注入条件 |
| --- | --- | --- | --- | --- |
| `OMPI_MCA_orte_keep_fqdn_hostnames` | 让 OpenMPI 在节点间通信时保留 FQDN 主机名 | `1` | `string` | 仅多节点部署 |

## 服务账号（Service Account）

以下组件类型会自动获得专属的 service account：

- **Planner**：`planner-serviceaccount`
- **EPP**：`epp-serviceaccount`

## Image Pull Secrets

operator 会自动发现并为容器镜像注入 image pull secret。当组件指定了一个容器
镜像后，operator 会：

1. 扫描组件命名空间中所有类型为 `kubernetes.io/dockerconfigjson` 的 Secret
2. 从每个 Secret 的认证配置中提取 docker registry 服务器 URL
3. 把容器镜像所属的 registry host 与已发现的 registry URL 进行匹配
4. 自动把匹配到的 secret 注入到 pod 规范的 `imagePullSecrets` 中

这样就免去了为每个组件手工指定 image pull secret 的麻烦。operator 会维护一个
内部的 docker secret 与对应 registry 的索引，并定期刷新。

**要为某个组件关闭 image pull secret 自动发现**，请添加以下 annotation：

```yaml
annotations:
  nvidia.com/disable-image-pull-secret-discovery: "true"
```

## 自动伸缩默认值

启用自动伸缩但未指定指标时，operator 会应用：

- **默认指标**：CPU 利用率
- **目标平均利用率**：`80%`

## 端口配置

各组件类型默认配置的容器端口如下：

### 前端组件
- **端口**：8000
- **协议**：TCP
- **名称**：`http`

### Worker 组件
- **端口**：9090（system）
- **协议**：TCP
- **名称**：`system`
- **端口**：19090（NIXL）
- **协议**：TCP
- **名称**：`nixl`

### Planner 组件
- **端口**：9085
- **协议**：TCP
- **名称**：`metrics`

### EPP 组件
- **端口**：9002（gRPC）
- **协议**：TCP
- **名称**：`grpc`
- **端口**：9003（gRPC health）
- **协议**：TCP
- **名称**：`grpc-health`
- **端口**：9090（metrics）
- **协议**：TCP
- **名称**：`metrics`

## 后端相关配置

### VLLM
- **Ray Head 端口**：6379（用于多节点 TP/PP 部署中的 Ray 集群协调）
- **数据并行 RPC 端口**：13445（用于多节点数据并行部署）

### SGLang
- **Distribution Init 端口**：29500（用于多节点部署）

### TensorRT-LLM
- **SSH 端口**：2222（用于多节点 MPI 通信）
- **OpenMPI 环境**：`OMPI_MCA_orte_keep_fqdn_hostnames=1`

## 实现参考

如果你希望了解实现细节或为 operator 贡献代码，本文档中所述默认值的实现位于
以下源代码：

- **健康探针、安全上下文与 Pod 规范**：[`internal/dynamo/graph.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/dynamo/graph.go) —— 包含应用默认探针、安全上下文、环境变量、共享内存与 pod 配置的主要逻辑
- **组件特定默认值**：
  - [`internal/dynamo/component_common.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/dynamo/component_common.go) —— 所有组件共享的基础容器与 pod spec
  - [`internal/dynamo/component_frontend.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/dynamo/component_frontend.go)
  - [`internal/dynamo/component_worker.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/dynamo/component_worker.go)
  - [`internal/dynamo/component_planner.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/dynamo/component_planner.go)
  - [`internal/dynamo/component_epp.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/dynamo/component_epp.go)
- **Image Pull Secrets**：[`internal/secrets/docker.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/secrets/docker.go) —— 实现 docker secret 索引器与自动发现
- **后端相关行为**：
  - [`internal/dynamo/backend_vllm.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/dynamo/backend_vllm.go)
  - [`internal/dynamo/backend_sglang.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/dynamo/backend_sglang.go)
  - [`internal/dynamo/backend_trtllm.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/dynamo/backend_trtllm.go)
- **Checkpoint / Restore**：
  - [`internal/checkpoint/podspec.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/checkpoint/podspec.go) —— Checkpoint 环境变量注入与卷设置
  - [`internal/checkpoint/resolve.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/checkpoint/resolve.go) —— Checkpoint 解析逻辑
  - [`internal/checkpoint/resource.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/checkpoint/resource.go) —— Checkpoint 资源管理
- **常量与 Annotation**：[`internal/consts/consts.go`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/operator/internal/consts/consts.go) —— 定义 annotation key 与其他常量

## 注意事项

- 上述所有默认值都可以通过在 DynamoComponentDeployment 或
  DynamoGraphDeployment 资源中显式指定来覆盖
- 用户指定的探针（通过 `livenessProbe`、`readinessProbe` 或 `startupProbe`
  字段）优先于 operator 默认值
- 对于安全上下文，只要在 `extraPodSpec` 中提供了**任意** `securityContext`，
  默认值就不会被注入，从而把控制权完全交给你
- 对于多节点部署，部分默认值会按上文所述被修改或移除，以适配分布式执行模式
- 可以通过 `extraPodSpec.mainContainer` 字段覆盖 operator 设置的探针配置
