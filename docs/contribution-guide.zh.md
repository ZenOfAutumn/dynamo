---
# SPDX-FileCopyrightText: Copyright (c) 2024-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
subtitle: How to contribute to Dynamo
max-toc-depth: 3
---

# 贡献指南

Dynamo 是一个开源的分布式推理（inference）平台，由不断壮大的贡献者社区共同打造。项目采用 [Apache 2.0](https://github.com/ai-dynamo/dynamo/blob/main/LICENSE) 协议，欢迎从修复拼写到新增重大特性的各类贡献。社区贡献已经塑造了 Dynamo 的多个核心领域，包括后端集成、文档、部署工具与性能优化。

凭借 200+ 外部贡献者、220+ 已合并的社区 PR 以及每月新增的贡献者，Dynamo 是增长最快的开源推理项目之一。可以查看我们的 [提交活动](https://github.com/ai-dynamo/dynamo/graphs/commit-activity) 与 [GitHub stars](https://github.com/ai-dynamo/dynamo/stargazers)。本指南将帮助你迈出第一步。

加入社区：

- [CNCF Slack（`#ai-dynamo`）](https://communityinviter.com/apps/cloud-native/cncf) — 加入 CNCF Slack，并在 `#ai-dynamo` 频道找到我们
- [Discord](https://discord.gg/nvidia-dynamo)
- [GitHub Discussions](https://github.com/ai-dynamo/dynamo/discussions)

## TL;DR

面向有经验的贡献者：

1. Fork 并克隆仓库
2. 对于 ≥100 行的改动或新特性，先 [开 issue](https://github.com/ai-dynamo/dynamo/issues/new?template=contribution_request.yml)
3. 创建分支：`git checkout -b yourname/fix-router-timeout`
4. 修改后运行 `pre-commit run`
5. 使用 DCO sign-off 提交：`git commit -s -m "fix: description"`
6. 向 `main` 分支提交 PR

---

## 贡献方式

### 报告 Bug

发现了问题？[提交 bug 报告](https://github.com/ai-dynamo/dynamo/issues/new?template=bug_report.yml)，请包含：

- 复现步骤
- 期望行为 vs 实际行为
- 环境信息（OS、GPU、Python 版本、Dynamo 版本）

### 改进文档

文档改进永远受欢迎：

- 修正错别字或不清晰的描述
- 增加示例或教程
- 改进 API 文档

小型文档修改可以直接提 PR，无需先开 issue。

### 提议特性

有想法？在实现前 [提交 feature request](https://github.com/ai-dynamo/dynamo/issues/new?template=feature_request.yml) 与维护者讨论。

### 贡献代码

准备写代码？请参见下方 [贡献流程](#contribution-workflow)。

### 帮助社区

并非所有贡献都是代码。你也可以：

- 在 [Discord](https://discord.gg/nvidia-dynamo) 或 [CNCF Slack](https://communityinviter.com/apps/cloud-native/cncf) 的 `#ai-dynamo` 频道回答问题
- 审阅 PR
- 分享你如何使用 Dynamo — 博客、演讲、社交媒体
- 给 [仓库](https://github.com/ai-dynamo/dynamo) 加 star

---

## 起步

### 找一个 Issue

浏览 [open issues](https://github.com/ai-dynamo/dynamo/issues) 或关注：

| Issue 类型 | 说明 |
|------------|-------------|
| [Good First Issues](https://github.com/ai-dynamo/dynamo/labels/good-first-issue) | 适合新手，并附带指引 |
| [Help Wanted](https://github.com/ai-dynamo/dynamo/labels/help-wanted) | 欢迎社区贡献 |

### Fork 并克隆

1. 在 GitHub [Fork 仓库](https://github.com/ai-dynamo/dynamo/fork)
2. 克隆你的 fork：

```bash
git clone https://github.com/YOUR-USERNAME/dynamo.git
cd dynamo
git remote add upstream https://github.com/ai-dynamo/dynamo.git
```

### 从源码构建

> [!TIP]
> 完整构建说明见下。展开折叠以搭建本地开发环境。

<details>
<summary>展开构建说明</summary>

#### 1. 安装系统库

**Ubuntu：**

```bash
sudo apt install -y build-essential libhwloc-dev libudev-dev pkg-config libclang-dev protobuf-compiler python3-dev cmake
```

**macOS：**

```bash
# Install Homebrew if needed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install cmake protobuf

# Verify Metal is accessible
xcrun -sdk macosx metal
```

#### 2. 安装 Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

#### 3. 创建 Python 虚拟环境

如未安装 [uv](https://docs.astral.sh/uv/#installation)，请先安装：

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

创建并激活虚拟环境：

```bash
uv venv .venv
source .venv/bin/activate
```

#### 4. 安装构建工具

```bash
uv pip install pip maturin
```

[Maturin](https://github.com/PyO3/maturin) 是 Rust-Python 绑定的构建工具。

#### 5. 构建 Rust 绑定

```bash
cd lib/bindings/python
maturin develop --uv
```

#### 6. 安装 GPU Memory Service

```bash
# Return to project root
cd "$(git rev-parse --show-toplevel)"
uv pip install -e lib/gpu_memory_service
```

#### 7. 安装 wheel

```bash
uv pip install -e .
```

#### 8. 验证构建

```bash
python3 -m dynamo.frontend --help
```

> [!TIP]
> VSCode 与 Cursor 用户可使用 [`.devcontainer`](https://github.com/ai-dynamo/dynamo/tree/main/.devcontainer) 目录获得预配置的开发环境。详见 [devcontainer README](https://github.com/ai-dynamo/dynamo/blob/main/.devcontainer/README.md)。

</details>

### 设置 Pre-commit 钩子

```bash
uv pip install pre-commit
pre-commit install
```

环境就绪！保持好奇心 — 探索代码、把玩 [示例](https://github.com/ai-dynamo/dynamo/tree/main/examples)、看看各部分如何配合。准备好了就从 [Good First Issues](https://github.com/ai-dynamo/dynamo/labels/good-first-issue) 中挑一个，或继续阅读完整的贡献流程。

---

## 贡献流程

贡献流程取决于改动的规模与范围。即使非必要，开 issue 也是与 Dynamo 维护者预先沟通、避免在 PR 上做无用功的好方式。

| 规模 | 修改行数 | 例子 | 所需流程 |
|------|---------------|---------|---------------|
| **XS** | 1–10 | 拼写修正、配置微调 | 直接提交 PR |
| **S** | 10–100 | 小 bug 修复、文档改进、聚焦的重构 | 直接提交 PR |
| **M** | 100–200 | 新增特性、中等重构 | 先 [开 issue](https://github.com/ai-dynamo/dynamo/issues/new?template=contribution_request.yml) |
| **L** | 200–500 | 多文件特性、新组件 | 先 [开 issue](https://github.com/ai-dynamo/dynamo/issues/new?template=contribution_request.yml) |
| **XL** | 500–1000 | 大型特性、跨组件改动 | 先 [开 issue](https://github.com/ai-dynamo/dynamo/issues/new?template=contribution_request.yml) |
| **XXL** | 1000+ | 架构改动 | 需要 [DEP](https://github.com/ai-dynamo/enhancements) |

**小改动（不到 100 行）：** 直接提 PR — 无需 issue。包括拼写、简单 bug 修复、格式化。如果 PR 解决某个已批准的 issue，可用 "Fixes #123" 关联。

**较大改动（≥100 行）：** 先 [提交 Contribution Request](https://github.com/ai-dynamo/dynamo/issues/new?template=contribution_request.yml) issue，等到打上 `approved-for-pr` 标签后再提交 PR。

**架构改动：** 影响多个组件、引入或修改公共 API、变更通信面架构、或影响后端集成契约的改动需要 [Dynamo Enhancement Proposal（DEP）](https://github.com/ai-dynamo/enhancements)。在动手实现前，请到 [`ai-dynamo/enhancements`](https://github.com/ai-dynamo/enhancements) 仓库提交 DEP。

### 提交 Pull Request

1. **创建 GitHub Issue**（如必要） — [提交 Contribution Request](https://github.com/ai-dynamo/dynamo/issues/new?template=contribution_request.yml)，描述你要解决的问题、方案、估计的 PR 体量与受影响的文件。

2. **获得批准** — 等待维护者审阅并打上 `approved-for-pr` 标签。

3. **提交 Pull Request** — [开 PR](https://github.com/ai-dynamo/dynamo/compare)，使用 GitHub 关键字关联 issue（如 "Fixes #123"）。

4. **处理 Code Rabbit 评审** — 回应 Code Rabbit 的自动化建议（包括小细节）。

5. **触发 CI 测试** — 对外部贡献者，需由维护者评论 `/ok to test COMMIT-ID` 才能运行完整 CI；其中 `COMMIT-ID` 是你最新提交的短 SHA。在请求人工评审前，请先修复任何失败的测试。

6. **请求评审** — 把批准你 issue 的人加为评审者。根据修改的文件，参考 [CODEOWNERS](https://github.com/ai-dynamo/dynamo/blob/main/CODEOWNERS) 添加必需的批准者。

> [!IMPORTANT]
> **AI 生成的代码：** 我们鼓励使用 AI 工具，但你必须完全理解 PR 中的每一处改动。无法解释所提交代码的将被拒绝。

### 分支命名

使用能识别你与改动的描述性分支名：

```text
yourname/fix-description
```

例：

```text
jsmith/fix-router-timeout
jsmith/add-lora-support
```

---

## 代码风格与质量

维护者基于代码风格、测试覆盖、架构契合度与评审响应度来评估贡献质量。一致、高质量的贡献是建立项目信任的基础。

### Pre-commit 钩子

所有 PR 都会经 [pre-commit 钩子](https://github.com/ai-dynamo/dynamo/blob/main/.pre-commit-config.yaml) 检查。[安装 pre-commit](#set-up-pre-commit-hooks) 后可在本地运行：

```bash
pre-commit run --all-files
```

### 提交信息规范

使用 [conventional commit](https://www.conventionalcommits.org/) 前缀：

| 前缀 | 适用场景 |
|--------|---------|
| `feat:` | 新特性 |
| `fix:` | bug 修复 |
| `docs:` | 文档改动 |
| `refactor:` | 不改变行为的重构 |
| `test:` | 增加或更新测试 |
| `chore:` | 维护、依赖更新 |
| `ci:` | CI/CD 改动 |
| `perf:` | 性能改进 |

例：

```text
feat(router): add weighted load balancing
fix(frontend): resolve streaming timeout on large responses
docs: update quickstart for macOS users
test(planner): add unit tests for scaling policy
```

### 语言约定

| 语言 | 风格指南 | 格式化器 |
|----------|-------------|-----------|
| **Python** | [PEP 8](https://peps.python.org/pep-0008/) | `black`、`ruff` |
| **Rust** | [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) | `cargo fmt`、`cargo clippy` |
| **Go** | [Effective Go](https://go.dev/doc/effective_go) | `gofmt` |

### 测试

提交 PR 前请运行测试套件：

```bash
# Run all tests
pytest tests/

# Run unit tests only
pytest -m unit tests/

# Run a specific test file
pytest -s -v tests/test_example.py
```

Rust 组件：

```bash
cargo test
```

Kubernetes operator（Go）：

```bash
cd deploy/operator
go test ./... -v
```

### 通用准则

- 保持 PR 聚焦 — 一个 PR 解决一件事
- 编写清晰且文档完善的代码，便于后来者理解
- 为新功能与 bug 修复添加测试
- 确保构建干净（无警告或错误）
- 所有测试都必须通过
- 不要保留被注释掉的代码
- 及时、建设性地回应评审反馈

### 在本地运行 GitHub Actions

使用 [act](https://nektosact.com/) 在本地运行工作流：

```bash
act -j pre-merge-rust
```

或使用 VS Code 扩展 [GitHub Local Actions](https://marketplace.visualstudio.com/items?itemName=SanjulaGanepola.github-local-actions)。

---

## 流程预期

### 状态标签

| 状态 | 含义 |
|--------|---------------|
| `needs-triage` | 我们正在审阅你的 issue |
| `needs-info` | 需要你提供更多信息 |
| `approved-for-pr` | 可以开始实现了 — 请提交 PR |
| `in-progress` | 已有人在处理 |
| `blocked` | 等待外部依赖 |

### 响应时长

我们的目标：

- 在数个工作日内对新 issue **作出响应**
- 一周内对高优先级 issue **完成分流（triage）**

30 天无活动的 issue 可能会被自动关闭（可重开）。

### 评审流程

提交 PR 并完成 [提交 Pull Request](#submitting-a-pull-request) 中的步骤后：

1. 评审者将给出反馈 — 请在合理时间内回应所有评论
2. 若有变更要求，请在处理后 ping 评审者重新审阅
3. 7 天内未被评审，可自由 ping 评审者或留言

### Good First Issues

被打上 `good-first-issue` 的 issue 体量适合新人。我们对其提供更多指引 — 留意 issue 描述中清晰的验收标准与建议方案。

---

## DCO 与许可

### Developer Certificate of Origin

Dynamo 要求所有贡献都附带 [Developer Certificate of Origin（DCO）](https://developercertificate.org/) 的 sign-off。它表明你有权在本项目的 [Apache 2.0 协议](https://github.com/ai-dynamo/dynamo/blob/main/LICENSE) 下提交贡献。

每次提交都必须包含 sign-off 行：

```text
Signed-off-by: Jane Smith &lt;jane.smith@email.com&gt;
```

通过 `-s` 标志自动添加：

```bash
git commit -s -m "fix: your descriptive message"
```

**要求：**

- 使用真实姓名（不接受化名或匿名贡献）
- git 中必须配置好 `user.name` 与 `user.email`

**DCO 检查失败？** 参见 [DCO 故障排查指南](https://github.com/ai-dynamo/dynamo/blob/main/DCO.md) 获得逐步修复步骤。

### 许可

通过贡献，即表示你同意你的贡献按 [Apache 2.0 License](https://github.com/ai-dynamo/dynamo/blob/main/LICENSE) 授权。

---

## 行为守则

我们致力于打造欢迎、包容的环境。所有参与者都应遵守 [行为守则](https://github.com/ai-dynamo/dynamo/blob/main/CODE_OF_CONDUCT.md)。

---

## 安全

如果发现安全漏洞，请按 [Security Policy](https://github.com/ai-dynamo/dynamo/blob/main/SECURITY.md) 操作。请勿为安全漏洞开公共 issue。

---

## 获取帮助

- **CNCF Slack**：[加入 CNCF Slack](https://communityinviter.com/apps/cloud-native/cncf)，在 `#ai-dynamo` 找到我们
- **Discord**：[加入社区](https://discord.gg/nvidia-dynamo)
- **Discussions**：[GitHub Discussions](https://github.com/ai-dynamo/dynamo/discussions)
- **文档**：[docs.nvidia.com/dynamo](https://docs.nvidia.com/dynamo/)

感谢为 Dynamo 贡献！
