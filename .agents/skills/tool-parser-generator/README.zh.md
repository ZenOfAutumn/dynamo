# 工具解析器生成器（Tool Parser Generator）技能

一个 Claude Code 技能，通过分析 HuggingFace 模型的 chat template，为 Dynamo 添加工具调用（tool calling）支持。

## 概述

该技能为以下流程提供了一套系统化的工作流：
1. 从 HuggingFace 模型获取 chat template
2. 分析工具调用模式与格式
3. 与 Dynamo 已有的解析器进行匹配
4. 在需要时生成新的解析器实现
5. 创建合适的测试与集成代码

## 主要特性

- **以 LLM 为驱动**：充分利用 Claude 的代码分析与生成能力
- **最小化改动**：在可行情况下优先采用配置而非新增代码
- **参考已有实现**：与 sglang 和 vLLM 的实现进行对比
- **自动生成测试**：自动产出全面的测试用例
- **文档完善**：包含示例与集成指南

## 使用方式

只需让 Claude 为某个模型添加工具调用支持即可：

```
Add tool calling support for Qwen/Qwen2.5-72B-Instruct
```

Claude 将会：
1. 从 HuggingFace 获取该模型的 tokenizer 配置
2. 提取并分析 chat template
3. 与 Dynamo 已有的解析器进行对比
4. 决定复用已有解析器，还是生成一个新的
5. 创建测试用例并给出集成说明

## 目录结构

```
tool-parser-generator/
├── SKILL.md                       # 主技能文档与工作流
├── README.md                      # 本文件
└── references/
    ├── parser-patterns.md         # 常见模式速查
    └── integration-guide.md       # 分步集成指南
```

## 工作流阶段

1. **抓取与提取**：从 HuggingFace Hub 获取 chat template
2. **分析**：识别标记、格式类型与结构
3. **对比**：与 Dynamo 已有解析器进行匹配
4. **生成 / 配置**：创建新解析器或编写配置
5. **测试**：生成完整的测试用例
6. **集成**：将其加入 Dynamo 代码库并完成正确的注册

## 设计理念

- **优先复用**：大多数模型（>80%）可直接复用已有解析器
- **代码尽量少**：能用配置解决就不写实现
- **充分测试**：每个解析器都需要完善的测试
- **参考驱动**：从 sglang 与 vLLM 的实现中借鉴经验

## 参考资料

- **Dynamo 解析器**：`/lib/parsers/src/tool_calling/`
- **sglang**：https://github.com/sgl-project/sglang/tree/main/python/sglang/srt/function_call
- **vLLM**：https://github.com/vllm-project/vllm/tree/main/vllm/tool_parsers
- **HuggingFace**：https://huggingface.co/docs/transformers/chat_templating

## 示例

完整的 Qwen/Qwen2.5-72B-Instruct 支持添加流程，请参见 `SKILL.md`。

## 许可证

Apache 2.0 - 详情请参阅顶层 LICENSE 文件。
