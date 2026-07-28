<!-- # SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License. -->

## Trace 文件格式

下面这些工具用于分析与基于 [mooncake trace 文件格式](https://github.com/kvcache-ai/Mooncake/blob/d21da178bae8db9651cf18a76824c084145fc725/mooncake_trace.jsonl) 合成新的数据。在该格式中，前几行示例如下：

```
{"timestamp": 0, "input_length": 6755, "output_length": 500, "hash_ids": [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13]}
{"timestamp": 0, "input_length": 7319, "output_length": 490, "hash_ids": [0, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27]}
{"timestamp": 3052, "input_length": 7234, "output_length": 794, "hash_ids": [0, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41]}
{"timestamp": 3052, "input_length": 2287, "output_length": 316, "hash_ids": [0, 42, 43, 44, 45]}
```

**Hash ID 生成方式：** 每个新的 hash ID 是上一次使用编号之后的下一个连续整数。两个 `hash_ids` 共享相同整数表示前缀重叠（prefix overlap）。要从一组文本生成 hash ID，可使用 `aiperf.dataset.synthesis.rolling_hasher` 中的 `texts_to_hashes`。

**Timestamp：** 请求自第一个请求起的到达时间（毫秒）。可以多个请求同时到达而具有相同时间戳。

**Block Size 与 Hash IDs：** 在该示例中，假定 `block_size`（KV 缓存的页大小）为 512。`hash_ids` 数组的长度等于 `input_length // block_size`。

## Prefix Analyzer（前缀分析器）

Prefix Analyzer 提供 trace 文件的统计信息，例如输入序列长度（Input Sequence Length，ISL）、输出序列长度（Output Sequence Length，OSL）以及理论缓存命中率。
它有助于理解数据集的结构与复用模式。

```bash
datagen analyze --input-file <path_to_trace.jsonl> --block-size <block_size>
```

- `--input-file`：jsonl 格式的 trace 文件路径（默认：`mooncake_trace.jsonl`）
- `--block-size`：用于前缀计算的块大小（默认：512）

脚本会输出 ISL、OSL、用户 prompt 长度的汇总统计信息，以及理论缓存命中率（假设缓存无限大）。

## Synthesizer（合成器）

Synthesizer 更进一步：
它从原始 trace 文件构建前缀树（prefix tree），提取前缀统计信息，并基于这些统计信息生成新的合成数据集。
你可以通过可调旋钮控制合成数据生成的多个方面，例如请求速率、上下文/prompt 长度倍率、以及树副本数量。

这对于在保留原始数据集结构性质的同时，生成大规模、贴近真实的合成 trace 用于基准测试或仿真非常有用。

### 运行方式

```bash
datagen synthesize --input-file <path_to_trace.jsonl> --num-requests <N> [其他选项...]
```

**选项：**
- `--input-file`：输入 trace 文件路径（默认：`mooncake_trace.jsonl`）
- `--num-requests`：要合成的请求数量（默认：100000）
- `--speedup-ratio`：请求间隔的加速倍数。会等效地将合成时间戳除以该值（默认：1）
- `--prefix-len-multiplier`：前缀长度倍率（默认：1.0）
- `--prefix-root-multiplier`：核心 radix 树的复制次数（默认：1）
- `--prompt-len-multiplier`：叶子路径长度倍率（默认：1.0，<1 表示使用更短的 prompt）
- `--max-isl`：写入输出的最大输入序列长度（默认：None，不过滤）
- `--block-size`：用于预填充与解码的块大小（默认：512）
- `--output-file`：输出文件路径（默认：根据输入文件与选项自动生成）

### 示例

假设我们仅有以下 hash 列表：

```
[0, 1, 2, (3)]
[0, 1]
[0, 1, 2]
[0, (4), (5)]
```

首先，我们识别 “核心前缀节点” 为 [0, 1, 2]，因为它们被访问超过一次。节点 [3, 4, 5] 仅出现一次（用括号标注），将被视为 “用户 prompt”。

如果将 `prefix-len-multiplier` 设为 2，则核心前缀分支会被拉伸，效果如下：

```
[0, 1, 2, 3, 4, 5, (6)]
[0, 1, 2, 3]
[0, 1, 2, 3, 4, 5]
[0, 1, (7), (8)]
```


注意 “prompt 分支” 不会被 `prefix-len-multiplier` 拉伸，可单独通过 `prompt-len-multiplier` 修改。

接下来，如果将 `prefix-root-multiplier` 设为 2，则每一行有 50% 的概率被加上一个较大整数偏移，使其等效地分裂到一棵新的 radix 树，与原树具有相同统计特性，但根节点完全不同。

例如，如果第 2 与第 4 行被偏移，会得到：

```
[0, 1, 2, 3, 4, 5, (6)]
[10, 11, 12, 13]
[0, 1, 2, 3, 4, 5]
[10, 11, (14), (15)]
```

### 实现细节

简化后的生成算法如下：

- 将 hash id 存入有向树结构（前缀树）
- 每条有向边的 `weight` 表示该边被遍历的次数，用于计算转移概率。
- 收缩树中的一元路径（chain），使其呈 radix-tree 形式，即每个唯一子节点都会与父节点合并。因此每个节点需要一个 `length` 属性，表示压缩后的长度（无压缩时为 1）。深度倍率会按比例放大该压缩长度（取整），从而增加每个 radix 节点的有效长度。
- 找出仅被访问一次的叶节点，将其从树中剪除，因为它们极有可能不属于核心 radix 树。换言之，无需存储那些实际是用户 prompt 的节点。
- 此时每个节点都具备（可能为零的）转移概率：到子前缀节点、到 “用户 prompt” 节点，以及到 “终止” 节点。利用这些概率在核心 radix 树上采样一条路径，然后追加新生成的 hash id 表示一个长度从数据集采样得到的用户 prompt。宽度倍率会将整棵 radix 树整体复制指定次数，每次使用全新的 hash id，从而创造更多样化的请求模式。

## 测试

为了测试 “正确性”，即对原始 trace 统计的忠实度，可运行：
```
python -m benchmarks.data_utils.synthesizer \
--input-file mooncake_trace.jsonl \
--num-requests 500000 \
```
然后将合成的 ISL 统计（均值、中位数、标准差）与原始 ISL 统计做对比；后者可通过：
```
python -m benchmarks.data_utils.prefix_analyzer \
--input-file mooncake_trace.jsonl \
```
得到。我认为这是最 “稳健” 的端到端测试。务必采样足够多的请求（如几十万）以确保统计有意义（大数定律）。特别地，均值类统计（如 ISL 均值）应在合成数据中保持得很好；但标准差类统计 —— 尤其是 ISL —— 不会精确匹配，因为合成器并未捕捉原始数据中上下文长度与 prompt 长度之间的相关性。
