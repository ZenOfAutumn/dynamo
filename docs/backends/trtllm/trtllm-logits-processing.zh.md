---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Logits 处理
---

有关 TensorRT-LLM 的通用特性与配置，请参阅[参考指南](trtllm-reference-guide.md)。

---

Logits 处理器允许你在每一步解码时修改 next-token logits（例如施加自定义约束或采样变换）。Dynamo 提供了一个与后端无关的接口，以及一个面向 TensorRT-LLM 的适配器，使你能够接入自定义处理器。

### 工作原理

- **接口**：实现 `dynamo.logits_processing.BaseLogitsProcessor`，定义 `__call__(input_ids, logits)` 并对 `logits` 进行原地修改。
- **TRT-LLM 适配器**：使用 `dynamo.trtllm.logits_processing.adapter.create_trtllm_adapters(...)` 将 Dynamo 处理器转换为 TRT-LLM 兼容的处理器，并赋值给 `SamplingParams.logits_processor`。
- **示例**：参见 `lib/bindings/python/src/dynamo/logits_processing/examples/` 中的示例处理器（[temperature](https://github.com/ai-dynamo/dynamo/tree/main/lib/bindings/python/src/dynamo/logits_processing/examples/temperature.py)、[hello_world](https://github.com/ai-dynamo/dynamo/tree/main/lib/bindings/python/src/dynamo/logits_processing/examples/hello_world.py)）。

### 快速测试：HelloWorld 处理器

可以启用一个仅用于测试的处理器，强制模型回复 "Hello world!"。这有助于在不修改模型或引擎代码的情况下验证接线是否正确。

```bash
cd $DYNAMO_HOME/examples/backends/trtllm
export DYNAMO_ENABLE_TEST_LOGITS_PROCESSOR=1
./launch/agg.sh
```

<Note>
- 启用后，Dynamo 会初始化 tokenizer，以便 HelloWorld 处理器可以将文本映射为 token ID。
- 预期的对话回复包含 "Hello world"。
</Note>

### 自带处理器

通过遵循 `BaseLogitsProcessor` 实现处理器，并对 logits 进行原地修改。例如温度缩放：

```python
from typing import Sequence
import torch
from dynamo.logits_processing import BaseLogitsProcessor

class TemperatureProcessor(BaseLogitsProcessor):
    def __init__(self, temperature: float = 1.0):
        if temperature <= 0:
            raise ValueError("Temperature must be positive")
        self.temperature = temperature

    def __call__(self, input_ids: Sequence[int], logits: torch.Tensor):
        if self.temperature == 1.0:
            return
        logits.div_(self.temperature)
```

通过适配并附加到 `SamplingParams` 来接入 TRT-LLM：

```python
from dynamo.trtllm.logits_processing.adapter import create_trtllm_adapters
from dynamo.logits_processing.examples import TemperatureProcessor

processors = [TemperatureProcessor(temperature=0.7)]
sampling_params.logits_processor = create_trtllm_adapters(processors)
```

### 当前限制

- 仅支持逐请求处理（batch size 必须为 1）；不支持 beam width > 1。
- 处理器必须就地修改 logits，不可返回新的张量。
- 如果你的处理器需要 tokenization，请确保 tokenizer 已初始化（不要跳过 tokenizer 初始化）。
