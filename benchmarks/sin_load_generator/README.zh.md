<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 正弦曲线负载生成器（Sinusoidal Load Generator）

`sin_synth.py` 是一个简单脚本，用于生成请求速率与 isl/osl 比例呈正弦变化的合成负载。输出采用 [mooncake 风格](https://github.com/kvcache-ai/Mooncake) 的 jsonl 格式，可直接用于 [AIPerf](https://github.com/ai-dynamo/aiperf)。

## 使用方法

```bash
cd benchmarks/sin_load_generator
python sin_synth.py [OPTIONS]
```

### 基础选项

- `--block-size INT`（默认：512）
  - 用于哈希的块大小。由于本工具不涉及前缀缓存，因此该 block size 不必与引擎实际使用的 KV block size 相同。

- `--total-blocks INT`（默认：10000)
  - ISL 所用的 prompt 块从此范围中随机采样。值越大，重复 prompt 出现的概率越低。

- `--output-file STR`（默认：自动生成）
  - 输出文件名（jsonl 格式）
  - 若未指定，脚本将基于参数生成一个文件名

- `--time-duration INT`（默认：100）
  - 数据集总时长，单位秒

- `--process-interval INT`（默认：1）
  - 用于生成数据集的采样间隔
  - 间隔越小，请求速率与 isl/osl 比例的变化越精确，但生成时间会更长。

### 请求速率参数

请求速率遵循正弦模式：
```
request_rate(t) = (min + max) / 2 + (max - min) / 2 * sin(2 * π / period * t - π / 2)
```

注意 `-π/2` 的相位偏移使得请求速率在 `t = 0` 时从最小值开始。

- `--request-rate-min FLOAT`（默认：5）
  - 最小请求速率，单位：请求/秒

- `--request-rate-max FLOAT`（默认：10）
  - 最大请求速率，单位：请求/秒

- `--request-rate-period FLOAT`（默认：10）
  - 正弦请求速率的周期，单位：秒

### 输入/输出序列长度参数

脚本会从两组预设的 ISL/OSL 组合中采样请求。
ISL/OSL 比例定义了多少比例的请求遵循第一组预设 ISL/OSL 模式：值为 0 表示全部请求采用第一组，值为 1 表示全部采用第二组。

ISL/OSL 比例同样按正弦曲线变化：
```
isl-osl-ratio(t) = (min + max) / 2 + (max - min) / 2 * sin(2 * π / period * t - π / 2)
```

同样地，`-π/2` 的相位偏移使得 ISL/OSL 比例在 `t = 0` 时从最小值开始。

- `--isl1 INT`（默认：100）
  - 最小输入序列长度

- `--osl1 INT`（默认：2000）
  - 最小输出序列长度

- `--isl2 INT`（默认：5000）
  - 最大输入序列长度

- `--osl2 INT`（默认：100）
  - 最大输出序列长度

- `--isl-osl-ratio-min FLOAT`（默认：0.2）
  - 输入序列长度与输出序列长度比例的最小值

- `--isl-osl-ratio-max FLOAT`（默认：0.8）
  - 输入序列长度与输出序列长度比例的最大值

- `--isl-osl-ratio-period FLOAT`（默认：10）
  - 正弦输入/输出序列长度比例的周期

### 示例

#### 请求速率变化、ISL/OSL 比例固定

```bash
python sin_synth.py \
  --time-duration 60 \
  --request-rate-min 2 \
  --request-rate-max 8 \
  --request-rate-period 20 \
  --isl1 3000 \
  --osl1 150 \
  --isl2 3000 \
  --osl2 150 \
  --output-file dataset.jsonl
```

生成 60 秒数据集：请求速率以 20 秒为周期在 2–8 请求/秒之间变化，ISL=3000、OSL=150。ISL/OSL 比例固定为 0.2。

#### ISL/OSL 比例变化、请求速率固定

```bash
python sin_synth.py \
  --time-duration 60 \
  --request-rate-min 5 \
  --request-rate-max 5 \
  --isl1 3000 \
  --osl1 150 \
  --isl2 500 \
  --osl2 2000 \
  --isl-osl-ratio-min 0.2 \
  --isl-osl-ratio-max 0.8 \
  --isl-osl-ratio-period 20 \
  --output-file dataset.jsonl
```

生成 60 秒数据集：请求速率固定为 5 请求/秒，ISL/OSL 比例以 20 秒为周期在 0.2 与 0.8 之间变化，对应 I3000O150 与 I500O2000 之间的切换。
