---
name: tool-parser-generator
description: Generate optimized tool call parsers for dynamo from HuggingFace model chat templates. Use this when you need to add support for a new model's tool calling format. Takes a HuggingFace model name, analyzes its chat template, compares with existing parsers, and either maps to existing parser or generates new Rust code with tests for the dynamo tool_calling library.
license: "Apache-2.0"
---

# Tool Parser Generator Skill

通过分析模型的 chat 模板并生成相应的 parser 实现，为新模型的 tool 调用格式增加 Dynamo 支持。

## 何时使用本 Skill

- 用户要求为某个具体的 HuggingFace 模型增加 tool calling 支持
- 用户想了解某个模型的 tool call 是如何组织的
- 用户需要用新格式扩展 Dynamo 的 parser 库

## 工作流

当用户提供一个 HuggingFace 模型名时，遵循以下系统化流程。

### 阶段 1：拉取并提取 chat 模板

1. **从 HuggingFace Hub 拉取 tokenizer 配置**：
   ```
   URL: https://huggingface.co/{model_id}/resolve/main/tokenizer_config.json
   ```

2. **提取 chat 模板**：
   - 解析 JSON 响应
   - 查找 `chat_template` 字段
   - 处理两种格式：
     - 字符串：单一模板
     - 数组：包含 `name` 和 `template` 字段的模板列表
       - 优先选用 `tool_use` 模板（如有）
       - 否则回退到 `default` 模板

3. **提取特殊 token**（如相关）：
   - `bos_token`、`eos_token`、`unk_token`
   - `additional_special_tokens`
   - 配置中任何与工具相关的 token

### 阶段 2：分析 chat 模板

chat 模板是 Jinja 模板。分析它以识别 tool call 模式：

1. **找到与工具相关的部分**：
   - 关注包含关键字的条件块：`tools`、`tool_call`、`function`、`available_tools`
   - 提取 `{% if tools %}...{% endif %}` 块内的内容
   - 找到 `{% for tool in tools %}` 循环

2. **识别标记与格式**：
   - **起始标记**：tool call 之前的 token/字符串
     - 示例：`<tool_call>`、`[TOOL_CALLS]`、`<|python_tag|>`、`<｜tool▁call▁begin｜>`
   - **结束标记**：tool call 之后的 token/字符串
     - 示例：`</tool_call>`、`[/TOOL_CALLS]`、`<｜tool▁call▁end｜>`
   - **特殊 token**：Unicode 或编码 token（DeepSeek、Harmony）
   - **格式类型**：
     - JSON：查找 `tojson` filter、`{` `}` 括号
     - XML：查找 `<function=`、`<parameter=` 模式
     - Pythonic：查找 `function(arg=val)` 模式
     - DSML：查找 `<｜DSML｜` token

3. **识别 JSON 结构**（若为 JSON 格式）：
   - name 键：通常是 `name` 或 `function`
   - arguments 键：通常是 `arguments` 或 `parameters`
   - 数组 vs 单对象
   - 多 call 的处理

### 阶段 3：与现有 parser 对比

**阅读 `/lib/parsers/src/tool_calling/` 中现有的 parser 实现**：

1. **检查 JSON parser**（`json/` 目录）：
   - `base_json_parser.rs` - 通用的带标记 JSON
   - `deepseek_v3_parser.rs` - DeepSeek V3 格式
   - `deepseek_v3_1_parser.rs` - DeepSeek V3.1 格式

2. **检查 XML parser**（`xml/` 目录）：
   - `parser.rs` - Qwen3 Coder XML 格式

3. **检查其他格式**：
   - `pythonic/pythonic_parser.rs` - Python 语法
   - `harmony/harmony_parser.rs` - Harmony 协议
   - `dsml/parser.rs` - DeepSeek V3.2 DSML

4. **回顾 `config.rs` 中的配置预设**：
   - 查看 `ToolCallConfig::hermes()`、`mistral()`、`llama3_json()` 等
   - 每个预设定义了起止 token、键名、parser 类型

5. **查看 `parsers.rs` 中的 parser 注册表**：
   - 看看 parser 是如何在 `get_tool_parser_map()` 中注册的
   - 理解 `ParserType` 枚举与路由逻辑

**与分析得到的格式做匹配**：
- 起止 token 与格式与现有 parser 匹配 → 用现有 parser + 配置
- 类似但 token 不同 → 调整现有 parser 配置
- 完全不同的格式 → 生成新 parser

### 阶段 4：生成或配置 parser

#### 选项 A：使用现有 parser（首选）

如果找到匹配，则创建一个配置预设：

1. 在 `/lib/parsers/src/tool_calling/config.rs` 中新增一个预设函数：
   ```rust
   impl ToolCallConfig {
       pub fn new_model_name() -> Self {
           Self {
               config: ParserConfig::Json(JsonParserConfig {
                   start_token: Some("<marker>".to_string()),
                   end_token: Some("</marker>".to_string()),
                   function_name_key: Some("name".to_string()),
                   function_arguments_key: Some("arguments".to_string()),
                   parser_type: JsonParserType::Basic,
               }),
           }
       }
   }
   ```

2. 在 `/lib/parsers/src/tool_calling/parsers.rs` 的 parser map 中注册

3. **编写测试**以验证配置可工作

#### 选项 B：生成新 parser（如有必要）

如果没有现成的 parser 合适，则生成新 parser 代码：

1. **根据格式选 parser 模板**：
   - JSON 格式 → 以 `base_json_parser.rs` 为模板
   - XML 格式 → 以 `xml/parser.rs` 为模板
   - 自定义格式 → 实现三个核心函数

2. **实现必备函数**：
   ```rust
   // Detection
   pub fn detect_tool_call_start_<name>(chunk: &str, config: &Config) -> bool

   // Parsing
   pub fn try_tool_call_parse_<name>(
       message: &str,
       config: &Config,
       tools: Option<&[ToolDefinition]>,
   ) -> Result<(Vec<ToolCallResponse>, Option<String>)>

   // End detection (for streaming)
   pub fn find_tool_call_end_position_<name>(chunk: &str, config: &Config) -> usize
   ```

3. **使用正则表达式做 token 匹配**：
   - 使用 `OnceLock<Regex>` 缓存编译后的正则
   - 正确转义特殊字符
   - 处理流式中的部分 token

4. **解析 JSON/XML 内容**：
   - 用 `serde_json` 解析 JSON
   - 用正则提取 XML（复杂时再用 XML parser）
   - 构造 `ToolCallResponse` 结构体

5. **放到合适的目录**：
   - JSON 变体 → `json/` 目录
   - XML 变体 → `xml/` 目录
   - 全新格式 → 新建子目录

### 阶段 5：生成测试

为任意新 parser 或配置生成完整的测试：

1. **基本测试**：
   - 起始标记的检测
   - 解析单个 tool call
   - 解析多个 tool call
   - 普通文本提取

2. **边缘情况**：
   - 空参数
   - 缺失字段
   - 错误的 JSON/XML
   - 部分 token（流式）

3. **集成测试**：
   - 用真实模型输出做端到端测试（如有）
   - 工具校验（如提供 tools 列表）

4. **把测试加到合适位置**：
   - parser 文件内联（在 `#[cfg(test)]` 模块）
   - 或在 `/lib/parsers/src/tool_calling/tests.rs`

### 阶段 6：集成

1. **更新模块导出**：
   - 在父级 `mod.rs` 中加入 `mod` 声明
   - 按需导出函数

2. **若是新 parser，在 `parsers.rs` 中注册**：
   - 加入 `get_tool_parser_map()` 函数
   - **关键**：更新 `test_get_available_tool_parsers()` 测试
   - 把新 parser 名加到测试中的 `available_parsers` 数组

3. **为 parser 写文档**：
   - 加上说明格式的 doc 注释
   - 包含输入/输出示例
   - 引用所属模型家族

4. **跑测试**：
   ```bash
   cd lib/parsers
   cargo test tool_calling
   ```

5. **用 Dynamo 验证**：
   - 如可能，用真实模型测试
   - 验证流式行为
   - 检查错误处理

## 关键参考文件

**Dynamo 代码库**：
- `/lib/parsers/src/tool_calling/` - 所有 tool call parser
- `/lib/parsers/src/tool_calling/config.rs` - 配置预设
- `/lib/parsers/src/tool_calling/parsers.rs` - parser 注册表
- `/lib/llm/src/preprocessor/prompt/template/tokcfg.rs` - chat 模板结构
- `/lib/llm/src/preprocessor/prompt/template.rs` - 模板加载

**参考实现**：
- **sglang**：https://github.com/sgl-project/sglang/tree/main/python/sglang/srt/function_call
  - 关注 detector 模式（base_format_detector.py）
  - 模型专用 detector（qwen25_detector.py、deepseekv3_detector.py 等）
- **vLLM**：https://github.com/vllm-project/vllm/tree/main/vllm/tool_parsers
  - 关注 abstract_tool_parser.py
  - 模型专用 parser（llama_tool_parser.py、qwen3xml_tool_parser.py 等）
- **HuggingFace**：https://huggingface.co/docs/transformers/chat_templating

## 示例：为新模型添加支持

用户："为 Qwen/Qwen2.5-72B-Instruct 增加 tool calling 支持"

**步骤 1**：拉取 tokenizer 配置
- 用 WebFetch 获取 `https://huggingface.co/Qwen/Qwen2.5-72B-Instruct/resolve/main/tokenizer_config.json`

**步骤 2**：分析 chat 模板
- 提取 `chat_template` 字段
- 识别 `{% if tools %}` 块
- 找到标记：很可能是 `<tool_call>` 与 `</tool_call>`
- 识别格式：检查是否带 `tojson` filter 的 JSON

**步骤 3**：与现有 parser 对比
- 阅读 `/lib/parsers/src/tool_calling/config.rs`
- 检查 `ToolCallConfig::hermes()` —— 使用 `<tool_call>` 标记
- 看 Qwen 格式是否与 hermes 格式匹配

**步骤 4**：使用或调整现有 parser
- 若与 hermes 匹配：创建 `qwen2_5()` 配置预设
- 若不同：生成新 parser 或调整 base_json_parser

**步骤 5**：生成测试
- 用 Qwen tool call 的样例创建测试用例
- 测试检测、解析与边缘情况

**步骤 6**：集成
- 在 `config.rs` 中加入配置预设
- 在 parser map（`get_tool_parser_map()`）中注册
- 更新 `test_get_available_tool_parsers()` 测试
- 跑测试
- 写文档

## 提示

- **总是优先使用现有 parser**：大多数模型用现有 parser + 不同配置即可
- **阅读参考实现**：sglang 与 vLLM 通常已有热门模型的 parser
- **HF 模型用 WebFetch**：不要靠假设——一定要拉取真实的 tokenizer 配置
- **用真实输出测试**：可能的话，拿真实模型输出做对比
- **保持简单**：能用直接的正则就别上复杂解析
- **写好文档**：未来的你（或他人）会感谢你

## 常见模式

### 带方括号的 JSON
```
[TOOL_CALLS] [{"name": "func", "arguments": {}}]
```
→ 用带方括号标记的 `base_json_parser`

### 带 XML 标签的 JSON
```xml
<tool_call>
{"name": "func", "arguments": {}}
</tool_call>
```
→ 用带 XML 风格标记的 `base_json_parser`

### XML 结构
```xml
<tool_call>
<function=name>
<parameter=key>value</parameter>
</function>
</tool_call>
```
→ 用 `xml/parser.rs` 或创建变体

### 嵌套 token
```
<｜tool▁call▁begin｜>name<｜tool▁sep｜>args<｜tool▁call▁end｜>
```
→ 创建专用 parser（参考 DeepSeek 系列 parser）

## 最小改动哲学

1. **首先**：用新配置尝试现有 parser
2. **其次**：对现有 parser 做小改造
3. **最后**：才完全新建 parser

大多数模型（>80%）都能用合适配置的现有 parser。
