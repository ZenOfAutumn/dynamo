<!--
SPDX-FileCopyrightText: Copyright (c) 2024-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Planner

面向 Dynamo 推理图的 SLA 驱动弹性伸缩控制器。

## 伸缩模式

SLA Planner 支持两种伸缩模式，可以独立使用，也可以组合使用：

### 基于吞吐（Throughput-Based）的伸缩

使用部署前的 profiling 数据 + 流量预测，计算为达成 TTFT 与 ITL SLA 目标所需的 prefill/decode 副本数。需要由 Dynamo profiler 产出的 profiling 数据。

### 基于负载（Load-Based）的伸缩

使用来自 Dynamo event plane 的 ForwardPassMetrics（FPM），通过在线线性回归做出 SLA 感知的伸缩决策。**不需要** profiling 数据，也**不需要** KV Router。对突发流量响应迅速。当前**仅支持 vLLM**（FPM 仅在 vllm 中可用）。

当两种模式同时启用时：基于吞吐的伸缩为副本数提供下界（lower bound），而基于负载的伸缩负责实时调整。

### 支持矩阵

| 部署类型        | 基于吞吐         | 基于负载                   |
|-----------------|:----------------:|:-------------------------:|
| Disaggregated   | 支持             | 支持                      |
| Aggregated      | 不支持           | 支持                      |

## 文档

- **用户文档**：[Planner Guide](../../../../docs/components/planner/planner-guide.md)（部署、配置、示例）
- **设计文档**：[Planner Design](../../../../docs/design-docs/planner-design.md)（架构、算法）
- **手动操作流程**：[tests/manual/README.md](tests/manual/README.md)（dry run 辅助、性能配置、手动伸缩脚本）

