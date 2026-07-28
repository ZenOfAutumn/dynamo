---
name: dynamo-docs
description: Maintain Dynamo Fern docs site -- add, update, move, or remove pages. Use for any documentation changes.
---

# Dynamo 文档维护

用于在 Dynamo Fern 文档站点上新增、更新、迁移与删除页面的统一 skill。

## 分支规则

**所有改动都发生在 `main`（或基于 `main` 的特性分支）上。**
`docs-website` 分支由 CI 管理，**绝不能**手动编辑。

## 操作

### 新增页面

1. 收集信息：页面标题、目标章节、文件名（kebab-case 的 `.md`）、`docs/` 下的子目录。
2. 创建 `docs/<subdirectory>/<filename>.md`：

```markdown
---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: <Page Title>
---

# <Page Title>
```

3. 在 `docs/index.yml` 中正确的章节下添加导航条目：

```yaml
- page: <Page Title>
  path: <subdirectory>/<filename>.md
```

### 更新页面

1. 通过文件路径、页面标题或关键字搜索（在 `docs/` 中执行 `grep -rn`）定位页面。
2. **仅修改内容** —— 直接编辑 markdown 文件。
3. **修改标题** —— 同时更新 frontmatter 中的 `title:` 与 `docs/index.yml` 中的 `- page:` 名称。
4. **跨章节移动** —— 使用 `git mv` 移动文件，删除旧的导航条目并新增条目，更新所有指向它的链接。

### 删除页面

1. 查找指向该页的链接：`grep -r "<filename>" docs/ --include="*.md"`。
2. 执行 `git rm docs/<subdirectory>/<filename>.md`。
3. 从 `docs/index.yml` 中移除对应的 `- page:` 块。如果该章节只剩这一页，则一并移除整个 `- section:` 块。
4. 修复或移除步骤 1 中找到的所有指向链接。

---

## 内容指南

使用 GitHub 风格的 Markdown。CI 会自动将 callout 转换为 Fern 格式：

| GitHub 语法 | Fern 组件 |
|---|---|
| `> [!NOTE]` | `<Note>` |
| `> [!TIP]` | `<Tip>` |
| `> [!IMPORTANT]` | `<Info>` |
| `> [!WARNING]` | `<Warning>` |
| `> [!CAUTION]` | `<Error>` |

引用图片请使用 `docs/assets/`。

## `index.yml` 中的章节横幅

请搜索：
- `# ==================== Getting Started ====================`
- `# ==================== Kubernetes Deployment ====================`
- `# ==================== User Guides ====================`
- `# ==================== Backends ====================`
- `# ==================== Components ====================`
- `# ==================== Integrations ====================`
- `# ==================== Documentation ====================`
- `# ==================== Design Docs ====================`
- `# ==================== Blog ====================`
- `# ==================== Hidden Pages ====================`

## 校验

```bash
fern check
fern docs broken-links
```

可选的本地预览：`fern docs dev`（localhost:3000，热加载，无需 token）。

## 提交

```bash
git add docs/
git commit -s -m "docs: <add|update|remove> <page-title>"
```

## 调试

| 现象 | 解决方法 |
|---|---|
| `fern check` 报 YAML 错误 | 检查是否使用 2 空格缩进；`- page:` 必须位于 `contents:` 内部 |
| 缺失/孤儿文件 | `index.yml` 中的 `path:` 必须与实际文件位置一致 |
| CI 报告坏链接 | 执行 `grep -r "<filename>" docs/` 修复过时的引用 |
| MDX 解析错误 | 将 `<https://...>` 替换为 `[text](https://...)` |
| 站点上看不到该页 | 确保 `index.yml` 中存在导航条目；同步可能需要数分钟 |

## 关键参考

| 文件 | 用途 |
|---|---|
| `docs/index.yml` | 导航树 |
| `docs/` | 内容目录 |
| `docs/assets/` | 图片、SVG、字体 |
| `fern/docs.yml` | Fern 站点配置 |
| `fern/convert_callouts.py` | callout 转换（GitHub -> Fern） |
| `docs/README.md` | 完整架构指南 |
