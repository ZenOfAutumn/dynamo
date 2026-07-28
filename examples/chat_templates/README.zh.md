# 聊天模板（Chat Templates）

为某些模型内置的 HuggingFace 聊天模板未输出 Dynamo 的工具调用（tool-call）/推理（reasoning）解析器所需标记的情况，提供配套的 Jinja 聊天模板。

在启动服务时，可通过以下参数引用其中任一模板：

```bash
--custom-jinja-template examples/chat_templates/<file>.jinja
```

## 模板列表

| 文件 | 模型 | 配套使用 |
|---|---|---|
| `gemma4_tool.jinja` | Google Gemma 4 思维（thinking）模型 | `--dyn-tool-call-parser gemma4 --dyn-reasoning-parser gemma4` |

## `gemma4_tool.jinja`

输出 Gemma 4 自定义语法：

- 工具调用：`<|tool_call>call:name{key:<|"|>value<|"|>}<tool_call|>`
- 推理（reasoning）：`<|channel>thought\n...<channel|>`
- 工具定义：嵌套的 `declaration:fn{description:<|"|>...<|"|>,parameters:{...}}` 块

该模板逐字复制自上游 vLLM 项目的 `examples/tool_chat_template_gemma4.jinja`（Apache-2.0 许可）。当 Gemma 4 的 HF 聊天模板缺少解析器所要求的 `<|"|>` 分隔工具定义编码时，必须使用此模板。
