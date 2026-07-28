# Reasoning Parser 边界案例

`src/reasoning/` 下 **reasoning** 解析器（Granite、GPT-OSS、Gemma、
Qwen3 think-tag、Minimax、DeepSeek V3 think-tag 等）单元测试的
参考分类法。相邻阶段的同类文档：

- **工具调用解析器（Tool-call parsers）**（`src/tool_calling/`）：见 `PARSER_CASES.md`。
- **前端门控（Frontend gating）**：见
  `components/src/dynamo/frontend/tests/FRONTEND_CASES.md`。
- **流水线边界（Pipeline boundary）**：见 `PIPELINE_CASES.md`。

reasoning 分类法在 阶段 / 模式 / 格式 三个轴上与 parser 分类法保持一致：

- **阶段** — `REASONING.*`（本文件）。
- **模式** — `batch`（整个模型输出作为单一字符串）或 `stream`
  （增量的 `delta_text`）。
- **格式** — `PARSER_CASES.md` 中的 `PARSER.fmt|xml|harmony.*` 标签
  当 reasoning 解析器消费对应格式时也会附加到 reasoning 测试上。
  格式标签描述的是 *语法*，而非解析阶段。

每个 `#[test]` 都带有一个或多个 `// REASONING.<tag>` 注解，
并在适用时附带格式标签。`N/A` 会被显式标出。

## 速查表

### Reasoning，batch 模式

编号与 `PARSER.batch.{N}` 对应，因此 "PARSER.batch.X 在工具调用侧
所测试的形态，与 REASONING.batch.X 在 reasoning 侧测试的形态相同"。
某些编号在 reasoning 侧并不适用（reasoning 块没有"空 args"的对应物），
会被显式标记为 `N/A`，而不是悄无声息地缺失。

- **`REASONING.batch.1`** 单个 reasoning 块，理想路径——存在 `<think>...</think>`（或类似形式），无工具调用。解析器填充 `reasoning_text`，`tool_calls` 留空。
- **`REASONING.batch.2`** reasoning + 紧随其后的工具调用——模型先发出 `<think>...</think>`，随后是工具调用 token。reasoning 解析器必须提取 think 内容，**同时把工具调用标记原样保留在 `normal_text` 中**，供下游工具调用解析器消费。需小心 "未闭合 think-tag 吞掉工具调用" 这一 bug。
- **`REASONING.batch.3`** reasoning + 普通文本（无工具调用）——`<think>...</think>` 与对用户可见的叙述交错出现。reasoning 内容进入 `reasoning_text`；叙述进入 `normal_text`。
- **`REASONING.batch.4`** 格式异常的 reasoning 内容——孤立的结束标记（无开始的闭合）、有开始无结束（截断），或 reasoning 块内部存在非法语法。行为由实现决定：可优雅回退到 `normal_text`、部分提取或显式报错均可。
- **`REASONING.batch.5`** 缺失结束标记的恢复——reasoning 侧对应 `PARSER.batch.5`。引擎在 think 中途触发 `max_tokens`；明确行为（恢复部分 reasoning、暴露为截断、或全部当作 `normal_text`）。
- **`REASONING.batch.6`** 空 reasoning 内容——`<think></think>`（或两个标记之间零字节的等价形式）。仍必须登记为一段 reasoning，而不能被静默丢弃。
- **`REASONING.batch.7`** 复杂 reasoning 内容——大块、多段落、特殊字符、Unicode、换行符。验证内容能完整通过，没有截断或转义 bug。
- **`REASONING.batch.8`** 空/null 内容变体——空输入、仅空白输入、上游 chunk 中的 null。解析器必须不崩溃，且产生一致输出。
- **`REASONING.batch.9`** 单次响应中的多个 reasoning 段——例如两个 `<think>...</think>` 块连续出现。行为由实现决定：拼接、只暴露第一个，或作为列表暴露。

### Reasoning，stream 模式

- **`REASONING.stream.1`** 跨 N 个 chunk 的单个 reasoning 块——在任意大小的多个 chunk 上增量组装 `<think>...</think>` 内容。
- **`REASONING.stream.2`** 起始标记跨 chunk 切分——开始标记 `<think>`（或类似形式）跨越 chunk 边界；部分 token 匹配必须继续缓冲，而不能把已到达的字节当作普通文本刷出。
- **`REASONING.stream.3`** 结束标记跨 chunk 切分——闭合标记 `</think>` 跨越 chunk 边界；与 `REASONING.stream.2` 同样的缓冲约定。
- **`REASONING.stream.4`** 累积文本发散——累积的文本始终未匹配上起始标记；解析器必须干净地放弃，把全部输入当作 `normal_text`。

### 跨阶段的格式条件标签

`PARSER_CASES.md` 中的 `PARSER.fmt|xml|harmony.*` 标签同样适用于
消费对应格式的 reasoning 解析器。例如，GPT-OSS reasoning 解析器
处理 Harmony 的 `<|channel|>analysis<|message|>...<|end|>` 信封，
其测试会同时携带 `REASONING.batch.1` 与 `PARSER.harmony.1`。

### reasoning 解析器中 N/A 的分类

- `PARSER.batch.2` —— "多个工具调用" 对应到 `REASONING.batch.9`
  （多个 reasoning 段），并非直接对应。工具调用形态的标签不适用。
- `PARSER.batch.8` —— 工具调用标记与普通文本交错；
  reasoning 的对应物是 `REASONING.batch.3`（reasoning + 普通文本）。
- `FRONTEND.tool_choice` —— 请求时门控，见
  `FRONTEND_CASES.md`。
- `PIPELINE.finish_reason` —— 流水线边界契约，见
  `PIPELINE_CASES.md`。

---

## `REASONING.batch.1` — 仅 reasoning 内容

存在 `<think>...</think>`（或类似形式：`<seed:think>`，Harmony 的
`<|channel|>analysis`），无工具调用。

- 适用于每一个 reasoning 解析器。
- 解析器使用标记之间的内容填充 `reasoning_text`；`tool_calls`
  为空；`normal_text` 为空，或承载 reasoning 块之外的文本（参见
  `REASONING.batch.3`）。

## `REASONING.batch.2` — reasoning + 紧随其后的工具调用

模型在 reasoning 内容之后发出工具调用 token：
`<think>...</think><|tool_call_begin|>...`。两者都必须被提取——
`reasoning_text` 被填充，**同时**工具调用标记保留在 `normal_text` 中
以便下游工具调用解析器消费。

- 适用于每一对 (reasoning, tool-call) 解析器。
- 失败模式：贪婪的 reasoning 解析器吞掉随后的工具调用内容。
  必须显式锚定边界。

## `REASONING.batch.3` — reasoning + 普通文本

`<think>...</think>` 在用户可见叙述之后（或之前）出现，且无工具调用。
reasoning 内容 → `reasoning_text`；叙述 → `normal_text`。

- 适用于每一个 reasoning 解析器。

---

## `REASONING.stream.1` — 跨 N 个 chunk 的单个 reasoning 块

一个完整的 reasoning 块通过任意大小的多个 SSE chunk 传送。
解析器增量重建 `reasoning_text`。

- 适用于每一个 reasoning 解析器。生产环境中的主路径。

## `REASONING.stream.2` — 起始标记跨 chunk 切分

reasoning 的开始标记（`<think>`、`<|channel|>analysis` 等）跨越
chunk 边界。部分 token 匹配必须返回 "继续缓冲"，而不是把部分字节
作为 `normal_text` 刷出，且在下一个 chunk 到达时完成匹配。

- 适用于每一个 reasoning 解析器。

## `REASONING.stream.3` — 结束标记跨 chunk 切分

reasoning 的闭合标记（`</think>`、`<|end|>` 等）跨越 chunk 边界。
缓冲约定与 `REASONING.stream.2` 相同。

- 适用于每一个 reasoning 解析器。

## `REASONING.stream.4` — 累积文本发散

累积文本始终未匹配上起始标记（模型从一开始就输出普通文本，
没有 reasoning 块）。解析器必须干净地放弃，把整个流当作
`normal_text`，而不是无限缓冲。

- 适用于每一个 reasoning 解析器。

---

## 客户事件回归测试

约定与 `PARSER_CASES.md` 相同：在 `#[test]` 注释中内联给出原始来源。

```rust
#[test] // REASONING.batch.2 (PR #1234)
fn test_unclosed_think_tag_no_longer_swallows_tool_call() { ... }
```

---

## 添加新 reasoning 解析器：必备清单

最小可行集合：

1. `REASONING.batch.{1, 3}` —— 基线 reasoning 提取 +
   reasoning 与叙述的拆分。
2. `REASONING.batch.2` —— 与下游工具调用解析器之间的边界契约。
   对任何与工具调用配合使用的 reasoning 解析器而言不可妥协。
3. `REASONING.batch.{4, 5}` —— 异常输入 + 缺失结束标记的恢复。
   明确行为；静默丢弃即为失败模式。
4. `REASONING.batch.{6, 7, 8}` —— 空 / 复杂 / null 内容。
5. `REASONING.batch.9` —— 多个 reasoning 段。把契约写下来。
6. `REASONING.stream.{1, 2, 3, 4}` —— 流式。对任何在流式
   前端之后的解析器而言基本不可妥协。
7. 适用时附加格式标签：消费 Harmony 格式输入的解析器配
   `PARSER.harmony.1`，等等。
