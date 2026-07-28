# 容器开发指南

## 概述

NVIDIA Dynamo 项目使用容器化的开发与部署，以在不同 AI 推理（inference）框架与部署场景之间保持一致的环境。本目录包含构建与运行 Dynamo 容器的工具：

### 渲染依赖

- Python
- Python 包：
  - pyyaml
  - jinja2

### 核心组件

- **`render.py`** - 用于为 AI 推理框架（vLLM、TensorRT-LLM、SGLang）以及前端镜像生成 Dockerfile 的渲染脚本。生成的 Dockerfile 包含开发与生产配置所需的多阶段步骤。

- **`run.sh`** - 容器运行时管理器，负责以正确的 GPU 访问、卷挂载与环境配置启动 Docker 容器。它支持从基于 root 的旧式配置到基于用户的开发环境等多种工作流。

### 各框架的 Stage 概览

<details>
<summary>展开 Stage 概览表</summary>
Dockerfile 通用结构

下面是各框架 Dockerfile stage 的通用文件结构概览，存在个别例外。

| Stage/Filepath | Target |
| --- | --- |
| **STAGE dynamo_base** | **FROM ${BASE_IMAGE}** |
| /bin/uv, /bin/uvx | COPY from ghcr.io/astral-sh/uv:latest (→ framework, runtime) |
|  /usr/bin/nats-server | Downloaded from GitHub (→ runtime) |
|  /usr/local/bin/etcd/ | Downloaded from GitHub (→ runtime) |
|  /usr/local/rustup/ | Installed via rustup-init (→ wheel_builder, dev) |
|  /usr/local/cargo/ | Installed via rustup-init (→ wheel_builder, dev) |
|  /usr/local/cuda/ | Inherited from BASE_IMAGE (→ wheel_builder, runtime) |
| **STAGE: wheel_builder** | **FROM quay.io/pypa/manylinux_2_28_${ARCH_ALT}** |
|  /usr/local/ucx/ | Built from source (→ runtime)
|  /opt/nvidia/nvda_nixl/ | Built from source (→ runtime)
|  /opt/nvidia/nvda_nixl/lib64/ | Built from source (→ runtime)
|  /opt/dynamo/target/ | Cargo build output (→ runtime) |
|  /opt/dynamo/dist/*.whl | Built wheels (→ runtime) |
|  /opt/dynamo/dist/nixl/ | Built nixl wheels (→ runtime) |
| **STAGE: framework** | **FROM ${BASE_IMAGE}** |
|  /opt/dynamo/venv/ | Created with uv venv (→ runtime) |
|  /${FRAMEWORK_INSTALL} | Built framework (→ runtime) |
| **STAGE: runtime** | **FROM ${RUNTIME_IMAGE}** |
|  /usr/local/cuda/{bin,include,nvvm}/ | COPY from dynamo_base |
|  /usr/bin/nats-server | COPY from dynamo_base |
|  /usr/local/bin/etcd/ | COPY from dynamo_base |
|  /usr/local/ucx/ | COPY from wheel_builder |
|  /opt/nvidia/nvda_nixl/ | COPY from wheel_builder |
|  /opt/dynamo/wheelhouse/ | COPY from wheel_builder |
|  /opt/dynamo/venv/ | COPY from framework |
|  /opt/vllm/ | COPY from framework |
|  /workspace/{tests,examples,deploy}/ |COPY from build context |
| **STAGE: dev** | **FROM runtime (via dev/Dockerfile.dev)** |
|  /usr/bin/, /usr/lib/, etc. | COPY from dynamo_tools (dev utilities, git, sudo, etc.) |
|  /usr/local/rustup/ | COPY from dynamo_tools |
|  /usr/local/cargo/ | COPY from dynamo_tools |
|  /usr/local/bin/maturin | COPY from dynamo_tools |
|  /opt/dynamo/venv/ | For SGLang: created with --system-site-packages, includes uv and maturin |
|  /workspace/ | Full source code copied from build context with editable install |
|  **💡 Recommendation** | **Use --mount-workspace with run.sh** for live editing (bind mount overrides baked-in code) |
|  PATH | Includes /opt/dynamo/venv/bin:/usr/local/cargo/bin |
|  umask 002 | Login shell sources /etc/profile.d/00-umask.sh for group-writable files |
| **STAGE: local-dev** | **FROM dev (via dev/Dockerfile.dev)** |
|  /home/dynamo/.rustup/ | COPY from /usr/local/rustup (user-writable) |
|  USER | dynamo (UID/GID remapped to match host user) |
|  **💡 Recommendation** | **Use --mount-workspace with run.sh** for live editing (bind mount overrides baked-in code) |
|  RUSTUP_HOME | /home/dynamo/.rustup |
|  CARGO_HOME | /home/dynamo/.cargo |
</details>

### 为什么要容器化？

每个推理框架（vLLM、TensorRT-LLM、SGLang）都有特定的 CUDA 版本、Python 依赖以及系统库。容器在开发与生产之间提供一致的环境、框架隔离以及正确的 GPU 配置。

本目录中的脚本将 Docker 命令的复杂性抽象出来，同时对构建与运行配置提供细粒度控制。

### 便捷脚本 vs. 直接 Docker 命令

`run.sh` 与渲染脚本是简化常见 Docker 操作的便捷工具。它们会自动处理：
- GPU 访问配置与 runtime 选择
- 开发工作流的卷挂载设置
- 环境变量管理
- 多阶段构建的 build 参数构造

**你始终可以直接使用 Docker 命令**，如果你希望对脚本之外的内容进行更精细的控制。`run.sh` 支持 `--dry-run` 参数以打印它们将会执行的具体命令，便于理解与修改底层操作。

## 开发 Target 特性矩阵

**注意**：在 Dynamo 中，"target" 与 "Docker stage" 同义。每个 target 对应多阶段 Docker 构建中的一个 stage。同样，"framework" 与 "engine" 同义（vLLM、TensorRT-LLM、SGLang）。

| 特性 | **runtime + `run.sh`** | **local-dev（`run.sh` 或 Dev Container）** | **dev + `run.sh`**（旧式） |
|---------|----------------------|-------------------------------------------|--------------------------|
| **用途** | 推理与部署的基准测试，非 root | 本地开发、编译与测试 | 旧式工作流，root 用户，使用需谨慎 |
| **用户** | dynamo（UID 1000） | dynamo（UID=宿主机用户）+ sudo | root（UID 0，使用需谨慎） |
| **Home 目录** | `/home/dynamo` | `/home/dynamo` | `/root` |
| **工作目录** | `/workspace`（容器内或挂载） | `/workspace`（已写入镜像，可选 `--mount-workspace` 挂载） | `/workspace`（已写入镜像，可选 `--mount-workspace` 挂载） |
| **Rust 工具链** | 无（使用预构建 wheel） | 系统安装（`/usr/local/rustup`、`/usr/local/cargo`） | 系统安装（`/usr/local/rustup`、`/usr/local/cargo`） |
| **Cargo Target** | 无 | `/workspace/target` | `/workspace/target` |
| **Python Env** | vllm/trtllm 用 venv（`/opt/dynamo/venv`），sglang 使用系统 site-packages | 所有框架用 venv（`/opt/dynamo/venv`，sglang 使用 `--system-site-packages`） | 所有框架用 venv（`/opt/dynamo/venv`，sglang 使用 `--system-site-packages`） |

**注意（SGLang）**：SGLang 运行时使用系统 site-packages，但 `dev` 与 `local-dev` 镜像会创建带 `--system-site-packages` 的 `/opt/dynamo/venv`，用于 `maturin`、`uv` 等构建工具。

## 使用建议

- **使用 runtime target**：用于推理与部署基准测试。以非 root `dynamo` 用户运行（UID 1000，GID 0）以确保安全
- **使用 local-dev + `run.sh`**：用于命令行开发与挂载本地分区。以 `dynamo` 用户运行，UID 与本地用户匹配，GID 0。交互会话需加 `-it`
- **使用 local-dev + Dev Container**：使用 VS Code/Cursor Dev Container 插件，以 `dynamo` 用户运行，UID 与本地用户匹配，GID 0
- **使用 dev + `run.sh`**：root 用户，使用需谨慎。为兼容早期工作流以 root 运行

## 示例命令

### 1. runtime target（以非 root dynamo 用户运行）：
```bash
# Build runtime image
container/render.py --framework vllm --target runtime --output-short-filename
docker build -t dynamo:latest-vllm-runtime -f container/rendered.Dockerfile .

# Run runtime container
container/run.sh --image dynamo:latest-vllm-runtime -it
```

### 2. test 镜像（在 runtime 之上叠加测试依赖）：
```bash
# Build test image from a runtime image (for running tests locally)
docker build -f container/Dockerfile.test --build-arg BASE_IMAGE=dynamo:latest-vllm-runtime -t dynamo:latest-vllm-test .
```

### 3. local-dev + `run.sh`（以 UID/GID 与宿主机匹配的 dynamo 用户运行）：
```bash
run.sh --mount-workspace -it --image dynamo:latest-vllm-local-dev ...
```

### 3. local-dev + Dev Container 扩展：
使用 VS Code/Cursor 的 Dev Container 扩展并配合 devcontainer.json。`dynamo` 用户的 UID 会自动与你的本地用户匹配。

## 构建与运行脚本概览

### render.py - Docker 镜像生成器

`render.py` 负责为不同 AI 推理框架生成 Dockerfile，支持多种框架与配置：

**用途：**
- 为 NVIDIA Dynamo 生成 Dockerfile，支持 vLLM、TensorRT-LLM、SGLang 或独立配置
- 处理框架特定的依赖与优化
- 管理 build context、缓存与多阶段构建
- 配置开发与生产 target

**关键特性：**
- **框架支持**：vLLM（未指定 --framework 时为默认）、TensorRT-LLM、SGLang 或 NONE（独立 Dynamo）
- **多阶段构建**：包含 base 镜像的构建流程
- **开发 target**：通过 `render.py` 支持 `dev`、`runtime` 与 `local-dev` target
- **构建缓存**：Docker 层缓存与 sccache 支持
- **GPU 优化**：CUDA、EFA 与 NIXL 支持

#### Dockerfile 中的 BuildKit 缓存挂载

各框架的 Dockerfile 使用 BuildKit 缓存挂载（`RUN --mount=type=cache,...`）以减少跨构建的重复下载。这些缓存存储在宿主机的 Docker/BuildKit 缓存中（不在你的宿主机 `~/.cache`），并由使用相同 builder 的构建共享。

常见缓存挂载目标：
- `--mount=type=cache,target=/root/.cache/uv`：`uv` 的下载缓存（wheels/sdists、`uv` 用到的 git checkout 等）
- `--mount=type=cache,target=/var/cache/apt,sharing=locked`：apt 下载缓存（`sharing=locked` 避免与并发构建产生 apt/dpkg 竞争）
- `--mount=type=cache,target=/var/cache/{yum,dnf},sharing=locked`：yum/dnf 元数据缓存（`sharing=locked` 避免并发构建破坏数据）
- `--mount=type=cache,target=/root/.cargo/{registry,git}`：Cargo crate/git 下载缓存（Cargo 自身有锁；不需要 `sharing=locked`）

查看缓存使用：
```bash
docker buildx du
docker info --format 'DockerRootDir: {{.DockerRootDir}}'
```

##### 在宿主机上检查 BuildKit 缓存（速查清单）

1. 快速摘要：
```bash
docker buildx du | tail -5
```

2. 查找 Docker 根目录：
```bash
docker info | grep "Docker Root Dir"
# Output example: Docker Root Dir: /var/lib/docker
```

3. 检查 executor 存储大小：
```bash
DOCKER_ROOT="$(docker info --format '{{.DockerRootDir}}')"
sudo du -sh "${DOCKER_ROOT}/buildkit/executor" 2>/dev/null || true
```

4. 查找特定缓存（示例：BuildKit executor rootfs 下的 uv 缓存）：
```bash
DOCKER_ROOT="$(docker info --format '{{.DockerRootDir}}')"
sudo sh -c 'find '"${DOCKER_ROOT}"'/buildkit/executor/*/rootfs/root/.cache/uv -type d 2>/dev/null | while read -r dir; do
  parent=$(dirname "$(dirname "$(dirname "$dir")")")
  du -sh "$parent/root/.cache/uv" 2>/dev/null
done'
```

5. 列出所有较大的缓存目录：
```bash
DOCKER_ROOT="$(docker info --format '{{.DockerRootDir}}')"
sudo sh -c 'du -sh '"${DOCKER_ROOT}"'/buildkit/executor/* 2>/dev/null | sort -h | tail -10'
```

清理命令：
```bash
# Safe: clean only reclaimable cache
docker buildx prune

# Aggressive: clean everything
docker buildx prune --all

# Time-based: remove cache older than 3 days
docker buildx prune --filter until=72h
```

各 Dockerfile 中挂载的当前缓存类型：
1. `/root/.cache/uv` 与 `/home/dynamo/.cache/uv` - Python 包（uv；与当前 `USER` 匹配）
2. `/root/.cargo/registry` - Rust crate
3. `/root/.cargo/git` - Rust git 依赖
4. `/var/cache/yum`、`/var/cache/dnf` - AlmaLinux 软件包
5. `/var/cache/apt` - Ubuntu 软件包

注意：`uv` 命令会按 `RUN` 设置 `UV_CACHE_DIR`，使 `uv` 始终与缓存挂载使用同一路径（不再依赖 `$HOME`）。

> **💡 提示**：`dev` 与 `local-dev` 镜像已将源码烤入镜像，但 **建议在开发时配合 `run.sh` 使用 `--mount-workspace`**，bind mount 你的本地工作区以便实时编辑。

**常见用例：**

```bash
# Build a vLLM local-dev image called dynamo:latest-vllm-local-dev. The local-dev image will run as `dynamo` with UID/GID matched to your host user,
# which is useful when mounting partitions for development.
container/render.py --framework=vllm --target=local-dev --output-short-filename
docker build --build-arg USER_UID=$(id -u) --build-arg USER_GID=$(id -g) -f container/rendered.Dockerfile -t dynamo:latest-vllm-local-dev .

# Build TensorRT-LLM runtime image called dynamo:latest-trtllm-runtime
container/render.py --framework=trtllm --target=runtime --output-short-filename --cuda-version=13.1
docker build -t dynamo:latest-trtllm-runtime -f container/rendered.Dockerfile .
```

构建后，使用 `run.sh` 启动容器（完整选项参见下方 [run.sh - 容器运行时管理器](#runsh---容器运行时管理器)）：
```bash
# Launch local-dev container with workspace mounted for live editing
container/run.sh --image dynamo:latest-vllm-local-dev --mount-workspace -it
```

### 构建前端镜像

前端镜像是一种特殊容器，包含 Dynamo 组件（Dynamo、NIXL 等）以及 Endpoint Picker（EPP），用于 Kubernetes Gateway API Inference Extension 集成。该镜像主要用于推理 gateway 部署。

**构建 EPP 镜像**
```bash
sudo apt-get update && sudo apt-get install -y git build-essential protobuf-compiler libclang-dev
curl --retry 5 --retry-delay 3 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain stable
. "$HOME/.cargo/env"
cargo install cbindgen

pushd deploy/inference-gateway/epp
make all
popd

EPP_GIT_TAG=$(git describe --tags --dirty --always 2>/dev/null || echo "dev")
EPP_IMAGE="dynamo/dynamo-epp:${EPP_GIT_TAG}"
```

**构建前端镜像**
```bash
# Build the frontend image (automatically builds EPP image as a dependency)
container/render.py --framework=dynamo --target=frontend --output-short-filename
docker build -t dynamo:frontend --build-arg EPP_IMAGE=${EPP_IMAGE} -f container/rendered.Dockerfile .
```

构建过程会自动：
1. 构建用于 EPP KV 感知路由的 Dynamo 静态库
2. 通过 `deploy/inference-gateway/epp/Makefile` 中的 `make all` 构建自定义 EPP Docker 镜像
3. 构建包含 EPP 二进制与 Dynamo 运行时组件的前端镜像

更多细节，请参阅 [`deploy/inference-gateway/README.md`](../deploy/inference-gateway/README.md)。

#### 前端镜像内容

前端镜像包含：
- **EPP（Endpoint Picker）**：处理推理 gateway 的请求路由与负载均衡
- **Dynamo Runtime**：核心平台组件与路由逻辑
- **NIXL**：NVIDIA InfiniBand 库，用于高性能网络通信
- **基准测试工具**：性能测试工具（aiperf、aiconfigurator 等）
- **Python 环境**：包含全部所需依赖的虚拟环境

#### 部署

前端镜像专为 Kubernetes Gateway API Inference Extension 部署而设计。完整的 Helm chart 部署说明请参阅 [`deploy/inference-gateway/README.md`](../deploy/inference-gateway/README.md)。

### run.sh - 容器运行时管理器

`run.sh` 脚本以适合开发与推理工作负载的配置启动 Docker 容器。

**用途：**
- 以正确的 GPU 访问运行预构建的 Dynamo Docker 镜像
- 配置卷挂载、网络与环境变量
- 支持不同的开发工作流（root 与基于用户）
- 管理容器生命周期与资源分配

**关键特性：**
- **GPU 管理**：自动 GPU 检测与分配
- **卷挂载**：工作区与 HuggingFace 缓存挂载
- **用户管理**：以非 root `dynamo` 用户运行（UID 1000、GID 0），可选 `--user` 覆盖
- **网络配置**：可配置网络模式（host、bridge、none、container 共享）
- **资源限制**：内存、文件描述符与 IPC 配置
- **交互模式**：使用 `-it` 进入交互终端会话（shell、调试与交互式开发所需）

**常见用例：**

```bash
# Basic container launch with dev image (runs as root by default, non-interactive)
container/run.sh --image dynamo:latest-vllm -v $HOME/.cache:/root/.cache

# Interactive development with workspace mounted using dev image (runs as root)
container/run.sh --image dynamo:latest-vllm --mount-workspace -it -v $HOME/.cache:/home/dynamo/.cache

# Interactive development with local-dev image (runs as dynamo user with matched host UID/GID)
container/run.sh --image dynamo:latest-vllm-local-dev --mount-workspace -it -v $HOME/.cache:/home/dynamo/.cache

# Use specific image and framework for development
container/run.sh --image v0.1.0.dev.08cc44965-vllm-local-dev --framework vllm --mount-workspace -it -v $HOME/.cache:/home/dynamo/.cache

# Interactive development shell with workspace mounted (local-dev)
container/run.sh --image dynamo:latest-vllm-local-dev --mount-workspace -v $HOME/.cache:/home/dynamo/.cache -it -- bash

# Development with custom environment variables
container/run.sh --image dynamo:latest-vllm-local-dev -e CUDA_VISIBLE_DEVICES=0,1 --mount-workspace -it -v $HOME/.cache:/home/dynamo/.cache

# Dry run to see docker command
container/run.sh --dry-run

# Development with custom volume mounts
container/run.sh --image dynamo:latest-vllm-local-dev -v /host/path:/container/path --mount-workspace -it -v $HOME/.cache:/home/dynamo/.cache

# Run runtime image as non-root dynamo user (for production)
container/run.sh --image dynamo:latest-vllm-runtime -v $HOME/.cache:/home/dynamo/.cache

# Run dev image as specific user (override default root)
container/run.sh --image dynamo:latest-vllm --user dynamo -v $HOME/.cache:/home/dynamo/.cache
```

### 网络配置选项

`run.sh` 通过 `--network` 标志支持不同的网络模式（默认 `host`）：

#### Host 网络（默认）
```bash
# Examples with dynamo user
container/run.sh --image dynamo:latest-vllm-local-dev --network host -v $HOME/.cache:/home/dynamo/.cache
container/run.sh --image dynamo:latest-vllm-local-dev -v $HOME/.cache:/home/dynamo/.cache
```
**适用场景：**
- 高性能 ML 推理（GPU 工作负载默认）
- 需要直接访问宿主机端口的服务
- 最大网络性能、最小开销
- 与宿主机共享服务（NATS、etcd 等）

**⚠️ 端口共享限制：** Host 网络会与宿主机共享所有端口，意味着在宿主机加上所有容器中你只能跑 **一个实例** 的 NATS（4222）或 etcd（2379）等服务。

#### Bridge 网络（隔离）
```bash
# CI/testing with isolated bridge networking and host cache sharing (no -it for automated CI)
container/run.sh --image dynamo:latest-vllm --mount-workspace --network bridge -v $HOME/.cache:/home/dynamo/.cache
```
**适用场景：**
- 与宿主机网络的安全隔离
- 需要完全隔离的 CI/CD 流水线
- 需要严格控制端口
- 在保持隔离的同时向宿主机暴露特定服务

**注意：** 与宿主机共享端口时，使用 `--port` 或 `-p`，格式为 `host_port:container_port`（如 `--port 8000:8000` 或 `-p 9081:8081`）将特定容器端口暴露给宿主机。

#### 无网络 ⚠️ **功能受限**
```bash
# Complete network isolation - no external connectivity
container/run.sh --image dynamo:latest-vllm --network none --mount-workspace -it -v $HOME/.cache:/home/dynamo/.cache

# Same with local-dev image (dynamo user with matched host UID/GID)
container/run.sh --image dynamo:latest-vllm-local-dev --network none --mount-workspace -it -v $HOME/.cache:/home/dynamo/.cache
```
**⚠️ 警告：`--network none` 会严重限制 Dynamo 功能：**
- **无法下载模型** - 无法下载 HuggingFace 模型
- **无 API 访问** - 无法访问外部 API 或服务
- **无法分布式推理** - 多节点配置无法工作
- **无监控/日志** - 外部监控系统不可达
- **调试受限** - 无法访问外部调试工具

**非常有限的适用场景：**
- 已下载模型且只做本地处理
- 气隙安全环境（模型必须预先准备）

#### 容器网络共享
使用 `--network container:name` 与另一个容器共享网络命名空间。

**适用场景：**
- Sidecar 模式（日志、监控、缓存）
- 服务网格架构
- 在相关容器之间共享网络命名空间

`--network container:name` 用法详见 Docker 文档。

#### 自定义网络
为多容器应用使用自定义 Docker 网络。通过 `docker network create` 创建，并通过 `--network network-name` 指定。

**适用场景：**
- 多容器应用
- 通过容器名进行服务发现

自定义网络创建与管理详见 Docker 文档。

#### 网络模式对比

| 模式 | 性能 | 安全 | 适用场景 | Dynamo 兼容性 | 端口共享 | 端口发布 |
|------|-------------|----------|----------|---------------------|---------------|-----------------|
| `host` | 最高 | 较低 | ML/GPU 工作负载、高性能服务 | ✅ 完整 | ⚠️ **与宿主机共享**（仅一个 NATS/etcd） | ❌ 不需要 |
| `bridge` | 良好 | 较高 | 通用 Web 服务、可控端口暴露 | ✅ 完整 | ✅ 隔离端口 | ✅ `-p host:container` |
| `none` | N/A | 最高 | 仅气隙环境 | ⚠️ **极受限** | ✅ 无网络 | ❌ 无网络 |
| `container:name` | 良好 | 中 | Sidecar 模式、共享网络栈 | ✅ 完整 | ⚠️ 与目标容器共享 | ❌ 使用目标容器端口 |
| 自定义网络 | 良好 | 中 | 多容器应用 | ✅ 完整 | ✅ 隔离端口 | ✅ `-p host:container` |

## 工作流示例

### 开发工作流
```bash
# 1. Build local-dev image (builds runtime, then dev as intermediate, then local-dev as final image)
container/render.py --framework=vllm --target=local-dev --output-short-filename
docker build --build-arg USER_UID=$(id -u) --build-arg USER_GID=$(id -g) -f container/rendered.Dockerfile -t dynamo:latest-vllm-local-dev .

# 2. Run development container using the local-dev image
# RECOMMENDED: --mount-workspace for live editing in dev and local-dev images
container/run.sh --image dynamo:latest-vllm-local-dev --mount-workspace -v $HOME/.cache:/home/dynamo/.cache -it

# From this point forward, commands run inside the container started in step 2.

# 3. Sanity check (optional but recommended)
deploy/sanity_check.py

# 4. Run inference (requires both frontend and backend)
# Start frontend
python -m dynamo.frontend &

# Start backend (vLLM example)
python -m dynamo.vllm --model Qwen/Qwen3-0.6B --gpu-memory-utilization 0.20 &
```

### 生产工作流
```bash
# 1. Build production runtime image (runs as non-root dynamo user)
container/render.py --framework=vllm --target=runtime --output-short-filename
docker build -t dynamo:latest-vllm-runtime -f container/rendered.Dockerfile .

# 2. Run production container as non-root dynamo user
container/run.sh --image dynamo:latest-vllm-runtime --gpus all -v $HOME/.cache:/home/dynamo/.cache
```

### 测试工作流
```bash
# 1. Build dev image
container/render.py --framework=vllm --target=dev --output-short-filename
docker build -t dynamo:latest-vllm-dev -f container/rendered.Dockerfile .

# 2. Launch the container
# Without --network (default: host networking, ports shared with host -- simplest for development)
container/run.sh --image dynamo:latest-vllm-dev --mount-workspace -v $HOME/.cache:/home/dynamo/.cache -it
# Or with --network bridge (isolated networking, no port conflicts with host)
container/run.sh --image dynamo:latest-vllm-dev --mount-workspace --network bridge -v $HOME/.cache:/home/dynamo/.cache -it

# From this point forward, commands run inside the container started in step 2.

# 3. Start infrastructure services
nats-server -js &
etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://0.0.0.0:2379 --data-dir /tmp/etcd &

# 4. Compile code
cargo build --locked --features dynamo-llm/block-manager --workspace
cd lib/bindings/python && maturin develop --uv && cd -

# 5. Sanity check (optional but recommended)
deploy/sanity_check.py --runtime-check-only

# 6. Run tests
python -m pytest tests/

# 7. (Optional) Start frontend and backend for interactive testing
python -m dynamo.frontend &

# Start worker backend (choose one framework):
# vLLM
DYN_SYSTEM_PORT=8081 python -m dynamo.vllm --model Qwen/Qwen3-0.6B --gpu-memory-utilization 0.20 --enforce-eager --no-enable-prefix-caching --max-num-seqs 64 &

# SGLang
DYN_SYSTEM_PORT=8081 python -m dynamo.sglang --model Qwen/Qwen3-0.6B --mem-fraction-static 0.20 --max-running-requests 64 &

# TensorRT-LLM
DYN_SYSTEM_PORT=8081 python -m dynamo.trtllm --model Qwen/Qwen3-0.6B --free-gpu-memory-fraction 0.20 --max-num-tokens 8192 --max-batch-size 64 &
```

**框架特定的 GPU 内存参数：**
- **vLLM**：`--gpu-memory-utilization 0.20`（使用 20% GPU 内存）、`--enforce-eager`（禁用 CUDA graphs）、`--no-enable-prefix-caching`（节省内存）、`--max-num-seqs 64`（最大并发序列数）
- **SGLang**：`--mem-fraction-static 0.20`（静态分配 20% GPU 内存）、`--max-running-requests 64`（最大并发请求数）
- **TensorRT-LLM**：`--free-gpu-memory-fraction 0.20`（保留 20% GPU 内存）、`--max-num-tokens 8192`（批次内最大 token 数）、`--max-batch-size 64`（最大 batch 大小）
