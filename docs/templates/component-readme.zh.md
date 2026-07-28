---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Component README
---

{/* 该组件的 2-3 句概述以及其在 Dynamo 中的角色 */}

## 功能矩阵

| 特性 | 状态 |
|---------|--------|
| 特性 1 | ✅ 支持 |
| 特性 2 | 🚧 实验性 |
| 特性 3 | ❌ 不支持 |

## 快速开始

### 前置条件

- {/* 列出前置条件 */}

### 用法

```bash
# Add minimal usage example from existing docs
# Example pattern (from Router):
# python -m dynamo.frontend --router-mode kv --http-port 8000
```

### Kubernetes

```yaml
# Add DGDR example - use apiVersion: nvidia.com/v1beta1
# Example pattern (from Router):
# apiVersion: nvidia.com/v1beta1
# kind: DynamoGraphDeployment
# metadata:
#   name: <component>-deployment
# spec:
#   services:
#     ...
```

{/* 示例：Router 的 Quick Start 填好后看起来像：

### 前置条件

- 已安装 Dynamo 平台
- 至少有一个后端 worker 在运行

### 用法

```bash
python -m dynamo.frontend --router-mode kv --http-port 8000
```

### Kubernetes

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: router-example
spec:
  graphs:
    - name: frontend
      replicas: 1
``` */}

## 配置

| 参数 | 默认值 | 描述 |
|-----------|---------|-------------|
| {/* param */} | {/* default */} | {/* description */} |

## 后续步骤

| 文档 | 路径 | 描述 |
|----------|------|-------------|
| `<Component> Guide` | `<component>_guide.md` | 部署与配置 |
| `<Component> Examples` | `<component>_examples.md` | 用法示例 |
| `<Component> Design` | `/docs/design_docs/`\<component>`_design.md` | 架构 |

{/* 将表格行转为 Markdown 链接 */}
