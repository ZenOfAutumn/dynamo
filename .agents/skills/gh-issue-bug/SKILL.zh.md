---
name: dynamo-bug
description: File a GitHub bug issue against ai-dynamo/dynamo using context from the current conversation.
user-invocable: true
---

# 提交 Dynamo Bug Issue

利用当前对话上下文，通过 `gh` CLI 向 `ai-dynamo/dynamo` 提交结构良好的 bug 报告。

## 操作步骤

1. **从对话中收集上下文。** 回顾用户当前正在进行的工作、遇到的问题、错误信息、日志、堆栈跟踪以及已经讨论过的复现步骤。如果缺失关键信息，简要询问用户——但优先从对话上下文推断，而不是直接询问。

2. **收集环境信息。** 根据对话上下文判断用户运行在 **Kubernetes** 环境还是**本地开发**环境，然后收集相应的环境信息：

   对于 **Kubernetes** 环境：
   - K8s 版本 / 发行版（例如 EKS、GKE、kind）
   - Dynamo runtime 版本 / 容器镜像 tag
   - 节点 OS 与 CPU 架构
   - CUDA 版本与 GPU 架构（如适用）
   - Python 版本（如适用）
   - Helm chart 版本或 manifest 详细信息

   对于**本地开发**环境：
   - OS 及版本
   - Dynamo runtime 版本
   - CPU 架构
   - CUDA 版本与 GPU 架构（如适用）
   - Python 版本

   尽量使用 shell 命令自动检测能拿到的信息（例如 `uname -m`、`python3 --version`、`nvidia-smi`、`kubectl version`）。能填的填上，未知项标为 "N/A"。

3. **起草 issue**，使用以下模板，并在提交前请用户确认：

   ```
   **Describe the Bug**
   <clear, concise description>

   **Steps to Reproduce**
   1. ...
   2. ...
   <!-- Include relevant manifests or public container references if applicable -->

   **Expected Behavior**
   <what should have happened>

   **Actual Behavior**
   <what actually happened — include error messages, logs, or stack traces>

   **Environment**
   - **OS:** ...
   - **Dynamo Runtime Version:** ...
   - **CPU Architecture:** ...
   - **CUDA Version:** ...
   - **GPU Architecture:** ...
   - **Python Version:** ...
   <!-- Add K8s-specific fields if applicable -->
   ```

4. **将草稿展示给用户**，请其确认或修改后再提交。

5. **提交 issue**，使用：
   ```
   gh issue create --repo ai-dynamo/dynamo --title "<title>" --body "<body>"
   ```

6. **提交完成后，将 issue URL 返回给用户**。
