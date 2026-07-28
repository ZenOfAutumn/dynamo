# Tokenizers

## 简介
`dynamo-tokenizers` 为 NLP 工作负载提供高效、通用的分词（tokenization）能力。它通过简洁的编码/解码 API 支持 HuggingFace 与 TikToken 分词器（tokenizer），并额外提供 FastTokenizer 混合模式。

## 功能特性
- **哈希校验**：在不同模型之间确保分词的一致性与准确性。
- **简洁的编码与解码**：方便地完成文本与 token ID 之间的相互转换。
- **序列管理**：高效地管理 token 序列，以应对复杂的 NLP 任务。

## 快速上手

#### HuggingFace 分词器
```rust
use dynamo_tokenizers::hf::HuggingFaceTokenizer;

let hf_tokenizer = HuggingFaceTokenizer::from_file("tests/data/sample-models/TinyLlama_v1.1/tokenizer.json")
    .expect("Failed to load HuggingFace tokenizer");
```

### 文本的编码与解码

```rust
use dynamo_tokenizers::{HuggingFaceTokenizer, traits::{Encoder, Decoder}};

let tokenizer = HuggingFaceTokenizer::from_file("tests/data/sample-models/TinyLlama_v1.1/tokenizer.json")
    .expect("Failed to load HuggingFace tokenizer");

let text = "Your sample text here";
let encoding = tokenizer.encode(text)
    .expect("Failed to encode text");

println!("Encoding: {:?}", encoding);

let decoded_text = tokenizer.decode(&encoding.token_ids, false)
    .expect("Failed to decode token IDs");

assert_eq!(text, decoded_text);

// 使用 Sequence 对象进行编码与解码

use dynamo_tokenizers::{Sequence, Tokenizer};
use std::sync::{Arc, RwLock};

let tokenizer = Tokenizer::from(Arc::new(tokenizer));
let mut sequence = Sequence::new(tokenizer.clone());

sequence.append_text("Your sample text here")
    .expect("Failed to append text");

let delta = sequence.append_token_id(1337)
    .expect("Failed to append token_id");
```
