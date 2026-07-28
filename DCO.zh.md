# ✅ 修复 DCO 检查失败

**Developer Certificate of Origin（DCO）** 检查会确保所有提交都已 sign-off。
 如果你的 PR 未通过 DCO 检查，下面给出修复方法。

---

## 🖥️ 方式 1：通过 GitHub 网页编辑器修复
 ⚠️ 仅在你的 PR 只有 1 个提交时有效。

1. 进入你的 **Pull Request** → **Commits** 标签页。
2. 点击 **⋯ 菜单** → **Edit commit message**。
3. 在提交消息末尾添加这一行：

   ```text
   Signed-off-by: Your Name <your.email@example.com>
   ```
4. 保存修改 → GitHub 会创建一个带 sign-off 的新提交。
5. 重新运行 DCO 检查。

## 📦 方式 2：通过 GitHub Desktop 修复

1. 在 GitHub Desktop 中打开你的分支。
2. 进入 Repository → Repository Settings → Commit Behavior。
3. 勾选 ✅ Always sign-off commits。
4. Amend 最近一次提交：
      - 右键点击提交 → Amend Commit。
      - 在已启用 sign-off 的情况下再次保存。
5. 如有需要进行强制推送：
    ```
    git push --force-with-lease
    ```

## 💻 方式 3：通过 CLI 修复（多个提交）

1. 在配置中启用 sign-off：
   ```
    git config --global user.name "Your Name"
    git config --global user.email "your.email@example.com"
   ```
2. 交互式重新签署提交：
   ```
    git rebase -i HEAD~N
   ```
   将 N 替换为需要修复的提交数量。
   将提交标记为 edit，然后运行：
   ```
    git commit --amend --signoff
    git rebase --continue
   ```
3. 推送：
   ```
    git push --force-with-lease
   ```

## 🔀 与 main 同步后分支变得混乱

- 最简单的修复方式：将所有提交压缩为一个新的、已签署的提交（通过 Desktop 或 CLI）。
- 或者，请维护者使用 Squash and Merge 并在合并时附带 sign-off。

## ✨ 小技巧
这样可以确保你以后再也不会在 DCO 上失败。

- 在 CLI 提交时使用 -s 标志：
   ```
     git commit -s -m "Your commit message"
   ```
- 在你的客户端中开启 Always sign-off commits（GitHub Desktop 或 Git CLI）。
    1. GitHub Desktop
       在客户端中开启 Always sign-off commits。

    2. Git CLI
       你可以通过提交模板自动 sign-off 所有提交（注意：仅当你使用交互式 `git commit` 输入提交消息时有效，**不**适用于 `git commit -m "<message>"`）：
       1. 创建 ~/.git-commit-template.txt：
        ```
          Signed-off-by: Your Name <your.email@example.com>
         ```
    2. 告知 Git 使用它：
       ```
        git config --global commit.template ~/.git-commit-template.txt
       ```
