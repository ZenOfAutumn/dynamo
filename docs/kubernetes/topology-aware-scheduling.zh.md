---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Topology Aware Scheduling
---

拓扑感知调度（Topology Aware Scheduling，TAS）允许你按照集群的网络拓扑来控制 Dynamo 推理工作负载 pod 的放置位置。把相关 pod 打包到同一机架（rack）、block 或其他拓扑域内，可以降低跨节点延迟、提升吞吐——尤其适用于解耦服务（disaggregated serving），其中 prefill、decode 与路由组件之间通信频繁。

TAS 是 **可选启用** 的。已有的、未配置拓扑约束的部署会保持原状不受影响。

## 前置要求

| 要求 | 说明 |
|-------------|---------|
| **Grove** | 已在集群上安装。详见 [Grove Installation Guide](https://github.com/NVIDIA/grove/blob/main/docs/installation.md)。 |
| **ClusterTopology CR** | 由集群管理员配置的集群级 `ClusterTopology` 资源，把拓扑域名映射到节点 label。配置说明参见 [Grove documentation](https://github.com/NVIDIA/grove)。 |
| **KAI Scheduler** | 拓扑感知 pod 放置由 Grove 委托 [KAI Scheduler](https://github.com/NVIDIA/KAI-Scheduler) 实现。 |
| **Dynamo operator** | 最新的 Dynamo operator Helm chart 通过专用 ClusterRole 包含对 `clustertopologies.grove.io` 的只读 RBAC，无需额外配置。 |

## 拓扑域

拓扑域是 **自由形式** 的标识符，由集群管理员在 `ClusterTopology` CR 中定义。常见示例包括 `region`、`zone`、`datacenter`、`block`、`rack`、`host`、`numa`，但任何匹配 `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$` 模式的名字都合法（不能以连字符开头或结尾）。

域名必须与 `topologyProfile` 引用的 `ClusterTopology` CR 中所配置的完全一致。在创建 DGD 时，Dynamo webhook 会校验每个 `packDomain` 在所引用的 `ClusterTopology` 中存在。

当你指定 `packDomain` 时，调度器会把受约束组件的所有副本打包在该域的同一实例中。例如 `packDomain: rack` 表示"把所有 pod 放到同一机架内"。

## 拓扑 Profile

任何使用拓扑约束的 DGD 都必须通过 `topologyProfile` 字段按名称引用一个 `ClusterTopology` CR。该字段位于 `spec.topologyConstraint`（部署级）并由所有服务继承——服务自身不应再设置 `topologyProfile`。

`topologyProfile` 告诉 Dynamo operator 与底层框架在调度与校验时使用哪一个拓扑层级。

## 在 DGD 上启用 TAS

在你的 `DynamoGraphDeployment` 上，于部署级、服务级或两者同时添加 `topologyConstraint` 字段。部署级必须包含 `topologyProfile`。每个约束指定一个 `packDomain`。

### 示例 1：部署级约束（服务继承）

所有服务都继承部署级约束。当你想要统一的拓扑打包时，这是最简单的配置。

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-llm
spec:
  topologyConstraint:
    topologyProfile: my-cluster-topology
    packDomain: zone
  services:
    VllmWorker:
      componentType: worker
      replicas: 2
      envFromSecret: hf-token-secret
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        mainContainer:
          image: my-image
          command: ["/bin/sh", "-c"]
          args:
            - python3 -m dynamo.vllm --model Qwen/Qwen3-0.6B
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: my-image
          command: ["/bin/sh", "-c"]
          args:
            - python3 -m dynamo.frontend
```

### 示例 2：仅服务级约束

只有指定的服务获得拓扑打包。其他服务在无拓扑约束下调度。部署级仍必须设置 `topologyProfile`。

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-llm
spec:
  topologyConstraint:
    topologyProfile: my-cluster-topology
  services:
    VllmWorker:
      componentType: worker
      replicas: 2
      multinode:
        nodeCount: 4
      topologyConstraint:
        packDomain: rack
      envFromSecret: hf-token-secret
      resources:
        limits:
          gpu: "8"
      extraPodSpec:
        mainContainer:
          image: my-image
          command: ["/bin/sh", "-c"]
          args:
            - python3 -m dynamo.vllm --model meta-llama/Llama-4-Maverick-17B-128E
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: my-image
          command: ["/bin/sh", "-c"]
          args:
            - python3 -m dynamo.frontend
```

### 示例 3：混合（部署级默认 + 单服务覆盖）

在部署级设置宽泛约束，并在特定服务上设置更窄的覆盖。服务级约束必须与部署级约束 **相等或更窄**（由 `ClusterTopology` CR 中层级的顺序决定）。

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-llm
spec:
  topologyConstraint:
    topologyProfile: my-cluster-topology
    packDomain: zone
  services:
    VllmWorker:
      componentType: worker
      replicas: 2
      multinode:
        nodeCount: 4
      topologyConstraint:
        packDomain: block    # narrower than zone — valid
      envFromSecret: hf-token-secret
      resources:
        limits:
          gpu: "8"
      extraPodSpec:
        mainContainer:
          image: my-image
          command: ["/bin/sh", "-c"]
          args:
            - python3 -m dynamo.vllm --model meta-llama/Llama-4-Maverick-17B-128E
    Frontend:
      componentType: frontend
      replicas: 1
      # inherits zone from spec.topologyConstraint
      extraPodSpec:
        mainContainer:
          image: my-image
          command: ["/bin/sh", "-c"]
          args:
            - python3 -m dynamo.frontend
```

## 层级规则

当 **同时** 设置部署级与服务级 `topologyConstraint` 时，服务的 `packDomain` 必须 **相等或更窄** 于部署级 `packDomain`。"更窄"由 `ClusterTopology` CR 中层级的顺序决定——`spec.levels` 数组中越靠后的层级越窄。

如果服务约束比部署约束更宽（在与某个 `ClusterTopology` CR 校验时），Dynamo webhook 会在创建 DGD 时拒绝它。

当只设置一级（仅部署级或仅服务级）时，不进行层级检查。

| 配置 | 行为 |
|---------------|----------|
| 设置了 `spec.topologyConstraint`，服务未设置 | 服务继承部署级约束 |
| 同时设置了 `spec.topologyConstraint` 与服务级 | 都生效；服务必须更窄或相等 |
| 设置了 `spec.topologyConstraint.topologyProfile`，spec 未设 `packDomain` | profile 仅供服务级约束使用 |
| 都未设置 | 没有拓扑约束（默认） |

## 字段参考

| 字段 | 层级 | 必填 | 说明 |
|-------|-------|----------|-------------|
| `topologyProfile` | `spec.topologyConstraint` | 当设置任意约束时必填 | 定义拓扑层级的 `ClusterTopology` CR 名称。 |
| `topologyProfile` | 服务级 `topologyConstraint` | 不适用（schema 中无） | 从 `spec.topologyConstraint` 继承。服务级类型不包含此字段。 |
| `packDomain` | `spec.topologyConstraint` | 可选 | 没有自定义 packDomain 的服务的默认 packDomain。 |
| `packDomain` | 服务级 `topologyConstraint` | 必填 | 该服务的 packDomain。必须与 `ClusterTopology` CR 中的某一层级匹配。 |

## 多节点（Multinode）注意事项

对于多节点服务（包含 `multinode` 段的服务），拓扑约束会施加在 **scaling group** 层级，而不是单个 worker pod 上。这一点很重要，因为多节点服务会派生 `replicas × nodeCount` 个 pod——例如 2 副本 + `nodeCount: 4` 会跨 8 个节点产生 8 个 pod。把约束施加在 scaling group 层级意味着调度器为每个副本的整套节点在所请求的域内打包，而不会过度约束单个 pod 到同一台主机。

例如，在以下配置中：

```yaml
VllmWorker:
  replicas: 2
  multinode:
    nodeCount: 4
  topologyConstraint:
    packDomain: rack
```

每个副本的 4 个节点会被打包在同一机架内。两个副本可能落到不同机架（约束按副本应用，而不是跨所有副本）。

**建议：** 对多节点服务，使用 `rack` 或 `block` 作为 `packDomain`，把 worker 保留在高带宽域内，同时仍允许调度器在该域内跨主机分布。多节点服务避免使用 `host`，因为把多个节点装到一台主机上没有意义。

## 不可变性

拓扑约束 **在 DGD 创建后不能被修改**。这包括：

- 给原本没有约束的 DGD 或服务添加拓扑约束
- 删除已有的拓扑约束
- 修改 `topologyProfile` 取值
- 修改 `packDomain` 取值

要修改拓扑约束，请 **删除并重建** DGD。这与底层框架一致——后者会对所生成资源的拓扑约束强制不可变。

## 拓扑生效情况监控

当设置了任何拓扑约束时，DGD status 中会包含 `TopologyLevelsAvailable` condition，报告你约束所引用的拓扑层级是否仍存在于集群拓扑中。

**健康状态：**

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
    - type: TopologyLevelsAvailable
      status: "True"
      reason: AllTopologyLevelsAvailable
      message: "All required topology levels are available in the cluster topology"
```

**降级状态**（例如管理员在部署后从 `ClusterTopology` CR 中移除了某层级）：

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
    - type: TopologyLevelsAvailable
      status: "False"
      reason: TopologyLevelsUnavailable
      message: "Topology level 'rack' is no longer available in the cluster topology"
```

当拓扑层级变为不可用时，Dynamo 会在 DGD 上发出 **Warning** 事件。由于底层框架仍维持 pod 运行，部署可能仍显示为 `Ready`，但拓扑放置不再被保证。

## 故障排查

### DGD 被拒绝："ClusterTopology not found"

当设置了任何拓扑约束时，Dynamo webhook 会校验 `topologyProfile` 引用的 `ClusterTopology` CR 是否存在。如果它读取不到该 CR：

- 确认集群管理员已创建以 `topologyProfile` 命名的 `ClusterTopology` 资源。配置参见 [Grove documentation](https://github.com/NVIDIA/grove)。
- 确认 Dynamo operator 拥有读取 `clustertopologies.grove.io` 的 RBAC（默认 Helm chart 已包含）。

### DGD 被拒绝："packDomain not found in cluster topology"

指定的 `packDomain` 在引用的 `ClusterTopology` CR 中不存在。可查看已定义的域：

```bash
kubectl get clustertopology <topology-profile-name> -o yaml
```

确保你请求的域（例如 `rack`）已在 `ClusterTopology` 中配置，且具有对应的节点 label。

### DGD 被拒绝："topologyProfile is required"

任何具有拓扑约束（spec 或服务级）的 DGD 都必须把 `spec.topologyConstraint.topologyProfile` 设为某个 `ClusterTopology` CR 的名称。请在 `spec.topologyConstraint` 中加上 `topologyProfile` 字段。

### Pod 卡在 Pending

调度器无法满足拓扑约束。常见原因：

- 所请求域的单一实例内节点数不足（例如要求把 8 个 GPU 打包在同一机架，但没有任何机架有 8 个可用 GPU）。
- 节点 label 与 `ClusterTopology` 配置不一致。

查看调度器事件了解细节：

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### TopologyLevelsAvailable 为 False

DGD 已成功部署，但拓扑定义后来被改变。底层框架检测到一个或多个所需拓扑层级不再可用。

- 查看 condition 的 message 获取细节。
- 查看 `ClusterTopology` CR 是否有域被删除或重命名。
- 如果是有意修改拓扑，请删除并重建 DGD 以采用新拓扑。

### DGD 被拒绝：层级违规

服务级 `packDomain` 比部署级 `packDomain` 更宽。"更宽"与"更窄"由 `ClusterTopology` CR 中层级的顺序决定——`spec.levels` 中越靠前的层级越宽。

请确保服务级约束相等或更窄于部署级约束。
