---
name: dep-create
description: Create or update Dynamo Enhancement Proposals as GitHub issues, including lightweight DEPs, implementation plans, and retroactive DEPs for ai-dynamo/dynamo.
---

# Skill：将 DEP 创建为 GitHub Issue

## 目的

在 `ai-dynamo/dynamo` 上将一份新的 Dynamo Enhancement Proposal（DEP）创建为 GitHub Issue。该 issue 编号会成为 DEP 编号。也支持为现有工作添加实现计划与追溯式 DEP。

## 何时使用

当用户希望通过基于 issue 的 DEP 工作流提议一个新特性、架构变更或流程改进时使用。也用于为现有 DEP 添加实现计划，或为已合并的工作补充追溯式 DEP。

## 工作流

### 创建一个新的 DEP

1. **询问源资料**：提示用户提供包含背景、客户上下文或详细需求的 Google Doc、Confluence 页面或其他 NVIDIA 内部文档。使用合适的工具（gdocs、Confluence MCP、WebFetch）阅读它。在 issue 的 References 部分附上链接——该文档仅 NVIDIA 员工可访问，作为不能出现在公开 issue 中的客户特定上下文记录。

2. **从用户和源文档收集必填字段**（缺失则提问）：
   - **Summary**：对提案的一段话描述
   - **Motivation**：为什么需要这个变更
   - **Proposal**：对所提议变更的详细描述

3. **基于提案内容确定 area 标签**。Area 标签是裸名（例如 `frontend`、`router`、`backend-vllm`），与 CODEOWNERS 团队对应。

4. **决定模板**：完整版还是轻量版。
   如果只需要 Summary、Motivation 和 Proposal，使用轻量版。

5. **创建 issue**（完整 DEP）：

```bash
gh issue create \
  --repo ai-dynamo/dynamo \
  --title "DEP: <short descriptive title>" \
  --label "dep:draft" \
  --label "<area>" \
  --body "$(cat <<'EOF'
## Summary
<summary>

## Motivation
<motivation>

## Proposal
<proposal>

## Alternate Solutions
<alternates>

## Requirements
<requirements>

## References
<references — include internal doc link here>
EOF
)"
```

   **轻量版 DEP** 使用：

```bash
gh issue create \
  --repo ai-dynamo/dynamo \
  --title "DEP (light): <short descriptive title>" \
  --label "dep:draft" \
  --label "dep:lightweight" \
  --label "<area>" \
  --body "$(cat <<'EOF'
## Summary
<summary>

## Motivation
<motivation>

## Proposal
<proposal>
EOF
)"
```

6. **向用户报告**已创建的 issue 编号和 URL。

### 添加实现计划

1. **阅读 DEP issue** 及其讨论：

```bash
gh issue view <number> --repo ai-dynamo/dynamo
gh issue view <number> --repo ai-dynamo/dynamo --comments
```

2. **起草计划**，包含阶段、任务、工作量估计、依赖、风险和测试策略。

3. **作为评论发布**：

```bash
gh issue comment <number> --repo ai-dynamo/dynamo --body-file /tmp/plan.md
```

### 追溯式 DEP

对于已合并、未经过 DEP 的工作，使用 `dep:implementing` 或 `dep:done` 提交，并引用现有 PR。

## 注意事项

- issue 正文就是规范——把它当作活文档对待。
- `dep:draft` 会自动应用。准备好后，PIC 会改为 `dep:under-review`。
- 轻量版 DEP 使用 `dep:lightweight` 标签，并省略可选章节。
- 计划修订时，发布一条新评论，并在顶部附上变更日志。不要修改原文——保留时间线。
- **客户名称剥离**：在创建或更新 DEP 之前，扫描 summary、motivation、proposal 及所有其他字段，查找具体的客户名、公司名或合作伙伴名。将其替换为通用引用（例如 "a customer"、"a cloud partner"、"an enterprise user"）。DEP 是公开的——issue 正文、评论和计划中都不应出现客户名。
