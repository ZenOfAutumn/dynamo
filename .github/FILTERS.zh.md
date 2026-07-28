# CI 过滤器

`filters.yaml` 文件根据变更的文件来控制执行哪些 CI 任务。

## 工作原理

当你打开一个 PR 时，CI 会检查变更了哪些文件，然后只运行相关的任务：

| 过滤器 | 触发的任务 |
|--------|-----------|
| `core` | 主测试套件（vLLM、SGLang、TRT-LLM 容器） |
| `operator` | Kubernetes operator 测试 |
| `deploy` | 部署相关的测试 |
| `vllm` / `sglang` / `trtllm` | 各后端（backend）专属测试 |
| `benchmarks` | Dynamo 运行时（runtime）流水线（运行 `tests/benchmarks/**` pytest 套件） |
| `sample` | 示例后端的统一测试（搭载在 vllm 镜像之上） |
| `docs` | 不触发任何任务（仅做分类） |
| `examples` | 不触发任何任务（仅做分类） |
| `ignore` | 不触发任何任务（仅做分类） |
| `rust` | Rust 合并前检查 |

> **注意：** `docs`、`examples` 与 `ignore` 不会触发任何 CI 任务。它们存在的目的是满足覆盖率要求 —— 每个文件至少要匹配一个过滤器。

## 修复 "Uncovered Files" 错误

如果 CI 失败并提示：
```
ERROR: The following files are not covered by any CI filter
```

请向 `filters.yaml` 中添加模式：

1. **新增源码文件** → 加入 `core` 或对应后端过滤器
2. **新的 examples/recipes** → 加入 `examples`
3. **文档** → 加入 `docs`
4. **不需要 CI 的配置文件** → 加入 `ignore`

## 本地测试

```bash
cd .github/scripts
npm install
npm run coverage  # 检查仓库内所有文件是否都被覆盖
```

## 模式语法

- `**` 匹配任意层级路径（默认不匹配点开头的文件）
- `*` 仅在单个目录内匹配
- `!pattern` 排除文件（在 `core` 中用于跳过文档）
- 如果要匹配点开头的文件，请显式写明类似 `dir/.*` 这样的模式

示例：`lib/**/*.rs` 匹配 `lib/` 下所有的 Rust 文件。

## 添加一个新的过滤器组

如果你在 `filters.yaml` 中新增了一个过滤器，还必须更新共享的
changed-files action，让覆盖率检查知道该过滤器：

1. 在 `filters.yaml` 中添加该过滤器。
2. 编辑 `.github/actions/changed-files/action.yml`：
   - 把新过滤器作为输出（output）暴露出来（参见文件顶部已有的
     `core`、`planner`、`vllm`、`sglang`、`trtllm` 等条目）。
   - 在 "Check for uncovered files" 步骤中，把它的
     `*_all_modified_files` 加到 `COVERED_FILES` 行。

如果跳过这一步，即使你的过滤器已经存在，CI 也会因为 "uncovered files" 而失败。

