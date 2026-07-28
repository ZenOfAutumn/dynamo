---
name: pr-monitor
description: Check CI status, analyze failures, and explain skips for a Dynamo PR
user-invocable: true
disable-model-invocation: true
---

# PR CI Monitor

对一个 Dynamo pull request 进行完整的健康检查。以 PR 编号作为参数（例如 `/dynamo:pr-monitor 6554`）。

## 第 1 步：PR 概览

收集 PR 元数据，并判断该 PR 期望的 CI 形态。

```bash
gh pr view $PR_NUMBER --repo ai-dynamo/dynamo --json title,body,author,state,isDraft,additions,deletions,changedFiles,labels,reviewDecision,headRefName,baseRefName

gh pr diff $PR_NUMBER --repo ai-dynamo/dynamo --name-only
```

**判断完整 CI 是否应该运行。** dynamo 仓库的 CI 分为两层：

- **轻量级（`pre-merge.yml`）**：在所有 `pull_request` 事件上触发。会运行 pre-commit、版权检查、DCO，以及（在 rust filter 命中时）可选的 rust-clippy/rust-tests。这一层始终会跑。
- **完整流水线（`pr.yaml`、`container-validation-dynamo.yml`）**：仅在向 `main` 或向 `pull-request/[0-9]+` 分支 `push` 时触发。包括 docker 构建、GPU 测试、deploy 测试。它**不会**在普通 PR 分支上触发。

要在 PR 上跑完整 CI，必须存在一个 `pull-request/$PR_NUMBER` 分支（由 NVIDIA 维护者批准后由 copy-pr-bot 创建）。检查方式：

```bash
gh api repos/ai-dynamo/dynamo/branches/pull-request/$PR_NUMBER 2>/dev/null
```

如果该分支不存在，则完整 CI 还未被触发。常见原因：

1. **等待批准**：需要 NVIDIA 维护者在 PR 上评论 `/ok to test <commit_sha>` 来创建该分支并触发完整 CI。无论是 fork PR 还是来自尚未在批准列表中的内部作者的 PR，都适用。
2. **DCO 失败**：commit 未签名（缺少 `Signed-off-by` 行）。检查是否有 DCO bot 评论。修复：作者签名其 commit（参见 `DCO.md`）。
3. **Draft PR**：在被标记为 ready for review 之前，部分检查可能不会运行。

**核对实际针对 PR 的 HEAD commit 跑了哪些 workflow：**

```bash
# Get the HEAD SHA
HEAD_SHA=$(gh pr view $PR_NUMBER --repo ai-dynamo/dynamo --json headRefOid --jq '.headRefOid')

# Check which workflow runs exist for this SHA
gh api "repos/ai-dynamo/dynamo/actions/runs?head_sha=$HEAD_SHA" --jq '.workflow_runs[] | {name: .name, status: .status, conclusion: .conclusion}'
```

将实际跑了的 workflow 与预期对照。如果只看到 `Pre Merge`、`Copyright Checks`、`DCO Commenter` 等，但没有 `PR` 或 `Dynamo Validation`，说明完整 CI 未触发。

**判断哪些 CI filter 是激活的。** 如果完整 CI 已运行，请从实际 workflow run 中读取 `changed-files` 任务的输出 —— 这是 filter 判定的权威来源：

```bash
# Find the PR workflow run ID
PR_RUN_ID=$(gh api "repos/ai-dynamo/dynamo/actions/runs?head_sha=$HEAD_SHA" --jq '.workflow_runs[] | select(.name == "PR") | .id')

# Get the changed-files job output (look for the filter results in the logs)
gh api "repos/ai-dynamo/dynamo/actions/runs/$PR_RUN_ID/jobs" --jq '.jobs[] | select(.name == "changed-files") | {id: .id, status: .status, conclusion: .conclusion}'
```

如果完整 CI 还没跑过，则回退到拉取 `filters.yaml` 进行手工匹配：

```bash
gh api repos/ai-dynamo/dynamo/contents/.github/filters.yaml --jq '.content' | base64 -d
```

手工匹配时的关键规则：
- `core` 为 true 会触发**所有**框架流水线（vllm、sglang、trtllm）。
- filter 使用 YAML 锚点（如 `*ci`）—— 解读时要展开。
- 形如 `!**/*.md` 的取反模式意味着仅 markdown 改动**不会**触发该 filter。

**`pr.yaml` 中无视 filter 始终运行的任务：**
- `changed-files`、`deploy-operator`、`backend-status-check`、`clean-k8s-builder`、`cleanup` —— 这些任务无条件运行，或带有 `if: always()`。

## 第 2 步：CI 状态总览

获取所有 check run。注意：`gh pr checks` 在有 check 处于 pending 或 failed 时会返回非零退出码 —— 这是正常行为，并不是错误。

```bash
gh pr checks $PR_NUMBER --repo ai-dynamo/dynamo
```

如果完全没有返回任何 check，请回到第 1 步的诊断（DCO、批准、draft 状态）。

**忽略外部 CI check**（例如 GitLab 镜像 `ci/gitlab/*`）。它们是 NVIDIA 内部流水线，无法从 GitHub 检视。只分析 GitHub Actions 的 check。

**区分两种情况：**
1. **完整 CI 已触发**（workflow run 中包含 `PR`、`Dynamo Validation`）：照常分析所有任务。
2. **只跑了轻量级 CI**（仅 `Pre Merge` 与工具类 workflow）：清晰报告这一点。第 1 步的 filter 推断描述的是完整 CI 触发后**将会**运行什么；目前除了轻量级 check 之外没有可分析的内容。

如果完整 CI 已运行，请基于第 1 步的 filter 映射，**识别关键路径** —— 即与本 PR 改动最相关的 check。

按状态分组，给出简明的 dashboard：
- **Failed** —— 需要立刻关注
- **Pending/In-progress** —— 仍在运行（注明哪些处于关键路径）
- **Passed** —— 健康（仅计数即可，除非被要求列出）
- **Skipped** —— 在第 4 步处理

## 第 3 步：失败分析

对每个失败的 GitHub Actions 任务，深入日志找根因。

先在每个 run 中找出失败的任务：

```bash
gh api repos/ai-dynamo/dynamo/actions/runs/$RUN_ID/jobs --jq '.jobs[] | select(.conclusion == "failure") | {name: .name, id: .id, html_url: .html_url}'
```

然后获取特定失败任务的日志。`gh pr checks` 返回的 check URL 形如 `https://github.com/.../actions/runs/{RUN_ID}/job/{JOB_ID}`。提取 `RUN_ID`（第一个数字）：

```bash
gh run view $RUN_ID --repo ai-dynamo/dynamo --log-failed 2>&1 | tail -200
```

注意：`--log-failed` 会拼接所有失败任务的日志，可能比较吵。多任务失败时，建议按任务单独抓取，便于隔离根因。

对每个失败，报告：
- **任务名称**及其所属 workflow
- **根因** —— 实际错误（编译错误、测试断言、超时、基础设施问题、DCO 签名失败等）
- **关键日志片段** —— 关键几行（最多 20 行），不要全量倾倒
- **建议的修复** —— PR 作者可以采取的具体动作
- 如果疑似基础设施 flake，附上：`gh run rerun $RUN_ID --repo ai-dynamo/dynamo --failed`

如果没有失败，明确说明并继续。

## 第 3b 步：与 main 分支交叉对照

对第 3 步识别出的每个失败任务，检查同名任务在 main 上是否也失败。这有助于把 PR 引入的回归与既有的 flake 区分开。

**先检查 PR 落后于 main 的程度：**

```bash
gh api "repos/ai-dynamo/dynamo/compare/main...$HEAD_SHA" --jq '{behind_by: .behind_by, ahead_by: .ahead_by, status: .status}'
```

如果 PR 显著落后于 main（>20 个 commit），请在报告中注明 —— 一些失败可能在 main 上已被修复，看似的"回归"可能只是 PR 没跟上最新修复。

**然后检查 main 分支近期的 CI 结果：**

```bash
# Get last 3 completed PR workflow runs on main
MAIN_RUNS=$(gh api "repos/ai-dynamo/dynamo/actions/workflows/pr.yaml/runs?branch=main&per_page=3&status=completed" --jq '[.workflow_runs[].id] | join(" ")')

# For each failed job name from Step 3, check if it also failed on main
for RUN_ID in $MAIN_RUNS; do
  gh api "repos/ai-dynamo/dynamo/actions/runs/$RUN_ID/jobs?per_page=100&filter=latest" \
    --jq ".jobs[] | select(.name == \"$FAILED_JOB_NAME\") | {run: $RUN_ID, conclusion: .conclusion}"
done
```

**分类：**
- **`[PR-CAUSED]`** —— 在该 PR 上失败，但在最近的 main run 中均通过 → 很可能是该 PR 引入的回归。需要关注。
- **`[PRE-EXISTING]`** —— 在最近 main run 中也失败 → 既有 flake，与本 PR 无关。建议 rerun。
- **`[UNCLEAR]`** —— main 上结果混合（有时通过，有时失败）→ flaky 测试。注明 flaky 性并建议 rerun。
- **`[NEW JOB]`** —— main run 中没有该任务 → 本 PR 新增的 CI 任务，无法对比。

**报告中需要注意的提示：**
- 若 PR 落后 main 较多个 commit，`[PR-CAUSED]` 分类可能误报 —— 该失败可能在 main 上已修复。建议："考虑在调查前先 rebase 到 main。"
- 若 PR 已经领先于 main（包含 main 最新内容），分类比较可靠。
- 仅按任务名比对，不要按测试用例级别比对。任务在 main 上通过并不能保证同一组测试通过 —— 只能说明任务整体成功。

## 第 4 步：跳过与差异分析

本步骤**仅基于异常**。**不要**枚举预期之内的跳过 —— 那是噪声。

**如果完整 CI 未触发**，请整体跳过此步骤 —— 没有任务可分析。完整 CI 缺失的原因已在第 1-2 步说明。

**如果完整 CI 已运行**，请把第 1 步的 filter 结果与第 2 步的实际 CI 结果作对比。仅报告**意外**：

- 某 filter 应当为 `true`（文件命中其路径），但对应任务被跳过或缺失
- 关卡任务（`backend-status-check`、`dynamo-status-check`）被跳过 —— 它们使用 `if: always()`，应始终运行
- `core` 文件被改动时所有框架流水线却被跳过（core 应当触发所有框架）

**不算意外的情况**（不要报告）：
- 任务因其 filter 为 `false` 而跳过（例如未改文档时 docs 任务被跳过）
- pre-merge 上跳过 multi-GPU 测试（这些被门控到 post-merge/nightly）
- arm64 copy-to-acr 任务被跳过（仅在 merge 到 main 时触发）
- arm64 GPU 测试被跳过（GPU 测试只在 amd64 上跑）
- 上游被合理跳过导致下游被跳过
- 外部 CI（GitLab）状态 —— 整体忽略
- `operator=false` 时 `deploy-operator` 仍然运行 —— 这个任务在 `pr.yaml` 中始终运行

如果一切都符合预期，说一句 "No unexpected skips or discrepancies" 即可继续。

## 第 5 步：可执行总结

汇总成一份简明报告：

**PR Health: [PASSING | FAILING | PENDING | CI NOT TRIGGERED | PARTIAL — lightweight only]**

**如果完整 CI 未触发：**
- "完整 CI 等待批准 —— 需要 NVIDIA 维护者评论 `/ok to test <sha>` 以创建 `pull-request/$PR_NUMBER` 分支。"
- "DCO 检查失败 —— commit 需要签名。参见 DCO.md。"
- "Draft PR —— 一些检查在标记为 ready for review 前不会运行。"
- 注明哪些轻量级 check 通过/失败。

**阻塞性问题（PR 引入）** —— 在 main 上通过但在此 PR 失败：
- 每个：一行根因 + 建议修复
- 注明 `backend-status-check` 或 `dynamo-status-check` 关卡是否失败

**已存在的失败** —— 在 main 上也失败，与本 PR 无关：
- 每个：一行描述 + rerun 命令
- 如果 PR 落后 main 较多：建议 "考虑 rebase —— 这一项在 main 上可能已修复。"

**非阻塞问题**（如果有）：
- flaky 测试（来自第 3b 步的 `[UNCLEAR]`）、基础设施超时、意外跳过

**关键路径状态** —— 与 PR 改动最相关的 check：
- 列出当前状态（通过/失败/pending/未触发）
- 如果 pending，建议约 15 分钟后再查

**下一步行动** —— 按优先级排序的具体动作：
- "需要 NVIDIA 维护者评论 `/ok to test <sha>`" —— 当完整 CI 未触发
- "用 `git commit --amend -s` 给 commit 签名" —— DCO 失败时
- "在文件 Y 中修复 X" —— 代码失败时
- 基础设施 flake 的 rerun 命令：`gh run rerun $RUN_ID --repo ai-dynamo/dynamo --failed`
- 全部通过时："无需操作 —— CI 一切正常"

## 第 6 步：监控待定的 check

如果还有 check 处于 pending 或进行中，主动询问是否监控。

**列出剩余的 check：**

```bash
gh pr checks $PR_NUMBER --repo ai-dynamo/dynamo | grep -E 'pending|queued|in_progress'
```

报告：
- 还有多少个 check 处于 pending
- 哪些处于关键路径
- 基于已完成的相似任务估算等待时间（例如 `vllm-cuda12.9-amd64 / Test` 用了 20 分钟，那么仍在跑的 `vllm-cuda13.0-amd64 / Test` 估计还要约 20 分钟）

如果用户希望等待，则定期轮询：

```bash
# Re-check status
gh pr checks $PR_NUMBER --repo ai-dynamo/dynamo | grep -cE 'pass|fail|skipped'  # completed count
gh pr checks $PR_NUMBER --repo ai-dynamo/dynamo | grep -cE 'pending|queued|in_progress'  # remaining count
```

所有 check 完成后，用最终结果重跑第 5 步的总结。报告任何在上次查询后从 pending 变为 failed 的 check。

## 行为说明

- **并发取消**：如果 PR 有连续 push，更早的 run 会被取消。看到 cancelled run 时请注明，并建议改看最新的 run。
- **大日志输出**：始终截取相关片段。总结里不要倾倒超过 50 行的原始日志。
- **速率限制**：如果 `gh` 命令因为速率限制失败，报告已收集到的内容并建议稍后重试。
- **多 workflow**：单次 push 可能触发 `pr.yaml`、`pre-merge.yml` 与 `container-validation-dynamo.yml`。请全部检查。
- **`pull-request/[0-9]+` 分支**：在维护者批准后由 copy-pr-bot 创建。完整 CI 必需 —— 对 fork PR 与内部 PR 都适用。
- **`external-contribution` 标签**：fork PR 会自动获得。它的存在可确认 PR 来自外部贡献者。
- **外部 CI（GitLab）**：彻底忽略 `ci/gitlab/*` check。它们是 NVIDIA 内部流水线，无法从 GitHub 诊断。
