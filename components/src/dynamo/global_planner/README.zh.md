<!-- # SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0 -->

# Global Planner

面向多 DGD planner 部署的集中式扩缩容执行服务。

Global Planner 接收来自各本地 planner 的扩缩容决策，并对 Kubernetes `DynamoGraphDeployment` 资源执行副本数更新。无论多个 DGD 是否共享同一对外端点，只要它们希望通过一个集中式组件来委托扩缩容，该组件都很有用。

## 它解决了什么问题

如果没有 `GlobalPlanner`，每个 DGD 的本地 planner 只能直接对自己的部署进行扩缩容。
对于隔离的部署来说这没问题，但当你希望由一个集中点来完成以下事情时就会变得很尴尬：

- 在多个 DGD 之间应用集中式扩缩容策略
- 强制执行共享约束，例如鉴权或总 GPU 预算
- 协调单端点、多池（pool）部署中的扩缩容

`GlobalPlanner` 通过成为多个本地 planner 的公共扩缩容执行端点来解决上述问题。

## 部署模式

`GlobalPlanner` 通常用于两种模式：

1. **跨独立 DGD 的集中扩缩容**
   每个 DGD 仍保留各自的常规本地 planner，但本地 planner 将扩缩容执行委托给同一个 `GlobalPlanner`。当多个独立部署或模型需要共享某项全局策略（例如总 GPU 预算）时，这种模式非常有用。该模式**不需要** `GlobalRouter` 或共享的对外端点。
2. **分层式单端点部署**
   同一模型的多个池 DGD 共享一个对外的 `Frontend` 与一个 `GlobalRouter`。每个池仍拥有各自的本地 planner，并将扩缩容委托给 `GlobalPlanner`。

## 术语

- **SLA Planner**：常规的 `dynamo.planner` 组件，依据 SLA 目标、profile 和/或指标（metrics）计算期望的副本数。
- **本地 planner**：运行在某个 DGD 或某个池内的 planner 实例。
- **GlobalPlanner**：集中式的执行与策略层，负责接收来自本地 planner 的扩缩容请求，并将其应用到目标 DGD 上。
- **分层 planner（Hierarchical planner）**：架构层面的术语，并非独立的二进制文件。在实践中是指多个本地 planner 都将请求转发给同一个 `GlobalPlanner`，并经常与 `GlobalRouter` 一同使用。

## 概览

- 暴露一个供 planner 委托使用的远程扩缩容端点
- 可选地对调用方命名空间进行鉴权
- 通过 `KubernetesConnector` 执行扩缩容操作
- 返回操作状态以及观察到的副本数
- 通过 `--no-operation` 支持 dry-run 模式

## 运行时端点

给定 `DYN_NAMESPACE=<ns>`，本组件会提供以下服务：

- `<ns>.GlobalPlanner.scale_request`
- `<ns>.GlobalPlanner.health`

`health` 返回：

- `status`（`healthy`）
- `component`（`GlobalPlanner`）
- `namespace`
- `managed_namespaces`（在未启用鉴权时为 `all`）

## 用法

### 命令行

```bash
# Accept scale requests from any namespace
DYN_NAMESPACE=global-infra python -m dynamo.global_planner
```

```bash
# Restrict requests to specific planner namespaces
DYN_NAMESPACE=global-infra python -m dynamo.global_planner \
  --managed-namespaces app-ns-1 app-ns-2
```

```bash
# Dry-run mode (no Kubernetes updates)
DYN_NAMESPACE=global-infra python -m dynamo.global_planner --no-operation
```

```bash
# Enforce a maximum total GPU budget across managed pools
DYN_NAMESPACE=global-infra python -m dynamo.global_planner --max-total-gpus 16
```

### 参数

必需的环境变量：

- `DYN_NAMESPACE`：用于注册运行时端点的 Dynamo 命名空间。

可选的环境变量：

- `POD_NAMESPACE`：Global Planner 运行所在的 Kubernetes 命名空间（未设置时默认为 `default`）。

CLI 参数：

- `--managed-namespaces <ns1> <ns2> ...`：`caller_namespace` 的允许列表。如果省略则接受所有命名空间。
- `--environment kubernetes`：执行环境（目前仅支持 `kubernetes`）。
- `--no-operation`：仅记录传入的扩缩容请求并返回成功，不实际应用 Kubernetes 扩缩容。
- `--max-total-gpus <n>`：拒绝那些会让被管理池的总 GPU 数超过配置上限的扩缩容请求。

## 扩缩容请求契约

`scale_request` 端点接收 `ScaleRequest` 并返回 `ScaleResponse`。

请求字段：

- `caller_namespace`（string）：发送请求的 planner 的命名空间标识
- `graph_deployment_name`（string）：目标 `DynamoGraphDeployment` 的名称
- `k8s_namespace`（string）：目标部署所在的 Kubernetes 命名空间
- `target_replicas`（list）：期望的副本数目标
- `blocking`（bool，默认 `false`）：是否等待扩缩容完成
- `timestamp`（可选 float）：调用方提供的请求时间戳
- `predicted_load`（可选 object）：调用方提供的预测上下文

`target_replicas` 中每一项使用：

- `sub_component_type`：`prefill` 或 `decode`
- `desired_replicas`：副本数目标（整数）
- `component_name`：可选的组件名覆盖

响应字段：

- `status`：`success` 或 `error`
- `message`：状态详情
- `current_replicas`：观察到的副本数映射，例如 `{"prefill": 3, "decode": 5}`

## 行为

- 如果设置了 `--managed-namespaces` 且 `caller_namespace` 未被授权，则 Global Planner 返回 `error` 且不进行扩缩容。
- 在 `--no-operation` 模式下，Global Planner 会记录请求并返回 `success`，且 `current_replicas` 为空。

## 相关文档

- [Planner 指南](../../../../docs/components/planner/planner-guide.md) —— Planner 的配置与部署工作流
- [Global Planner 部署指南](../../../../docs/components/planner/global-planner.md) —— `GlobalPlanner` 的部署模式，含多模型协调与单端点多池的工作流
- [Planner 设计](../../../../docs/design-docs/planner-design.md) —— Planner 的架构与算法

当 planner 配置使用 `environment: "global-planner"` 并设置了 `global_planner_namespace` 时，planner 会将请求委托给该服务。
