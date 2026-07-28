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

# Dynamo Kubernetes Helm Charts

下面这个 Helm chart 适用于 Dynamo Kubernetes 平台：

- [platform](./charts/platform/README.md) - 该 chart 用于安装完整的 Dynamo Kubernetes 平台，包含 Dynamo Operator、NATS、etcd、Grove 与 Kai Scheduler。

## CRD 管理

CRD 已经打包在 operator 子 chart 的 `crds/` 目录下，并且会被自动管理：

- **首次安装**：在 `helm install` 期间，Helm 原生地从 `crds/` 目录安装 CRD。
- **升级**：`pre-upgrade` 钩子 Job 会基于 operator 镜像，使用 server-side apply 应用 CRD。这是必要的，因为在 `helm upgrade` 时 Helm 不会从 `crds/` 目录更新 CRD。可通过设置 `upgradeCRD: false` 关闭此行为。
