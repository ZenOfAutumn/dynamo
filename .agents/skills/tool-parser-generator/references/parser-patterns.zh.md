# Tool Call Parser 模式参考

LLM chat 模板中常见的 tool call 模式速查。

## 模式分类

### 1. 带特殊 token 的 JSON

#### 方括号标记（Mistral 风格）
```
[TOOL_CALLS] [{"name": "get_weather", "arguments": {"location": "NYC"}}]
```
- 模型：Mistral、Mixtral
- Parser：`base_json_parser` + 方括号配置
- 关键字段：`name`、`arguments`

#### XML 风格标签（Hermes 风格）
```xml
<tool_call>
{"name": "get_weather", "arguments": {"location": "NYC"}}
</tool_call>
```
- 模型：Hermes-2、Jamba
- Parser：`base_json_parser` + XML 风格标记
- 关键字段：`name`、`arguments`

#### 单 token 前缀（Llama 风格）
```
<|python_tag|>[{"name": "get_weather", "arguments": {"location": "NYC"}}]
```
- 模型：Llama 3.1、Llama 3.2
- Parser：`base_json_parser` + 单一起始 token
- 关键字段：`name`、`arguments`

### 2. 基于 XML

#### Qwen3 Coder 风格
```xml
<tool_call>
<function=get_weather>
<parameter=location>NYC</parameter>
</function>
</tool_call>
```
- 模型：Qwen3-Coder、Nemotron-Nano
- Parser：`xml/parser.rs`
- 基于属性的名称与参数

### 3. 嵌套特殊 token

#### DeepSeek V3
```
<｜tool▁call▁begin｜>function<｜tool▁sep｜>get_weather
```json
{"location": "NYC"}
```
<｜tool▁call▁end｜>
```
- 模型：DeepSeek-V3
- Parser：`deepseek_v3_parser.rs`
- 多行 + markdown 代码块

#### DeepSeek V3.1
```
<｜tool▁call▁begin｜>get_weather<｜tool▁sep｜>{"location": "NYC"}<｜tool▁call▁end｜>
```
- 模型：DeepSeek-V3.1
- Parser：`deepseek_v3_1_parser.rs`
- 内联 JSON

### 4. DSML（DeepSeek V3.2）
```xml
<｜DSML｜function_calls>
<｜DSML｜invoke name="get_weather">
<｜DSML｜parameter name="location" string="true">NYC</｜DSML｜parameter>
</｜DSML｜invoke>
</｜DSML｜function_calls>
```
- 模型：DeepSeek-V3.2
- Parser：`dsml/parser.rs`
- 显式参数类型

### 5. Pythonic
```python
[get_weather(location="NYC"), get_time(timezone="EST")]
```
- 模型：自定义/实验性
- Parser：`pythonic/pythonic_parser.rs`
- Python 函数调用语法

### 6. Harmony
```
<|channel|>commentary to=functions.get_weather
<|constrain|>json
<|message|>{"location": "NYC"}
```
- 模型：GPT-OSS
- Parser：`harmony/harmony_parser.rs`
- OpenAI Harmony 协议

## 快速识别指南

1. **看到 `tojson` filter** → JSON 格式
2. **看到 `<function=` 或 `<parameter=`** → XML 格式
3. **看到 `<｜DSML｜`** → DSML 格式
4. **看到 `function(arg=val)`** → Pythonic 格式
5. **看到 `<|channel|>commentary`** → Harmony 格式
6. **检查起止标记** → 匹配到对应的配置预设

## 配置键

对于 JSON 格式，关注模板中的以下键：
- 函数名：通常是 `name` 或 `function`
- 参数：通常是 `arguments` 或 `parameters`
- 结构：数组 `[{...}]` 或单对象 `{...}`

## 匹配逻辑

1. **完全匹配** → 使用已有的配置预设
2. **标记相近** → 用相同的 parser 创建新配置
3. **新格式** → 生成新的 parser 实现
