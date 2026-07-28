---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
sidebar-title: Dynamo Docs Guide
---

# Dynamo 文档的构建与发布方式

本文档介绍由 [Fern](https://buildwithfern.com) 驱动的 NVIDIA Dynamo 文档站点的架构、工作流与维护流程。

<Note>
文档站点完全托管在
[Fern](https://buildwithfern.com) 上。CI 将文档发布到
`dynamo.docs.buildwithfern.com`；正式域名
`docs.dynamo.nvidia.com` 是指向 Fern 站点的自定义域名别名。
没有独立服务器 —— 托管、CDN 与基于版本的 URL 路由都由 Fern 负责。
</Note>

<Error>
`docs-website` 分支由 **CI 管理，禁止人工编辑**。所有文档创作都发生在 `main`（或基于 `main` 的特性分支）上。同步工作流会自动将变更复制到 `docs-website`。
</Error>

---

## 目录

- [分支架构](#分支架构)
- [目录结构](#目录结构)
- [配置文件](#配置文件)
- [GitHub 工作流](#github-工作流)
  - [Fern Docs 工作流](#fern-docs-工作流-fern-docsyml)
  - [Docs Link Check 工作流](#docs-link-check-工作流-docs-link-checkyml)
- [内容创作](#内容创作)
- [Callout 转换](#callout-转换)
- [本地运行](#本地运行)
- [版本管理](#版本管理)
- [发布工作机制](#发布工作机制)
- [常见任务](#常见任务)
- [Claude Code Skills](#claude-code-skills)

---

## Claude Code Skills

一个 Claude Code 技能脚本可自动化常见文档任务。在 Claude Code 中以斜杠命令形式调用（如 `/dynamo-docs`）—— 该 skill 会引导你走完完整流程：创建、编辑或移除 markdown 文件，更新 `docs/index.yml` 中的导航，并运行 `fern check` 进行校验。

| Skill | 描述 |
|-------|-------------|
| [dynamo-docs](https://github.com/ai-dynamo/dynamo/blob/main/.agents/skills/dynamo-docs/SKILL.md) | 添加、更新、移动或移除文档页面 |

---

## 分支架构

文档系统采用**双分支模型**：

| 分支 | 用途 | 内容 | Fern 配置 |
|---|---|---|---|
| `main` | **dev**（未发布）文档的事实来源 | `docs/` | `fern/` |
| `docs-website` | 已发布文档，包含**所有版本快照** | `fern/pages/` | `fern/` |

作者在 `main` 上编辑页面。一个 GitHub Actions 工作流会自动将变更同步到 `docs-website` 分支并发布到 Fern。`docs-website` 分支不接受人工编辑 —— 完全由 CI 管理。

### 为什么分两个分支？

`docs-website` 分支会随时间累积版本快照（如 `pages-v0.8.0/`、`pages-v0.8.1/`）。将这些放在独立分支上，避免让 `main` 分支被旧版文档的冻结副本撑大。

---

## 目录结构

### 在 `main` 上

```text
fern/                             # Fern CLI 配置（fern/ 是 Fern 约定）
├── fern.config.json              # Fern 组织 + CLI 版本 pin
├── docs.yml                      # 站点配置（实例、品牌、布局）
├── components/
│   └── CustomFooter.tsx          # 站点 footer 的 React 组件
├── main.css                      # 自定义 CSS（NVIDIA 品牌、暗色模式等）
├── convert_callouts.py           # GitHub → Fern admonition 转换脚本
└── .gitignore                    # Fern 专属忽略规则

docs/                             # 文档内容
├── index.yml                     # dev 版本的导航树
├── getting-started/              # Markdown 内容（实际文档）
├── kubernetes/
├── reference/
├── ...
├── assets/                       # 图片、字体、SVG、logo
├── digest/                       # Digest 帖
└── diagrams/                     # D2 图源文件
```

### 在 `docs-website` 上

`docs-website` 分支采用面向 Fern 目录约定优化的不同布局，并包含版本快照：

```text
fern/
├── fern.config.json              # Fern 组织 + CLI 版本 pin
├── docs.yml                      # 包含完整 versions 数组
├── versions/
│   ├── dev.yml                   # “Next”/dev 导航（从 main 同步）
│   ├── v0.8.1.yml                # v0.8.1 快照导航
│   └── v0.8.0.yml                # v0.8.0 快照导航
├── pages/                        # 当前 dev 内容（从 main 同步）
├── pages-v0.8.1/                 # v0.8.1 时 pages/ 的冻结快照
├── pages-v0.8.0/                 # v0.8.0 时 pages/ 的冻结快照
├── components/                   # React 组件
├── main.css                      # 自定义 CSS
├── convert_callouts.py           # Callout 转换脚本
├── digest/                       # Digest 帖（从 main 同步）
└── assets/                       # 图片、字体、SVG
```

每个 `pages-vX.Y.Z/` 目录都是发布时 `pages/` 的不可变副本。对应的 `versions/vX.Y.Z.yml` 是 `dev.yml` 的副本，所有 `../pages/` 路径会被改写为 `../pages-vX.Y.Z/`。

同步工作流会把 `main` 的 `docs/` 复制到 `fern/pages/`，并相应地把 `index.yml` 的导航路径转换为 `versions/dev.yml`。

---

## 配置文件

### `fern/fern.config.json`

```json
{
  "organization": "nvidia",
  "version": "3.73.0"
}
```

- **organization**：拥有该文档站点的 Fern 组织。
- **version**：固定使用的 Fern CLI 版本。

### `fern/docs.yml`

这是 Fern 站点的主配置。关键段：

| 段 | 用途 |
|---|---|
| `instances` | 部署目标 —— staging URL 与自定义生产域名 |
| `products` | 定义产品（“Dynamo”）及其版本列表 |
| `navbar-links` | 导航栏中的 GitHub 仓库链接 |
| `footer` | 指向 `CustomFooter.tsx` React 组件 |
| `layout` | 页面宽度、侧栏宽度、搜索框位置等 |
| `colors` | NVIDIA 绿（`#76B900`）强调色，黑/白背景 |
| `typography` | NVIDIA Sans 正文字体，Roboto Mono 代码字体 |
| `logo` | NVIDIA logo（深/浅变体），高度 20px |
| `js` | Adobe Analytics 脚本注入 |
| `css` | 自定义 `main.css` 样式表 |

**重要：** 在 `main` 上，`docs.yml` 仅列出 `dev` 版本。在 `docs-website` 上，则包含**完整的 versions 数组**（dev + 所有 release）。同步工作流在从 `main` 复制 `docs.yml` 时，会保留来自 `docs-website` 的 versions 数组。

### `docs/index.yml`

定义导航树 —— 文档站点的侧栏结构。每个条目把页面标题映射到 markdown 文件路径：

```yaml
navigation:
  - section: Getting Started
    contents:
      - page: Quickstart
        path: getting-started/quickstart.mdxx
      - page: Support Matrix
        path: reference/support-matrix.md
```

路径相对 `docs/` 目录。section 可以嵌套。页面可标记 `hidden: true`，使其可通过 URL 访问但在侧栏不显示。

同步到 `docs-website` 时，工作流会把 `index.yml` 复制为 `fern/versions/dev.yml`，并把路径转换（如 `getting-started/X` → `../pages/getting-started/X`）以匹配 docs-website 的目录布局。

---

## GitHub 工作流

### Fern Docs 工作流（`fern-docs.yml`）

**位置：** `.github/workflows/fern-docs.yml`

这一个合并后的工作流统一处理 lint、同步、版本化与发布。它根据触发条件运行三种 job：

#### Job 1：Lint（PR）

**触发：** 修改了 `docs/**` 文件的 Pull Request。

**步骤：**
1. `fern check` —— 校验 Fern 配置语法
2. `fern docs broken-links` —— 检查内部链接是否失效

**目的：** 在合并前捕获文档错误。

#### Job 2：同步 dev（推送到 `main`）

**触发：** 修改了 `docs/**` 的 push 到 `main`，或手动 `workflow_dispatch`（不指定 tag）。

**步骤：**
1. 并排检出 `main` 与 `docs-website` 分支
2. 把 `main` 的 `docs/` 复制到 `docs-website` 的 `fern/pages/`
3. 用 `yq` 把 `docs/index.yml` 复制为 `fern/versions/dev.yml` 并按 docs-website 布局转换路径
4. 同步 `docs/assets/` 中的资源与 `docs/digest/` 中的 Digest 帖
5. 把 `fern/` 中的 Fern 配置文件复制到 docs-website 的 `fern/`
   （`fern.config.json`、`components/`、`main.css`、`convert_callouts.py`）
6. 运行 `convert_callouts.py`，将 GitHub 风格 callout 转换为 Fern 格式
7. 用 `main` 的内容更新 `docs.yml`，**同时保留 `docs-website` 中的 versions 数组**（用 `yq` 保存/恢复 versions 列表）
8. commit 并推送到 `docs-website`
9. 通过 `fern generate --docs` 发布到 Fern

#### Job 3：版本发布（tag）

**触发：** 形如 `vX.Y.Z` 的新 Git tag（如 `v0.9.0`、`v1.0.0`），或手动 `workflow_dispatch` 并指定 tag。

**步骤：**
1. 校验 tag 格式（必须是 `vX.Y.Z`，无 `-rc1` 等后缀）
2. 检查该版本不存在（避免重复快照）
3. 通过复制 `fern/pages/` 创建 `fern/pages-vX.Y.Z/`
4. 改写快照中的 GitHub 链接：
   - `github.com/ai-dynamo/dynamo/tree/main` → `tree/vX.Y.Z`
   - `github.com/ai-dynamo/dynamo/blob/main` → `blob/vX.Y.Z`
5. 在快照上运行 `convert_callouts.py`
6. 从 `dev.yml` 创建 `fern/versions/vX.Y.Z.yml`，并将路径更新为 `../pages-vX.Y.Z/`
7. 更新 `fern/docs.yml`：
   - 在 “dev” 条目之后插入新版本
   - 将产品默认 `path` 设为新版本
   - 更新 “Latest” 显示名为 `"Latest (vX.Y.Z)"`
8. commit 并推送到 `docs-website`
9. 通过 `fern generate --docs` 发布到 Fern

**防递归说明：** 用 `GITHUB_TOKEN` 进行的 push 不会触发其它工作流（GitHub 的内置防护）。这就是为什么发布步骤直接内嵌在每个 job 中，而不是放到独立的工作流里。

### Docs Link Check 工作流（`docs-link-check.yml`）

**位置：** `.github/workflows/docs-link-check.yml`

**触发：** 推送到 `main` 与 PR。

会运行两个独立的链接检查 job：

| Job | 工具 | 检查内容 |
|---|---|---|
| `lychee` | [Lychee](https://lychee.cli.rs/) | 外部 HTTP 链接（带缓存、重试、限流处理）。PR 上以离线模式运行。|
| `broken-links-check` | 自定义 Python 脚本（`detect_broken_links.py`）| 内部相对 markdown 链接与符号链接。在 PR 上创建 GitHub annotation，精确指向断链所在行。|

---

## 内容创作

### 在 `main` 上撰写文档

1. 在 `docs/` 中编辑或新增 markdown 文件。
2. 如果新增页面，请在 `docs/index.yml` 中添加条目，使其出现在侧栏导航中。
3. 使用标准 GitHub 风格 markdown。Callout（admonition）应使用 GitHub 原生语法 —— 同步过程中会被自动转换：
   ```markdown
   > [!NOTE]
   > 这条 note 会被转换为 Fern 的 `<Note>` 组件。

   > [!WARNING]
   > 这条 warning 会被转换为 Fern 的 `<Warning>` 组件。
   ```
4. 提 PR。lint job（`fern check`、`fern docs broken-links`、lychee、broken-links-check）会自动运行。
5. 一旦合入 `main`，sync-dev 工作流会在数分钟内发布变更。

### 资源与图片

把图片放在 `docs/assets/`，在 markdown 中以相对路径引用：

```markdown
![Architecture Diagram](../assets/img/dynamo-architecture.svg)
```

### 自定义组件

`fern/components/` 中的 React 组件可在 markdown 中通过 MDX 使用。`CustomFooter.tsx` 渲染包含合规链接与品牌的 NVIDIA footer。

---

## Callout 转换

`fern/convert_callouts.py` 脚本承接 GitHub 风格 markdown 与 Fern admonition 格式之间的转换。这让作者在 `main` 上使用 GitHub 原生 callout 语法，同时让 Fern 得到它要求的组件格式。

### 映射

| GitHub 语法 | Fern 组件 |
|---|---|
| `> [!NOTE]` | `<Note>` |
| `> [!TIP]` | `<Tip>` |
| `> [!IMPORTANT]` | `<Info>` |
| `> [!WARNING]` | `<Warning>` |
| `> [!CAUTION]` | `<Error>` |

### 用法

```bash
# 递归转换目录下所有文件（就地修改）
python fern/convert_callouts.py --dir docs/

# 转换单个文件
python fern/convert_callouts.py input.md output.md

# 运行内置测试
python fern/convert_callouts.py --test
```

转换会在 sync-dev 与 release-version 工作流中自动发生。作者无需手动运行。

---

## 本地运行

可以使用 [Fern CLI](https://buildwithfern.com/learn/cli-api/overview) 在本机预览文档站点。这有助于在提 PR 前核对布局、导航与内容。

### 前置条件

通过 npm 全局安装 Fern CLI：

```bash
npm install -g fern-api
```

### 校验配置

在仓库根目录运行 `fern check`，校验 `fern/docs.yml`、`fern/fern.config.json` 与导航文件的语法是否正确：

```bash
fern check
```

### 检查断链

使用 `fern docs broken-links` 扫描所有页面，找出内部链接中无法解析的条目：

```bash
fern docs broken-links
```

CI 在每次 PR 上运行的就是这个检查。

### 启动本地预览服务器

`fern docs dev` 会构建站点并以热重载方式在本地提供：

```bash
fern docs dev
```

本地服务器可让你看到与线上完全一致的页面渲染，包括导航、版本下拉与自定义样式。

---

## 版本管理

### 版本机制

Fern 站点支持 UI 中的版本下拉菜单。每个版本由以下三部分定义：

1. **导航文件**（`fern/versions/vX.Y.Z.yml`）—— 侧栏结构，指向版本专属页面（位于 `docs-website` 分支）。
2. **页面目录**（`fern/pages-vX.Y.Z/`）—— 发布时 markdown 内容的冻结快照（位于 `docs-website` 分支）。
3. **`fern/docs.yml` 中的条目** —— 告诉 Fern 该版本的展示名、slug 与配置路径。

### 版本类型

| 版本 | 显示名 | Slug | 描述 |
|---|---|---|---|
| Latest | `Latest (vX.Y.Z)` | `/` | 默认版本；指向最新发布 |
| 稳定版 | `vX.Y.Z` | `vX.Y.Z` | 不可变快照 |
| Dev | `dev` | `dev` | 跟随 `main`；每次 push 更新 |

### URL 结构

- **Latest（默认）：** `docs.dynamo.nvidia.com/dynamo/`
- **特定版本：** `docs.dynamo.nvidia.com/dynamo/v0.8.1/`
- **Dev：** `docs.dynamo.nvidia.com/dynamo/dev/`

### 创建新版本

只需推送一个 semver tag：

```bash
git tag v0.9.0
git push origin v0.9.0
```

`fern-docs.yml` 中的 `release-version` job 会自动处理其余事项。

---

## 发布工作机制

```text
┌─────────────────────────────────────────────────────────────────────┐
│                        CONTINUOUS (dev)                             │
│                                                                     │
│  Developer pushes to main                                           │
│       │                                                             │
│       ▼                                                             │
│  docs/** changed? ── No ──▶ (nothing happens)                      │
│       │                                                             │
│      Yes                                                            │
│       │                                                             │
│       ▼                                                             │
│  sync-dev job:                                                      │
│    1. Copy docs/ content → fern/pages/ on docs-website branch │
│    2. Copy fern/ configs → fern/ on docs-website branch             │
│    3. Convert GitHub callouts → Fern admonitions                    │
│    4. Preserve version list from docs-website's docs.yml            │
│    5. Commit + push to docs-website                                 │
│    6. fern generate --docs (publishes to Fern)                      │
│       │                                                             │
│       ▼                                                             │
│  Live on docs.dynamo.nvidia.com/dynamo/dev/ within minutes          │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      VERSION RELEASE                                │
│                                                                     │
│  Maintainer pushes vX.Y.Z tag                                       │
│       │                                                             │
│       ▼                                                             │
│  release-version job:                                               │
│    1. Validate tag format (vX.Y.Z only)                             │
│    2. Check version doesn't already exist                           │
│    3. Snapshot fern/pages/ → fern/pages-vX.Y.Z/                     │
│    4. Rewrite GitHub links (tree/main → tree/vX.Y.Z)               │
│    5. Convert callouts in snapshot                                  │
│    6. Create fern/versions/vX.Y.Z.yml (paths → pages-vX.Y.Z/)     │
│    7. Update fern/docs.yml (insert version, set as default)         │
│    8. Commit + push to docs-website                                 │
│    9. fern generate --docs (publishes to Fern)                      │
│       │                                                             │
│       ▼                                                             │
│  New version visible in dropdown at docs.dynamo.nvidia.com/dynamo/  │
└─────────────────────────────────────────────────────────────────────┘
```

### Secrets

| Secret | 用途 |
|---|---|
| `FERN_TOKEN` | `fern generate --docs` 的认证 token。发布所必需。存储在 GitHub 仓库 secrets 中。|

---

## 常见任务

### 更新已有文档

1. 在特性分支上编辑 `docs/` 中的文件。
2. 如果新增页面，在 `docs/index.yml` 中添加对应条目。
3. 提 PR —— lint 自动运行。
4. 合并 —— 同步 + 发布自动完成。

### 新增顶层 section

1. 在 `docs/` 下创建目录（如 `docs/new-section/`）。
2. 为每个页面添加 markdown 文件。
3. 在 `docs/index.yml` 中加入新的 `- section:` 块，按所需层级。

### 发布版本化文档

```bash
git tag v1.0.0
git push origin v1.0.0
```

仅此而已。工作流会对当前 dev 文档拍快照、创建版本配置并发布。

### 手动触发同步或发布

进入 **Actions → Fern Docs → Run workflow**：
- **tag** 留空触发 dev 同步。
- 输入一个 tag（如 `v0.9.0`）触发版本发布。

### 排查发布失败

1. 在 **Actions** 标签页查看失败的 `Fern Docs` 工作流运行。
2. 常见问题：
   - **断链：** 修复 `fern docs broken-links` 报告的链接。
   - **YAML 无效：** 检查 `fern/docs.yml` 或 `docs/index.yml` 语法。
   - **`FERN_TOKEN` 过期：** 在仓库 secrets 中轮换 token。
   - **重复版本：** tag 已发布过；查看 `docs-website` 是否已存在 `fern/pages-vX.Y.Z/` 目录。
