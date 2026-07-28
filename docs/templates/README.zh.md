---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Templates
---

用于生成风格统一的 Dynamo 文档的模板集合。

## 目录层级

### 组件（Router、Planner、KVBM、Frontend、Profiler）

```
┌──────────────────────────────────────────────────────────────┐
│ Tier 1: components/src/dynamo/<component>/README.md          │ ← Redirect stub
│   Content: 1-5 lines pointing to docs/components/<component>/│
│   Template: incode_readme.md                                 │
└─────────────────────┬────────────────────────────────────────┘
                      ▼
┌──────────────────────────────────────────────────────────────┐
│ Tier 2: docs/components/<component>/                         │ ← User docs
│   • README.md ← component_readme.md                          │
│   • <component>_guide.md ← component_guide.md                │
│   • <component>_examples.md ← component_examples.md          │
└─────────────────────┬────────────────────────────────────────┘
                      ▼
┌──────────────────────────────────────────────────────────────┐
│ Tier 3: docs/design_docs/<component>_design.md               │ ← Contributor docs
│   Template: component_design.md                              │
└──────────────────────────────────────────────────────────────┘
```

### 后端（SGLang、TRT-LLM、vLLM）

```
┌─────────────────────────────────────────────────────┐
│ Tier 1: components/src/dynamo/<backend>/README.md   │ ← Redirect stub
│   Content: 1-5 lines pointing to docs/backends/     │
│   Template: incode_readme.md                        │
└─────────────────────┬───────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────┐
│ Tier 2: docs/backends/<backend>/                    │ ← User docs
│   • README.md ← backend_readme.md                   │
│   • <backend>_guide.md ← backend_guide.md           │
│                                                     │
│ Tier 2.5: docs/backends/README.md (exists)          │
│   • Backend comparison table                        │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Tier 3: External                                    │
│   Backend internals documented in upstream repos    │
└─────────────────────────────────────────────────────┘
```

### 特性（Multimodal、LoRA、Speculative Decoding）

```
┌─────────────────────────────────────────────────────┐
│ Tier 1: N/A                                         │
│   No in-code README (features are not components)   │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Tier 2: docs/features/<feature>/                    │ ← User docs
│   • README.md ← feature_readme.md                   │
│   • <feature>_sglang.md ← feature_backend.md        │
│   • <feature>_trtllm.md ← feature_backend.md        │
│   • <feature>_vllm.md ← feature_backend.md          │
└─────────────────────┬───────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────┐
│ Tier 3: docs/design_docs/<feature>_design.md        │ ← Optional
│   Only if significant architecture                  │
└─────────────────────────────────────────────────────┘
```

### 集成（LMCache、HiCache、NIXL）

```
┌─────────────────────────────────────────────────────┐
│ Tier 1: N/A                                         │
│   No in-code README (external tools)                │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Tier 2: docs/integrations/<integration>/            │ ← User docs
│   • README.md ← integration_readme.md               │
│   • <integration>_setup.md (custom)                 │
│   • <integration>_<backend>.md (custom)             │
└─────────────────────────────────────────────────────┘
```

### Deploy（Kubernetes、Helm、Operator）

```
┌─────────────────────────────────────────────────────┐
│ Tier 1: N/A                                         │
│   No in-code README (deployment topics)             │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Tier 2: docs/deploy/                                │ ← User docs
│   • README.md (deployment overview)                 │
│   • installation_guide.md, dynamo_operator.md       │
│   • helm.md, examples/                              │
└─────────────────────────────────────────────────────┘
```

### 性能（Tuning、Benchmarks）

```
┌─────────────────────────────────────────────────────┐
│ Tier 1: N/A                                         │
│   No in-code README (performance topics)            │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Tier 2: docs/performance/                           │ ← User docs
│   • README.md (performance overview)                │
│   • tuning.md, benchmarking.md, etc.                │
└─────────────────────────────────────────────────────┘
```

### 基础设施（可观测性、容错、开发）

```
┌─────────────────────────────────────────────────────┐
│ Tier 1: N/A                                         │
│   No in-code README (operations topics)             │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Tier 2: docs/infrastructure/<topic>/                │ ← User docs
│   • README.md ← infrastructure_readme.md            │
│   • <subtopic>.md (detailed guides)                 │
└─────────────────────────────────────────────────────┘
```

## 三层模式

| 层级 | 用途 | 受众 | 位置 |
|------|---------|----------|----------|
| **Tier 1** | 重定向桩（5 行） | 浏览代码的开发者 | `components/src/dynamo/`\<name>`/README.md` |
| **Tier 2** | 面向用户的文档 | 用户、运维 | `docs/`\<category>`/`\<name>`/`（例如 `docs/components/router/`） |
| **Tier 3** | 设计文档 | 贡献者 | `docs/design_docs/`\<name>`_design.md` |

## 模板选择

| 你要文档化的内容 | 应使用的模板 |
|------------------------|------------------|
| 新组件 | `incode_readme.md` + `component_*.md`（共 4 个） |
| 新后端 | `incode_readme.md` + `backend_*.md`（共 2 个） |
| 新特性 | `feature_readme.md` + `feature_backend.md`（每个 backend 一份） |
| 新集成 | `integration_readme.md` |
| 新部署主题 | 自定义（遵循 `docs/deploy/` 结构） |
| 新性能主题 | 自定义（遵循 `docs/performance/` 结构） |
| 新基础设施主题 | `infrastructure_readme.md` |
| 迁移已有文档 | 使用与目标文件匹配的模板 |

## 用法

1. 确定你的文档属于哪个分类（组件、后端、特性、集成）
2. 创建上图所示的目录结构
3. 将模板复制到对应位置并使用正确的文件名
4. 将所有 `<placeholders>` 替换为实际值
5. 将 `{/* comments */}` 替换为实际内容
6. 删除不适用的小节

## 更新导航

在添加新文档之后：

1. **Sphinx（当前）：** 更新 `docs/index.rst` 或对应的 `_sections/*.rst` 文件，将新文档纳入导航
2. **Fern（未来）：** 在 `fern/docs.yml` 中添加新页面

构建文档的指引参见 [docs/README.md](../../README.md)。
