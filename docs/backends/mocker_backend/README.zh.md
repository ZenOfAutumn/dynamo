---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Mocker Backend (Rust)
---

# Mocker Backend (Rust)

Dynamo 的参考 Rust 后端。它将 `dynamo-mocker` 调度器封装到
[`dynamo-backend-common`](../../../lib/backend-common/) 中的
`LLMEngine` 契约中，提供高保真模拟引擎——一个共享的前向传播循环
为所有 in-flight 请求设定节奏（连续 batching），而不是为每个请求
独立计时。可以将其作为：编写自己 Rust 后端的模板、AIPerf / 端到端
流水线测试中的替身引擎，以及让 mocker 调度器始终绑定到
`LLMEngine` 契约的强制函数。

`LLMEngine` trait 的权威契约请参阅
[`engine.rs`](../../../lib/backend-common/src/engine.rs) 中的文档注释。

## 快速演示（docker compose）

一条命令即可启动 NATS、etcd、Dynamo 前端与 mocker 后端——
全部从仓库源码构建：

```bash
cd lib/backend-common/examples/mocker
docker compose up --build
```

首次 `up` 较慢——它会构建 Rust 镜像并从 HuggingFace 下载 Qwen3
tokenizer。后续运行会复用 Docker 的层缓存以及 HF 缓存的命名卷。

在另一个终端中发送一次 chat completion：

```bash
curl http://localhost:8000/v1/chat/completions \
    -H 'Content-Type: application/json' \
    -d '{
          "model": "mocker-model",
          "messages": [{"role": "user", "content": "hello"}],
          "max_tokens": 32
        }'
```

响应携带通过 Qwen 词表去 tokenize 的随机 token ID——这并非有意义
的文本，但证明了流水线的每个阶段都已端到端连通。

使用 `docker compose down` 拆除（加 `-v` 删除 HF 缓存卷）。

如果遇到 HuggingFace 限流，请在 shell 中设置 `HF_TOKEN`。

## 本地构建与运行

```bash
cargo build -p dynamo-mocker-backend --release

# For chat/completions endpoints the frontend needs a tokenizer + chat
# template, so point --model-path at an open HF repo. For tensor/prefill
# endpoints (no tokenization), omit --model-path for name-only mode.
./target/release/dynamo-mocker-backend --model-path Qwen/Qwen3-0.6B
```

需要可通过 `NATS_SERVER` / `ETCD_ENDPOINTS` 环境变量访问到的、与 Dynamo 一同
运行的基础设施服务（NATS、etcd）。

## 编写自己的 Rust 后端

1. 新建依赖 `dynamo-backend-common` 的 crate；放在 `lib/` 下。
2. 实现
   [`LLMEngine`](../../../lib/backend-common/src/engine.rs)
   以及一个固有方法
   `from_args(argv) -> Result<(Self, WorkerConfig), DynamoError>`。
3. 镜像 mocker 示例的三行 `main.rs`。
4. 在测试中运行一致性套件：

   ```toml
   [dev-dependencies]
   dynamo-backend-common = { workspace = true, features = ["testing"] }
   ```

   ```rust
   #[tokio::test]
   async fn my_engine_satisfies_contract() {
       let engine = MyEngine::new_for_test();
       dynamo_backend_common::testing::run_conformance(engine)
           .await
           .expect("conformance");
   }
   ```

## 目录布局

```text
lib/backend-common/examples/mocker/
├── Cargo.toml
├── Dockerfile              # builds the mocker backend binary
├── Dockerfile.frontend     # builds the Dynamo frontend from source
├── docker-compose.yml      # one-command infra + frontend + backend
└── src/
    ├── main.rs             # 3-line entry point
    └── engine.rs           # MockerBackend wrapping dynamo-mocker scheduler
```

## 参考

- Crate：[`lib/backend-common/`](../../../lib/backend-common/)
- 示例源：[`lib/backend-common/examples/mocker/`](../../../lib/backend-common/examples/mocker/)
- 一致性套件：[`lib/backend-common/src/testing.rs`](../../../lib/backend-common/src/testing.rs)
