<!--
SPDX-FileCopyrightText: Copyright (c) 2024-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# NVIDIA Dynamo 开发环境

> 警告：Dev Containers（即 `devcontainers`）仍在持续演进，且我们没有在 CI 中对其进行测试。如有任何问题或反馈，请通过 GitHub Issues 提交。

## 已知问题

### Docker 版本兼容性

**Docker 29.x 与 Dev Containers 扩展 v1.0.26（CLI v0.75.0）及更早版本之间存在已知不兼容：**

- **现象：** 容器构建并启动成功（输出 "Container started"），但 IDE 一直挂起、无法连接，`postCreateCommand` 也始终不执行。
- **根因：** 内置的 devcontainers CLI 通过 `docker events --format {{json .}} --filter event=start` 来检测容器启动事件。它会检查解析后 JSON 里的 `i.status || i.Status`。但是 Docker 29.x 把事件类型放在字段名 `"Action"` 下（例如 `{"Action":"start",...}`），而不是 `"status"`。因此 start 事件虽然被收到，却因字段名不匹配而被静默丢弃，CLI 就一直无限等待。
- **影响范围：** Docker Engine 29.0.0+ 搭配 Dev Containers 扩展 v1.0.26（CLI v0.75.0）。
- **修复版本：** Dev Containers 扩展 **v1.0.32+**。
- **已确认正常的组合：**
  - Docker 27.x / 28.x 与任意扩展版本
  - Docker 29.x 与 Dev Containers 扩展 v1.0.32+

**推荐的修复：** 如果你正在使用 Docker 29.x，请将 Dev Containers 扩展升级到 **v1.0.32 或更高版本**。Docker Engine 28.5.2 及以下版本与所有扩展版本一直保持稳定。

## 各框架专属的 devcontainer

本目录下包含由 Jinja2 模板生成的、针对各推理框架的 devcontainer 配置：

- **`vllm/`** —— vLLM 框架的开发环境
- **`sglang/`** —— SGLang 框架的开发环境
- **`trtllm/`** —— TensorRT-LLM 框架的开发环境

### 模板系统

devcontainer 配置由以下两个文件生成：
- **`devcontainer.json.j2`** —— 带框架变量的 Jinja2 模板
- **`gen_devcontainer_json.py`** —— 生成配置的 Python 脚本

修改后重新生成各框架专属配置：
```bash
cd .devcontainer
python3 gen_devcontainer_json.py
```

**重要提示：** 请勿直接编辑生成出来的 `devcontainer.json` 文件。它们包含自动生成的警告，下次再生成时会被覆盖。请编辑 `devcontainer.json.j2` 模板再重新生成。

#### 为什么使用模板，而不是单一的 devcontainer.json

Dev Container 扩展要求每个 `devcontainer.json` 都遵循特定的目录约定，这就导致针对不同框架的配置之间存在大量重复。可参见 https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers

```
📁 project-root
    📁 .git
    📁 .devcontainer
      📁 python-container
        📄 devcontainer.json
      📁 node-container
        📄 devcontainer.json
    📁 python-src
        📄 hello.py
    📁 node-src
        📄 hello.js
    📄 docker-compose.yml
```

我们也尝试过一些未公开的方案（例如修改 devcontainer.json 文件名、自定义配置等），但都没有成功。模板系统是在保持与 Dev Container 扩展目录约定兼容的同时，最大限度减少重复的折衷方案。

在 Microsoft Dev Container 扩展提供新能力之前，这仍是我们推荐的多 Dev Container 配置管理方式。

```mermaid
graph TB
    subgraph "Developer Laptop"
        subgraph IDE["Cursor/VS Code"]
            EXT["Dev Container Extension"]
            SSHEXT["Remote-SSH Extension"]
        end
        TERM["iTerm/Terminal"]
    end

    subgraph WS["Linux Workstation"]
        DIR["~/dynamo<br/>(host directory)"]
        HFCACHE["~/.cache/huggingface<br/>(host cache)"]
        GITCONFIG["~/.gitconfig<br/>(host git config)"]

        subgraph CONTAINER["Docker Container<br/>vsc-dynamo-SHA-uid<br/>Running as: dynamo user"]
            MOUNT["/home/dynamo/dynamo<br/>(mounted directory)"]
            HFMOUNT["/home/dynamo/.cache/huggingface<br/>(mounted cache)"]
            GITCONFIGCOPY["/home/dynamo/.gitconfig<br/>(via Dev Container setting)"]
            TOOLS["rust-analyzer<br/>cargo<br/>etc."]
        end

        IMAGE["Docker Image<br/>dynamo:latest-{framework}-local-dev<br/>(vllm/sglang/trtllm)"]

        IMAGE -->|"docker run<br/>as dynamo user"| CONTAINER
    end

    TERM -->|"SSH Connection"| DIR
    SSHEXT -->|"1. Remote-SSH"| DIR
    EXT -->|"2. Dev Container:<br/>Open Folder (via ssh)"| IMAGE
    EXT -->|"3. IDE/shell via<br/>ssh+docker exec"| CONTAINER
    DIR <-.->|"Volume Mount<br/>(bidirectional)"| MOUNT
    HFCACHE <-.->|"Volume Mount<br/>(HF cache)"| HFMOUNT
    GITCONFIG -->|"Copy"| GITCONFIGCOPY

    style IDE fill:#e1f5fe
    style EXT fill:#cfe2ff
    style SSHEXT fill:#cfe2ff
    style TERM fill:#e1f5fe
    style DIR fill:#fff3e0
    style HFCACHE fill:#fff3e0
    style GITCONFIG fill:#fff3e0
    style IMAGE fill:#f3e5f5
    style CONTAINER fill:#d4edda
    style MOUNT fill:#fff9c4
    style HFMOUNT fill:#fff9c4
    style GITCONFIGCOPY fill:#fff9c4
    style TOOLS fill:#ffebee
```

## 前置条件

开始之前，请确保已安装以下内容：

- 在宿主机上安装并配置好 [Docker](https://docs.docker.com/get-started/get-docker/)
- IDE：使用 VS Code 或 Cursor。两者都支持 Dev Containers 扩展
- 合适的 NVIDIA 驱动（与 CUDA 12.8+ 兼容）
- 对于需要鉴权的模型，请在你本地的启动脚本（`.profile` 或 `.zprofile`）中设置 Hugging Face token 环境变量 `HF_TOKEN`。许多公开模型并不需要该 token。

### 必需的文件与目录

宿主机上必须存在以下路径：

- **`~/.cache/huggingface`**：该目录会挂载到容器中，用于 Hugging Face 模型缓存。


## 入门步骤

按以下步骤启动你的 NVIDIA Dynamo 开发环境：

### 步骤 0：构建开发容器镜像

从源码完整构建合适框架的镜像（例如 `dynamo:latest-vllm-local-dev`）：

```bash
# Single command approach (recommended)
export FRAMEWORK=vllm         # Note: any of vllm, sglang, trtllm can be used
python container/render.py --framework=${FRAMEWORK} --target=local-dev --output-short-filename
docker build --build-arg USER_UID=$(id -u) --build-arg USER_GID=$(id -g) -t dynamo:latest-${FRAMEWORK}-local-dev -f container/rendered.Dockerfile .

# Now you've created dynamo:latest-vllm-local-dev
```

local-dev 镜像会让容器内的本地用户权限与你的宿主机用户保持一致，并附带额外的开发工具（调试工具、文本编辑器、系统监控工具等）。

### 步骤 1：选择框架

根据你使用的框架选择对应的 devcontainer：
- vLLM 开发使用 `vllm/devcontainer.json`
- SGLang 开发使用 `sglang/devcontainer.json`
- TensorRT-LLM 开发使用 `trtllm/devcontainer.json`

在 VS Code/Cursor 里打开 devcontainer 时，请进入对应的框架子目录（例如 `.devcontainer/vllm/`），并打开其下的 devcontainer.json。

### 步骤 2：安装 Dev Containers 扩展

**对于 Cursor：**
- 按 `Cmd+Shift+X`（Mac）或 `Ctrl+Shift+X`（Linux/Windows）打开扩展面板
- 搜索 "Dev Containers"，安装 **Anysphere** 出品的版本（不要安装 Microsoft 版本，那个版本与 Cursor 不兼容）

**对于 VS Code：**
- 从 Microsoft 应用商店安装 [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)


### 步骤 3：启动开发环境

1. 在 IDE 中打开 `dynamo` 文件夹
2. 按 `Cmd+Shift+P`（Mac）或 `Ctrl+Shift+P`（Linux/Windows）
3. 选择 "Dev Containers: Open Folder in Container"，再选择你的 `dynamo` 文件夹并打开

### 步骤 4：可选但强烈推荐的设置

为了获得最佳开发体验，建议进行以下配置：

#### Git 配置
- **个人 `.gitconfig`**：如果还没有，请创建你自己的 `~/.gitconfig`，配置好姓名、邮箱以及偏好的 Git 设置
- **Dev Container 集成**：依次进入 IDE Settings → Extensions → Dev Containers，勾选 **☑ Copy the .gitconfig file to the devcontainer**

#### SSH 鉴权
若要在容器内顺畅地进行基于 SSH 的 Git 操作（push/pull）：

1. **在宿主机配置 ssh-agent**：搜索 "how to set up ssh-agent on Ubuntu on .profile"，确保它在你的 `~/.profile` 中配置好，以便 ssh-agent 在 Cursor 启动前就已运行

2. **在 IDE 中开启 SSH forwarding**：
   Dev Containers 设置中勾选 **☑ Dev Containers: Enable SSHAgent Forwarding**

3. **验证 SSH forwarding 是否生效**：
   - `post-create.sh` 脚本会检查并报告 SSH agent 状态
   - 容器启动时你应当能看到 "SSH agent forwarding is working"

完成这些配置后，你可以在容器中正常使用 Git 命令，无需额外鉴权步骤。


### 步骤 5：等待初始化

容器会自动完成：
- 把你本地的代码挂载到 `/workspace`
- 运行 `post-create.sh` 来构建项目并配置环境

如果 `post-create.sh` 失败，可以尝试自行排查，或在 GitHub 上 [提 issue](https://github.com/ai-dynamo/dynamo/issues)。

## 开发流程

### 构建 Rust 代码

如果你修改了 Rust 代码并希望编译，请使用 [cargo build](https://doc.rust-lang.org/cargo/commands/cargo-build.html)，它会更新 Rust 二进制：

```bash
cd /workspace && cargo build --locked --profile dev
```

确认构建产物位于预设的 `target` 目录：
```bash
$ cargo metadata --format-version=1 | jq -r '.target_directory'
/workspace/target  <-- this is the target path
```

如果 cargo 没有正确安装与配置，你会看到一个或多个错误，例如：
```
error: could not find `Cargo.toml` in $HOME or any parent directory
```

最后，在把代码推送到 GitHub 之前，记得运行 `cargo fmt` 与 `cargo clippy`。

### 更新 Python 绑定

如果你修改了 Rust 代码并希望同步到 Python 绑定，可以使用 [maturin](https://www.maturin.rs/#usage)（已预装）。它会用你最新的 Rust 修改更新 Python 绑定：

```bash
cd /workspace/lib/bindings/python && maturin develop
```

## 内置内容
开发环境：
- Rust 与 Python 工具链
- GPU 加速
- VS Code 或 Cursor 的 Rust/Python 扩展
- 在 `target/` 目录下持久化构建缓存，通过 `cargo build --locked --profile dev` 实现快速增量构建（仅重新编译变更的文件）
- 由于使用 volume 挂载，文件改动会同步到本地仓库
- 由 devcontainer 编排好的 SSH 与 GPG agent 透传

文件结构：
- 本地 dynamo 仓库挂载至 `/workspace`
- Python 虚拟环境位于 `/opt/dynamo/venv`
- 构建产物位于 `/workspace/target`
- Hugging Face 缓存在多次会话间保留（既可以挂载宿主机的 `.cache`，也可以把你的 `HF_HOME` 挂到 `/home/dynamo/.cache/huggingface`）
- Bash 历史在多次会话间保留，位于 `/home/dynamo/.commandhistory`，使用 docker volume `dynamo-bashhistory`
- pre-commit 缓存在多次会话间保留，位于 `/home/dynamo/.cache/precommit`，使用 docker volume `dynamo-precommit-cache`

## 文档

查看文档可运行：
```bash
cd /workspace/target/doc && python3 -m http.server 8000
```

VSCode 会自动做端口转发，你可以在浏览器中查看文档。

## 常见问题

### 用 GPG 密钥签名 Git 提交
按照 [VSCode 文档](https://code.visualstudio.com/remote/advancedcontainers/sharing-git-credentials#_sharing-gpg-keys) 的说明，GPG 签名应当开箱即用。

如果遇到版本兼容性问题，可以尝试：

```bash
# On Host
gpg --list-secret-keys
gpg --export-secret-keys --armor YOUR_KEY_ID > /tmp/key.asc

# In container
gpg1 --import /tmp/key.asc
git config --local gpg.program gpg1
```

> 警告：把本地 gpg 切换为 gpg1 在你离开容器后可能带来后续影响。

### 自定义 devcontainer.json 配置

你可以把主配置复制到 `.devcontainer` 下的另一个目录中，从而创建自定义 devcontainer 配置。下面例子使用的目录名是 `jensen_dev`，你可以随意命名：

```bash
# By convention, Dev Container will look at the project's .devcontainer/<path>/devcontainer.json file.
# Example: copy the main devcontainer configuration and then edit the new json file
mkdir -p .devcontainer/jensen_dev
cp .devcontainer/devcontainer.json .devcontainer/jensen_dev/devcontainer.json
```

常见的自定义包括：额外的挂载、环境变量、IDE 扩展、构建参数等。打开新的 Dev Container 时，你可以从所有可用的 `.devcontainer/<path>/devcontainer.json` 中选择。


### Git 操作所需的 SSH 密钥

只要你在宿主机上启动了 ssh-agent，`git push` 应该可以正常工作。否则，你需要先配置 ssh-agent，或者在容器内自行准备 SSH 密钥（更麻烦）。


## 故障排查

### 容器内的环境变量没生效？

出于安全考虑，dev 容器对宿主机环境变量的访问是受限的。下面介绍正确传递环境变量的方法：

#### 方法 1：使用 devcontainer.json
在 `.devcontainer/devcontainer.json` 中添加环境变量：
```json
{
  "remoteEnv": {
    "HF_TOKEN": "${localEnv:HF_TOKEN}",
    "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}",
    "SSH_AUTH_SOCK": "${env:SSH_AUTH_SOCK}"
  }
}
```

#### 方法 2：宿主机 shell 配置
要让宿主机的环境变量在 Dev Containers 中可用：

**对于 bash 用户：**
- 把变量放进 `~/.profile`（用于登录 shell）
- 如果同时存在 `~/.bash_profile` 和 `~/.bashrc`，请确保前者会 source 后者
- 有些用户反馈 `~/.bashrc` 中的变量不会被 dev 容器读取，这种情况下可以把变量改放到 `~/.profile` 中再试

**对于 zsh 用户：**
- 把变量放进 `~/.zprofile`（用于登录 shell）
- 有些用户反馈 `~/.zshrc` 中的变量不会被 dev 容器读取，这种情况下可以把变量改放到 `~/.zprofile` 中再试

#### 方法 3：使用环境文件
在项目根目录创建 `.env`，并在 devcontainer.json 中引用：

**`.env` 示例：**
```bash
# API Tokens
HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Development settings
DEBUG=true
LOG_LEVEL=debug

# Custom paths
CUSTOM_MODEL_PATH=/path/to/models
```

**在 devcontainer.json 中引用：**
```json
{
  "remoteEnv": {
    "ENV_FILE": "${localWorkspaceFolder}/.env"
  }
}
```

更多细节请参见 VS Code Dev Containers [文档](https://code.visualstudio.com/docs/devcontainers/containers)。

### 构建问题

如果遇到构建错误或异常的编译问题，可以先运行 `cargo clean`，然后从头重新构建。

如果 `cargo clean` 仍无法解决，有可能是某些文件由 root 创建（通过 `run.sh` 脚本所致）。这时你可以在宿主机（容器外）上手动删除 target 目录：

```bash
sudo rm -rf <your dynamo path on the host machine>/target
```

### 卷损坏问题

如果遇到怪异的错误（例如 `postCreateCommand` 退出码为 1），有可能是 Docker volume 损坏。

**解决办法：清理 Docker volume**

```bash
# Remove Dynamo volumes that are specified in devcontainer.json (may be corrupted)
docker volume rm dynamo-bashhistory dynamo-precommit-cache

# Or remove all volumes (use with caution).
docker rm -f <your running container(s)>
docker volume prune -f
```

**注意：** 这会同时清空 bash 历史和 pre-commit 缓存。

**devcontainer.json 中的 volume 挂载：**
- `dynamo-bashhistory` → `/home/dynamo/.commandhistory`（bash 历史）
- `dynamo-precommit-cache` → `/home/dynamo/.cache/pre-commit`（pre-commit 缓存）

### 权限问题

如果开始遇到权限相关的问题（例如 "Permission denied"），可能需要在容器外修复文件归属。常见原因是 `container/run.sh` 以 root 身份运行，从而创建了 root 所有的文件：

```bash
# Replace <user> with your actual username
cd <your dynamo directory at your host machine (not docker)>
sudo chown -R <user>:<user> .
```

这能修正宿主机和容器之间因 UID 不一致导致的归属问题。

### 容器启动后立即停止

如果你在 devcontainer 日志里看到 "container is not running" 或 "An error occurred setting up the container" 等错误，说明容器启动后立即崩溃。

**常见原因与解决办法：**

1. **缺少 local-dev 镜像：**
   ```bash
   # Check if the required local-dev image exists
   docker images | grep dynamo

   # If missing, build the dev image first, then build local-dev
   export FRAMEWORK=vllm  # Replace with vllm, sglang, or trtllm
   python container/render.py --framework=${FRAMEWORK} --target=local-dev --output-short-filename
   docker build --build-arg USER_UID=$(id -u) --build-arg USER_GID=$(id -g) -t dynamo:latest-${FRAMEWORK}-local-dev -f container/rendered.Dockerfile .
   ```

2. **容器启动失败：**
   ```bash
   # Check container logs for the specific error
   docker logs <container-id>

   # Or check all recent containers
   docker ps -a --filter "label=devcontainer.local_folder=$(pwd)"
   ```

3. **资源不足：**
   ```bash
   # Check available system resources
   free -h
   df -h

   # Restart Docker daemon if needed
   sudo systemctl restart docker
   ```

4. **彻底清理后重来：**
   ```bash
   # Remove all related containers and images
   docker ps -a --filter "label=devcontainer.local_folder=$(pwd)" -q | xargs docker rm -f
   docker images | grep "^vsc-" | awk '{print $3}' | xargs docker rmi
   ```
  然后在 IDE 中不带缓存重建：
  *Dev Containers: Rebuild Without Cache and Reopen in Container*

### 修改 devcontainer.json 没有生效

如果你修改了 `devcontainer.json`、`post-create.sh` 或其他 devcontainer 相关文件，但重建容器时没有生效，可能是被缓存了。

**解决办法：强制重建 devcontainer**

1. **重建容器（推荐）：**
   在 IDE 命令面板（Ctrl+Shift+P）中：
   *Dev Containers: Rebuild Container*

2. **如果还不生效，请不带缓存重建：**
   在 IDE 命令面板（Ctrl+Shift+P）中：
   *Dev Containers: Rebuild Without Cache and Reopen Container*

3. **如果问题持续存在，手动删除 devcontainer 镜像：**
   ```bash
   # List devcontainer images
   docker images | grep devcontainer

   # And remove all IDE devcontainer images (more thorough)
   docker images | grep "^vsc-" | awk '{print $3}' | xargs docker rmi

   # Then rebuild in your IDE
   Dev Containers: Rebuild Container
   ```

**注意：** "Rebuild Container Without Cache and Reopen Container" 选项最为彻底，能确保你的修改全部生效，但因为是从零开始重建，所需时间也更长。
