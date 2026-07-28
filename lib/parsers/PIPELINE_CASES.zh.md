# 流水线边界用例

用于固定**解析器（parser）与流水线其余部分之间边界**契约的测试参考分类——
即那些既不属于解析器内部实现、也不属于请求时门控（request-time gating）的输出格式。

同级文件：

- **工具调用解析器**：`PARSER_CASES.md`
- **推理（reasoning）解析器**：`REASONING_CASES.md`
- **前端门控**：
  `components/src/dynamo/frontend/tests/FRONTEND_CASES.md`

## 快速参考

- **`PIPELINE.finish_reason`** —— 解析器的输出与上游的
  `finish_reason`（`stop` / `tool_calls` / `length`）相互独立。
  即使引擎报告的流结束原因不同，相同的输入文本也必须产生相同的解析结果。

## `PIPELINE.finish_reason` —— 输出与上游流结束原因相互独立

当引擎因为出现工具调用而报告 `finish_reason=tool_calls` 时，
解析器不能"信任"该信号——它必须完全基于文本内容来抽取调用。
反过来，当引擎报告 `finish_reason=length`（被截断）时，
解析器仍必须恢复在截断点之前已完整落地的所有调用
（另见 `PARSER.batch.5`）。

解析器的职责是把 `text → (calls, normal_text)` 做映射；
而把 `(calls, finish_reason)` 映射成响应线协议格式
（如 `finish_reason: tool_calls` 等）则是**前端**的职责，不是解析器的。

- 适用于每一个工具调用解析器。
- 测试约定：用相同的文本喂入两次，但带上不同的上游
  `finish_reason` 值；断言两次解析器输出按字节完全一致。
- 配套的前端断言（关于在出现调用时*向外传播*
  `finish_reason=tool_calls` 那一半）位于
  `FRONTEND_CASES.md`。
