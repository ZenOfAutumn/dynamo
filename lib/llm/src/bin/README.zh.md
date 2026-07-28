# 可执行工具（bin）

> 存放 `dynamo-llm` crate 的辅助命令行可执行程序；当前只有一个：离线生成 HTTP 前端的 OpenAPI 规范文档。

## 这个模块解决什么问题

在 Rust 里，一个 crate 除了产出库（lib），还可以产出多个可执行程序（binary）。`src/bin/` 下的每个 `.rs` 文件就是一个独立的 binary 入口（带自己的 `fn main`），类似 Java 里一个工程里多个带 `public static void main` 的工具类。

本目录的程序解决的是工具链问题：CI 和文档工具需要拿到与前端 `/openapi.json` 端点**完全一致**的 OpenAPI 文档，但又不想真的启动 HTTP 服务再去抓取。于是提供一个离线生成器，直接构建路由图、导出规范并写到 `docs/frontends/openapi.json`。

## 目录结构

| 文件 | 职责 |
|---|---|
| `generate_frontend_openapi.rs` | binary `generate-frontend-openapi` 的入口：构建 `HttpService`（不监听网络），导出 OpenAPI 文档并写入 `docs/frontends/openapi.json` |

> 注：该 binary 在 `lib/llm/Cargo.toml` 中通过 `[[bin]] name = "generate-frontend-openapi"` 声明。

## 核心概念（Java 视角）

- **`fn main() -> anyhow::Result<()>`**：程序入口。返回 `Result` ≈ `main` 方法可以"抛受检异常"，出错时进程以非零码退出。`anyhow::Result` 是一个通用错误类型，类似把各种异常统一收敛成一个 `Exception`。

- **`.context(...)` / `?`**：`?` 相当于"自动 throw"——出错就向上返回；`.context("...")` 给错误附加可读上下文，类似在抛异常时用 `new RuntimeException("...", cause)` 包一层说明。

- **用大栈线程跑生成逻辑**：代码先 `thread::Builder::new().stack_size(8MB).spawn(generate_openapi)`，再 `join`。因为 utoipa 对深度嵌套的 OpenAI 类型做 schema 推导是递归的，默认栈可能溢出。这等价于 Java 里 `new Thread(runnable)` 时显式指定更大的栈大小，再 `thread.join()` 等它跑完。

- **builder 模式**：`HttpService::builder().enable_chat_endpoints(true)....build()` 就是典型的 Java 链式 Builder——逐项打开各类端点（chat / cmpl / embeddings / responses / anthropic），最终 `build()` 出对象。这里只构建路由图与文档，不启动任何网络监听。

## 与其他模块的关系

- **依赖**：使用 `dynamo_llm::http::service` 的 `HttpService`（builder + `route_docs`）和 `openapi_docs::generate_openapi_spec`。也就是说它把本 crate 当作库来调用——这是 bin 与 lib 协作的典型方式。
- **输出**：写文件到仓库相对路径 `docs/frontends/openapi.json`，因此通常从仓库根目录运行。
- 不被其他源码模块 import，是叶子节点式的独立工具。

## 阅读建议

1. 直接读 `generate_frontend_openapi.rs`，从 `fn main` 看起，理解"为什么要起一个 8MB 栈的线程"。
2. 再看 `generate_openapi` 函数：`HttpService::builder()...build()` 如何拼出全部前端端点，以及 `route_docs()` → `generate_openapi_spec()` → 写文件的流程。
3. 想跑一下，可在仓库根目录执行：`cargo run -p dynamo-llm --bin generate-frontend-openapi`。
