---
name: dep-status
description: Check Dynamo Enhancement Proposal issue status, list DEPs by lifecycle state or area, and find related DEP issues in ai-dynamo/dynamo.
---

# Skill：检查 DEP 状态

## 目的

列出 DEP issue 及其当前状态、所属 area、PIC（负责人）和审批状态。查找与给定主题或组件相关的 DEP。

## 何时使用

当用户希望查看一个或多个 DEP 的状态、查询哪些 DEP 正在等待审查、查找与某组件相关的 DEP，或获取一份分类整理（triage）摘要时使用。

## 工作流程

1. **列出处于打开状态的 DEP issue**：

```bash
gh issue list --repo ai-dynamo/dynamo \
  --search 'label:"dep:draft","dep:under-review","dep:approved","dep:implementing"' \
  --json number,title,labels,assignees,createdAt,updatedAt
```

2. **按 area 过滤**（如有需要）：

```bash
gh issue list --repo ai-dynamo/dynamo \
  --label "<area>" \
  --json number,title,labels,assignees
```

3. **按状态过滤**（如有需要）：

```bash
gh issue list --repo ai-dynamo/dynamo \
  --label "dep:<status>" \
  --json number,title,labels,assignees
```

4. **以摘要表格形式呈现**：

```text
| # | Title | Status | Area | PIC | Updated |
|---|-------|--------|------|-----|---------|
| 42 | DEP: KV router scheduling | dep:under-review | router | @pic | 2026-03-28 |
```

5. **通过搜索 issue 标题与正文查找相关 DEP**：

```bash
gh issue list --repo ai-dynamo/dynamo \
  --search 'DEP <keyword> label:"dep:draft","dep:under-review","dep:approved","dep:implementing","dep:done"' \
  --json number,title,labels,state
```

6. **如需要，包含已关闭的 DEP**：

```bash
gh issue list --repo ai-dynamo/dynamo \
  --state closed \
  --search 'label:"dep:done","dep:deferred","dep:rejected","dep:replaced"' \
  --json number,title,labels,assignees,closedAt
```

## 备注

- 完整 triage 视图应同时包含打开的与最近关闭的 DEP。
- 通过 `dep:lightweight` 标签可区分完整 DEP 与轻量级 DEP。
- area 标签为裸名（例如 `frontend`、`router`）—— 没有前缀。
