---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
sidebar-title: 从源码构建
description: 为开发与贡献从源码构建 Dynamo
---

# 从源码构建

当你想贡献代码、在开发分支上测试特性或自定义构建时，从源码构建 Dynamo。如果你只想运行 Dynamo，[本地安装](local-installation.md)指南更快。

本指南覆盖 Ubuntu 与 macOS。如需自动处理这些步骤的容器化开发环境，参见 [DevContainer](#devcontainer)。

## 1. 安装系统库

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

## 2. 安装 Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

## 3. 创建 Python 虚拟环境

如果还没有 [uv](https://docs.astral.sh/uv/#installation)，先安装：

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

创建并激活虚拟环境：

```bash
uv venv .venv
source .venv/bin/activate
```

## 4. 安装构建工具

```bash
uv pip install pip maturin
```

[Maturin](https://github.com/PyO3/maturin) 是 Rust-Python 绑定的构建工具。

## 5. 构建 Rust 绑定

```bash
cd lib/bindings/python
maturin develop --uv
```

## 6. 安装 GPU Memory Service

```bash
# Return to project root
cd "$(git rev-parse --show-toplevel)"
uv pip install -e lib/gpu_memory_service
```

## 7. 安装 wheel

```bash
uv pip install -e .
```

## 8. 验证构建

```bash
python3 -m dynamo.frontend --help
```

应当看到 frontend 命令的帮助输出。

## DevContainer

VSCode 与 Cursor 用户可以跳过手动配置，使用预配置的开发容器。DevContainer 自动安装所有工具链、构建项目并设置 Python 环境。

提供面向 vLLM、SGLang 与 TensorRT-LLM 的特定容器。配置说明见 [DevContainer README](https://github.com/ai-dynamo/dynamo/tree/main/.devcontainer)。

## 配置 Pre-commit Hooks

在提交 PR 之前，安装 pre-commit hooks 以确保代码通过 CI 检查：

```bash
uv pip install pre-commit
pre-commit install
```

手动对所有文件运行检查：

```bash
pre-commit run --all-files
```

## 故障排查

**缺少系统包**

如果 `maturin develop` 因链接错误失败，请验证所有系统依赖均已安装。在 Ubuntu 上：

```bash
sudo apt install -y build-essential libhwloc-dev libudev-dev pkg-config libclang-dev protobuf-compiler python3-dev cmake
```

**虚拟环境未激活**

Maturin 针对当前激活的 Python 解释器构建。如果出现关于 Python 或 site-packages 的错误，请确保已激活虚拟环境：

```bash
source .venv/bin/activate
```

**磁盘空间**

Rust `target/` 目录在开发期间可能增长到 10+ GB。如果构建因磁盘空间不足失败，清理构建缓存：

```bash
cargo clean
```

## 后续步骤

- [贡献指南](../contribution-guide.md) -- 贡献代码的流程
- [示例](https://github.com/ai-dynamo/dynamo/tree/main/examples) -- 浏览代码库
- [Good First Issues](https://github.com/ai-dynamo/dynamo/labels/good-first-issue) -- 寻找可参与的任务
