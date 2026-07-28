---
# SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Chat Processor Options
subtitle: Choose the right preprocessing pipeline for tool calling, reasoning, and tokenization
---

Dynamo 将工作拆分为一个 **frontend**（前端）进程（HTTP 服务器、tokenization、
路由、解析）和一个或多个 **worker** 进程（运行模型的引擎）。多个 CLI flag
控制由哪条代码路径来处理 chat 模板渲染、tool-call 解析以及 reasoning
内容的分离。本页说明可用的配置组合、何时使用各组合，以及它们与 KV 缓存
（KV cache）路由的交互方式。

各个 parser 的名称列表见
[Tool Calling](tool-calling.md) 和 [Reasoning](reasoning.md)。

## 配置组合

共有五种受支持的配置。每一种都在启动时设定——Dynamo 不会按请求在它们之间切换。

| | 前端 flag | worker flag | KV 路由 | 备注 |
|---|---|---|---|---|
| **A** Dynamo 原生（默认） | `--dyn-chat-processor dynamo` | `--dyn-tool-call-parser <name>` `--dyn-reasoning-parser <name>` | 是 | Rust 预处理器。延迟最低。 |
| **B** vLLM chat processor | `--dyn-chat-processor vllm` `--tool-call-parser <name>` `--reasoning-parser <name>` | *(无)* | 是 | 委托给 vLLM 的 Python 预处理器。 |
| **C** SGLang chat processor | `--dyn-chat-processor sglang` `--tool-call-parser <name>` `--reasoning-parser <name>` | *(无)* | 是 | 委托给 SGLang 的 Python 预处理器。详见 [SGLang Chat Processor](../backends/sglang/sglang-chat-processor.md)。 |
| **D** vLLM tokenizer 委托 | `--router-mode round-robin` | `--use-vllm-tokenizer` | 否 | 引擎侧 tokenization。Day-0 模型回退方案。 |
| **E** SGLang tokenizer 委托 | `--router-mode round-robin` | `--use-sglang-tokenizer` | 否 | **已弃用**——请改用选项 C。 |

> [!NOTE]
> 虽然 `dynamo` 是 `--dyn-chat-processor` 的默认值，但在启动脚本中
> 显式指定它能让选择在日志和支持诊断中可见。

## Flag 参考

### `--dyn-chat-processor {dynamo | vllm | sglang}`

前端 flag（默认 `dynamo`）。选择负责渲染模板、tokenize 并分发解析的 chat processor。

- `dynamo` —— Rust 预处理器。parser 名称来自 Dynamo 的注册表
  （见 [Tool Calling](tool-calling.md) 和 [Reasoning](reasoning.md)）。
- `vllm` —— vLLM 的 Python 预处理器。parser 名称来自 vLLM 的
  注册表，可能与 Dynamo 不同。
- `sglang` —— SGLang 的 Python 预处理器。parser 名称来自 SGLang 的
  注册表。详见 [SGLang Chat Processor](../backends/sglang/sglang-chat-processor.md)。

### `--dyn-tool-call-parser <name>` / `--dyn-reasoning-parser <name>`

worker flag。来自 Dynamo parser 注册表的名称。仅在
`--dyn-chat-processor dynamo`（选项 A）下生效；在其他 chat processor
下会被静默忽略。

这些 flag 声明在 worker 的 CLI 上，但解析器实际运行在前端——
名称通过模型元数据传播。受支持的名称见
[Tool Calling](tool-calling.md) 和 [Reasoning](reasoning.md)。

### `--tool-call-parser <name>` / `--reasoning-parser <name>`

前端 flag（无 `--dyn-` 前缀）。名称来自上游引擎的注册表。
仅当与匹配的 chat processor 搭配时被接受：

- 在 `--dyn-chat-processor vllm` 下：被接受。使用 vLLM parser 名称。
- 在 `--dyn-chat-processor sglang` 下：被接受。使用 SGLang parser 名称。
- 在 `--dyn-chat-processor dynamo` 下：**启动时被拒绝**，提示
  `Unknown arguments specified: ...`。请改用 `--dyn-*` worker flag。

上游 parser 名称与 Dynamo 容器中所打包的引擎版本绑定。对于同一
模型，它们可能与 Dynamo 的名称不同（例如 SGLang 用
`deepseekv3` 而 Dynamo 用 `deepseek_v3`）。

### `--use-vllm-tokenizer` / `--use-sglang-tokenizer`

worker flag（布尔）。把 tokenization 交给引擎处理而非前端。
该 flag 必须与 worker 上的引擎匹配。

`--use-sglang-tokenizer` 已弃用。新的 SGLang 部署应改用
`--dyn-chat-processor sglang`（选项 C）。详见
[Migration from --use-sglang-tokenizer](../backends/sglang/sglang-chat-processor.md#migration-from---use-sglang-tokenizer)。

## 我应该选哪种？

1. **Dynamo 是否有适用于你模型的 parser？** 查阅
   [Tool Calling](tool-calling.md) 和 [Reasoning](reasoning.md)
   中的逐模型表格。如果有，使用
   **选项 A**。这是默认路径：前端 Rust 解析、可 KV 路由、延迟最低。

2. **上游引擎有但 Dynamo 没有 parser？** 使用
   **选项 B**（vLLM）或 **选项 C**（SGLang）。仍然可 KV 路由。

3. **问题出在 tokenizer 本身**（day-0 模型、自定义特殊 token、
   rope 变体）？使用 **选项 D**。KV 路由关闭；搭配
   `--router-mode round-robin`。

4. **SGLang + day-0 模型？** 使用 **选项 C**，并填入合适的上游
   parser 名称。不要使用选项 E（已弃用）。

## 非法或静默错误的组合

### 启动时被拒绝

- **`--dyn-chat-processor dynamo` 与 `--tool-call-parser <name>`**
  （或 `--reasoning-parser`）。无前缀的 flag 在 Dynamo chat processor
  下不被识别。请改在 worker 上使用 `--dyn-tool-call-parser`。

- **同一 SGLang worker 上同时使用 `--tool-call-parser` 与
  `--dyn-tool-call-parser`**。SGLang 拒绝该组合：`Cannot use both
  --tool-call-parser and --dyn-tool-call-parser`。任选其一命名空间。

- **在 SGLang worker 上使用 `--use-vllm-tokenizer`**（反之亦然）。
  flag 必须与引擎匹配。

### 静默错误（无启动错误，但结果不正确）

- **tokenizer 委托 + `--router-mode kv`** —— 选项 D/E 与 `kv`
  路由组合会产生 prefix-hash 不匹配以及静默的缓存未命中。

- **同一 vLLM worker 上 `--dyn-tool-call-parser` + `--use-vllm-tokenizer`**。
  worker 绕过了 Dynamo 的预处理器，但前端侧的 parser 仍在生效，
  会导致 token 流不一致。当前没有互斥检查。

## 路由兼容性

`--router-mode kv` 需要前端 tokenization 来计算 prefix-hash 路由键。
选项 A、B、C 把 tokenizer 留在前端，因此可 KV 路由。选项 D 与 E
把 tokenization 移到了 worker 上，因此 **不能** KV 路由——
应搭配 `round-robin` 或 `random`。

| 选项 | `kv` 路由 | `round-robin` / `random` |
|--------|:---:|:---:|
| A（Dynamo 原生） | 是 | 是 |
| B（vLLM processor） | 是 | 是 |
| C（SGLang processor） | 是 | 是 |
| D（vLLM tokenizer 委托） | **否** | 是 |
| E（SGLang tokenizer 委托） | **否** | 是 |

## 各 flag 存在的原因

- **前端 tokenization** 是 KV cache 路由的前提。前端需要 token id 来
  在请求到达 worker 之前计算 prefix-hash 路由键。Rust 原生路径
  （选项 A）上的 parser flag 因此与前端的 tokenization 共置。

- **后端 tokenization** 是当前端 tokenization 无法或不应运行时的
  回退方案：不受支持的模型、day-0 支持、tokenizer 边角情形
  （自定义特殊 token、rope 变体）。在该模式下 tokenizer 由引擎拥有，
  KV 路由因此失效。

- **Chat-processor 切换**（选项 B/C）是折中方案：tokenization
  仍留在前端（可 KV 路由），但解析委托给上游引擎的 Python 实现。
  适用于 Dynamo 尚未编写对应 Rust 解析器的模型。

## 各模型对应的 parser 名称

完整支持的 parser 名称、覆盖的模型，以及上游名称差异
（与选项 B、C 相关）：

- [Tool Calling](tool-calling.md) —— 受支持的 tool call parser，
  含模型映射与上游名称差异
- [Reasoning](reasoning.md) —— 受支持的 reasoning parser，
  含模型映射与强制 reasoning 行为

## 标准启动示例

```bash
# A -- Dynamo-native (default).
python -m dynamo.vllm \
  --dyn-tool-call-parser kimi_k2 \
  --dyn-reasoning-parser kimi_k25
python -m dynamo.frontend --dyn-chat-processor dynamo

# B -- vLLM chat-processor (upstream parser names on the frontend).
python -m dynamo.vllm ...
python -m dynamo.frontend \
  --dyn-chat-processor vllm \
  --tool-call-parser hermes \
  --reasoning-parser deepseek_r1

# C -- SGLang chat-processor.
python -m dynamo.sglang ...
python -m dynamo.frontend \
  --dyn-chat-processor sglang \
  --tool-call-parser kimi_k2 \
  --reasoning-parser kimi_k25

# D -- vLLM tokenizer delegation (no KV routing).
python -m dynamo.vllm --use-vllm-tokenizer ...
python -m dynamo.frontend --router-mode round-robin
```

## 另请参阅

- [Tool Calling](tool-calling.md) —— 受支持的 tool call parser 名称、请求示例
- [Reasoning](reasoning.md) —— 受支持的 reasoning parser 名称、常用搭配
- [SGLang Chat Processor](../backends/sglang/sglang-chat-processor.md) —— 选项 C 的细节
- [Frontend Configuration Reference](../components/frontend/configuration.md) —— 完整 CLI flag 参考
