# lib/kv-router/src/standalone_indexer

独立 indexer 必须保持可在不依赖 Dynamo runtime 或 LLM 层的情况下使用。

## 准则

- 不要引入 `lib/runtime` / `dynamo-runtime` 依赖，除非它们被显式地通过 feature 开关进行隔离。
- 不要引入 `lib/llm` / `dynamo-llm` 依赖，无论是否通过 feature 开关。
