<!-- # SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License. -->

# 基准测试（Benchmarks）

本目录包含用于 Dynamo 部署的基准测试工具与脚本。基准测试直接使用 [AIPerf](https://github.com/ai-dynamo/aiperf) —— 一个用于度量生成式 AI 推理（inference）性能的全面工具。

## 快速开始

### 对一个 Dynamo 部署进行基准测试
首先按照[部署文档](../docs/kubernetes/)部署你的 DynamoGraphDeployment，然后：

```bash
# 将你的部署端口转发到 http://localhost:8000
kubectl port-forward -n <namespace> svc/<frontend-service-name> 8000:8000 > /dev/null 2>&1 &

# 运行一次基准测试
aiperf profile \
    --model <your-model> \
    --url http://localhost:8000 \
    --endpoint-type chat \
    --streaming \
    --concurrency 10 \
    --request-count 100

# 进行并发扫描以便做 Pareto 分析
for c in 1 2 5 10 50 100; do
    aiperf profile \
        --model <your-model> \
        --url http://localhost:8000 \
        --endpoint-type chat \
        --streaming \
        --concurrency $c \
        --request-count $(( c * 3 > 10 ? c * 3 : 10 )) \
        --artifact-dir "artifacts/my-benchmark/c$c"
done

# 生成对比图
aiperf plot artifacts/my-benchmark
```

## 目录内容

- **`incluster/`** —— 用于在集群（cluster）内部运行基准测试的 Kubernetes Job 清单
- **`router/`** —— KV 路由（router）基准测试脚本（前缀比例、trace 回放、agent、优先级队列）
- **`prefix_data_generator/`** —— 用于分析与合成具有前缀结构数据的工具

## 完整指南

如需了解包括服务端基准测试、Pareto 分析以及 AIPerf 高级特性在内的详细文档，请参阅[完整基准测试指南](../docs/benchmarks/benchmarking.md)。
