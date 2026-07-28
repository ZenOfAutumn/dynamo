# Tokenizer 基准测试

## tokenizer_simple

使用 Criterion 与本地测试数据。通过 `cargo bench` 自动运行。

```bash
cargo bench --bench tokenizer_simple -p dynamo-llm
```

## tokenizer_dataset

从 HuggingFace Hub 下载真实数据集（LongBench-v2，约 500 个样本），并测量编码吞吐量，对比 `HuggingFaceTokenizer` 与 `FastTokenizer`。

该基准是 **opt-in（按需启用）** 的：除非设置了 `RUN_BENCH=1`，否则会立即退出。这避免了它在 CI 中执行 `cargo test --all-targets` 时被运行，因为它需要几分钟才能完成。

### 基本运行（默认：Qwen/Qwen3-0.6B、LongBench-v2、503 个样本）

```bash
RUN_BENCH=1 cargo bench --bench tokenizer_dataset -p dynamo-llm
```

### 覆盖 tokenizer

```bash
RUN_BENCH=1 TOKENIZER_PATH=deepseek-ai/DeepSeek-V3 \
  cargo bench --bench tokenizer_dataset -p dynamo-llm
```

使用本地的 `tokenizer.json` 文件：

```bash
RUN_BENCH=1 TOKENIZER_PATH=/path/to/tokenizer.json \
  cargo bench --bench tokenizer_dataset -p dynamo-llm
```

### 覆盖数据集与样本数

```bash
RUN_BENCH=1 DATASET=RyokoAI/ShareGPT52K MAX_SAMPLES=50 \
  cargo bench --bench tokenizer_dataset -p dynamo-llm
```

### 批量模式

```bash
RUN_BENCH=1 BATCH_SIZE=64 cargo bench --bench tokenizer_dataset -p dynamo-llm
```

### 环境变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `RUN_BENCH` | 未设置 | 必须设为任意值才能运行该基准 |
| `TOKENIZER_PATH` | `Qwen/Qwen3-0.6B` | HuggingFace 模型名称或本地 `tokenizer.json` 路径 |
| `DATASET` | `zai-org/LongBench-v2` | HuggingFace 数据集名称（`zai-org/LongBench-v2` 或 `RyokoAI/ShareGPT52K`） |
| `MAX_SAMPLES` | `503` | 处理的最大样本数 |
| `BATCH_SIZE` | 未设置 | 若设置，则以批量模式运行而非顺序模式 |
