---
name: dep-update
description: Update Dynamo Enhancement Proposal lifecycle state in GitHub, including triage, PIC assignment, review, approval, and status label changes.
---

# 技能：更新 DEP 生命周期

## 目的

更新 DEP 在其生命周期中的状态 —— 分流（triage）、评审（review）、批准
（approve）、推迟（defer）或关闭。涵盖 PIC 工作流，从最初分配到最终批准。

## 何时使用

当对 DEP 议题进行分流、作为 PIC 或评审者评审 DEP、批准处于评审中的
DEP，或更新 DEP 状态时使用。

## 工作流

### 分流（指派 PIC）

1. **列出未指派的 DEP**：

```bash
gh issue list --repo ai-dynamo/dynamo \
  --label "dep:draft" \
  --json number,title,labels,assignees \
  --jq '.[] | select(.assignees | length == 0)'
```

2. **根据 area 标签指派 PIC**：

```bash
gh issue edit <number> --repo ai-dynamo/dynamo \
  --add-assignee "<github-username>"
```

3. **当 spec 准备好后转入评审**：

```bash
gh issue edit <number> --repo ai-dynamo/dynamo \
  --remove-label "dep:draft" \
  --add-label "dep:under-review"
```

### 评审

1. **阅读 DEP issue 及其讨论**：

```bash
gh issue view <number> --repo ai-dynamo/dynamo
gh issue view <number> --repo ai-dynamo/dynamo --comments
```

2. **将评审反馈作为评论发布到 issue 上**。

3. **向作者请求修改或澄清**。

### 批准

1. **确认 issue 处于评审中**：

```bash
gh issue view <number> --repo ai-dynamo/dynamo --json labels
```

2. **发布批准评论**：

```bash
gh issue comment <number> --repo ai-dynamo/dynamo --body "/approve"
```

3. **如果是 PIC 进行批准**（或所有必需的评审者均已批准），更新标签：

```bash
gh issue edit <number> --repo ai-dynamo/dynamo \
  --remove-label "dep:under-review" \
  --add-label "dep:approved"
```

## 备注

- 对于直接明了的 DEP，PIC 的 `/approve` 即可。
- 对于多评审者 DEP，PIC 维护一份置顶的批准检查清单，仅在所有必需的批准
  全部到位时才更新标签。
- `/approve` 评论可被搜索以便审计：
  `gh search issues --repo ai-dynamo/dynamo "/approve" in:comments`
- area 标签为裸名称（如 `frontend`、`router`），不带前缀。
