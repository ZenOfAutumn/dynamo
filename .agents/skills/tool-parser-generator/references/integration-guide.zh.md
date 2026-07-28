# Parser 集成指南

将新的 parser 或配置集成到 dynamo 的逐步指南。

## 选项 1：添加配置预设（最常见）

当已有的 parser 通过不同的配置就能处理新模型时使用。

### 步骤 1：添加 Config 预设

编辑 `/lib/parsers/src/tool_calling/config.rs`：

```rust
impl ToolCallConfig {
    /// Configuration for ModelName
    pub fn model_name() -> Self {
        Self {
            config: ParserConfig::Json(JsonParserConfig {
                start_token: Some("<start>".to_string()),
                end_token: Some("</end>".to_string()),
                function_name_key: Some("name".to_string()),
                function_arguments_key: Some("arguments".to_string()),
                parser_type: JsonParserType::Basic,
            }),
        }
    }
}
```

### 步骤 2：在 Parser Map 中注册

编辑 `/lib/parsers/src/tool_calling/parsers.rs`：

在 `get_tool_parser_map()` 函数中：

```rust
map.insert("model_name".to_string(), ParserType::Json);
```

### 步骤 3：添加测试

可写在同一文件或 `tests.rs` 中：

```rust
#[test]
fn test_model_name_parser() {
    let config = ToolCallConfig::model_name();
    let message = r#"<start>{"name": "test", "arguments": {}}</start>"#;

    let (calls, _) = detect_and_parse_tool_call(message, Some("model_name"), None).unwrap();

    assert_eq!(calls.len(), 1);
    assert_eq!(calls[0].function.name, "test");
}
```

### 步骤 4：运行测试

```bash
cd lib/parsers
cargo test model_name
```

## 选项 2：创建新 Parser

当格式确实独特，与已有 parser 不匹配时使用。

### 步骤 1：选择目录

- JSON 变种 → `json/`
- XML 变种 → `xml/`
- 全新格式 → 新建子目录

### 步骤 2：创建 Parser 文件

例如：`json/model_name_parser.rs`

```rust
// SPDX-FileCopyrightText: Copyright (c) 2024-2026 NVIDIA CORPORATION & AFFILIATES.
// SPDX-License-Identifier: Apache-2.0

use anyhow::Result;
use regex::Regex;
use std::sync::OnceLock;

use crate::tool_calling::{
    config::{JsonParserConfig, ToolDefinition},
    response::{CalledFunction, ToolCallResponse, ToolCallType},
};

static DETECT_REGEX: OnceLock<Regex> = OnceLock::new();

pub fn detect_tool_call_start_model_name(
    chunk: &str,
    config: &JsonParserConfig,
) -> bool {
    // Implementation
    todo!()
}

pub fn try_tool_call_parse_model_name(
    message: &str,
    config: &JsonParserConfig,
    tools: Option<&[ToolDefinition]>,
) -> Result<(Vec<ToolCallResponse>, Option<String>)> {
    // Implementation
    todo!()
}

pub fn find_tool_call_end_position_model_name(
    chunk: &str,
    config: &JsonParserConfig,
) -> usize {
    // Implementation
    todo!()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_detection() {
        // Tests
    }
}
```

### 步骤 3：更新 mod.rs

在父目录的 `mod.rs` 中：

```rust
pub mod model_name_parser;

pub use model_name_parser::*;
```

### 步骤 4：添加 Config 枚举变体

如果需要，在 `config.rs` 中添加你的 parser 类型：

```rust
pub enum ParserConfig {
    Json(JsonParserConfig),
    Xml(XmlParserConfig),
    // ... existing variants
    ModelName(ModelNameConfig), // If you need custom config
}
```

### 步骤 5：在 parsers.rs 中注册

在 `try_tool_call_parse()` 中添加路由分支：

```rust
match config {
    // ... existing matches
    ParserConfig::ModelName(cfg) => {
        model_name_parser::try_tool_call_parse_model_name(message, cfg, tools)
    }
}
```

### 步骤 6：运行所有测试

```bash
cd lib/parsers
cargo test tool_calling
cargo clippy
cargo fmt
```

## 文件结构

```
lib/parsers/src/tool_calling/
├── mod.rs                   # Main module exports
├── config.rs                # Configurations and presets
├── parsers.rs               # Parser routing and registry
├── response.rs              # Response types
├── tools.rs                 # High-level APIs
├── tests.rs                 # Integration tests
│
├── json/                    # JSON parsers
│   ├── mod.rs
│   ├── base_json_parser.rs
│   ├── deepseek_v3_parser.rs
│   ├── deepseek_v3_1_parser.rs
│   └── model_name_parser.rs  # Your new parser
│
├── xml/                     # XML parsers
│   ├── mod.rs
│   └── parser.rs
│
├── pythonic/                # Pythonic parsers
├── harmony/                 # Harmony parsers
└── dsml/                    # DSML parsers
```

## 测试策略

### 单元测试
对 parser 文件中的各个函数进行测试：
- `detect_tool_call_start_*`
- `try_tool_call_parse_*`
- `find_tool_call_end_position_*`

### 集成测试
通过 `tests.rs` 中的主 API 进行测试：
- 带 parser 名称的 `detect_and_parse_tool_call()`
- 流式行为
- 工具校验

### 真实输出测试
尽可能使用真实模型输出：
- 从模型获取真实的 tool call
- 验证解析得到的结构正确
- 检查正常文本提取

## 常见问题

### 问题：Parser 找不到
**解决方案**：检查在 `get_tool_parser_map()` 中的注册

### 问题：检测能用但解析失败
**解决方案**：检查正则表达式与 JSON 结构的键

### 问题：流式产生错误结果
**解决方案**：核对 `find_tool_call_end_position_*` 的实现

### 问题：测试因 "unknown tool" 失败
**解决方案**：要么提供 tools 列表，要么去掉校验

## 文档

为你的 parser 添加文档注释：

```rust
//! ModelName tool call parser
//!
//! Format: <start>JSON</start>
//!
//! Example:
//! ```text
//! <start>{"name": "func", "arguments": {}}</start>
//! ```
//!
//! Models: ModelName family
```

## 检查清单

- [ ] Parser 实现了三个必需函数
- [ ] 添加了 config 预设（如果使用已有 parser）
- [ ] Parser 已在 map 中注册
- [ ] 单元测试通过
- [ ] 集成测试通过
- [ ] 添加了文档
- [ ] 处理了 Clippy 警告
- [ ] 用 rustfmt 格式化了代码
