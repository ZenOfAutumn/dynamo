# Dynamo Codegen

用于 Dynamo Python 绑定的 Python 代码生成器。

## gen-python-prometheus-names

从 Rust 源文件 `lib/runtime/src/metrics/prometheus_names.rs` 生成 `prometheus_names.py`。

### 使用方式

```bash
cargo run -p dynamo-codegen --bin gen-python-prometheus-names
```

### 它做了什么

- 解析 `lib/runtime/src/metrics/prometheus_names.rs` 的 Rust AST
- 在 `lib/bindings/python/src/dynamo/prometheus_names.py` 生成包含常量的 Python 类

### 示例

**Rust 输入：**
```rust
pub mod kvrouter {
    pub const KV_CACHE_EVENTS_APPLIED: &str = "kv_cache_events_applied";
}
```

**Python 输出：**
```python
class kvrouter:
    KV_CACHE_EVENTS_APPLIED = "kv_cache_events_applied"
```

### 何时运行

在修改 `lib/runtime/src/metrics/prometheus_names.rs` 后运行，以重新生成 Python 文件。
