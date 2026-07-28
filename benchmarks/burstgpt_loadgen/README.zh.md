# BurstGPT 负载生成器转换工具

一个用于把包含 ChatGPT/GPT-4 对话日志的 CSV 文件转换为 mooncake 风格 JSONL 格式
的工具，便于负载测试与仿真。

> [!NOTE]
> 当前的输出未考虑 KV 复用。一旦 [BurstGPT](https://github.com/HPMLL/BurstGPT)
> 添加用户会话信息，我们会更新该脚本。

## 输入格式

输入 CSV 可从 [BurstGPT Release v1.1](https://github.com/HPMLL/BurstGPT/releases/tag/v1.1)
下载：
- `Timestamp`：请求时间戳，单位秒
- `Model`：模型名称（例如 "ChatGPT", "GPT-4"）
- `Request tokens`：输入 token 数量
- `Response tokens`：输出 token 数量
- `Total tokens`：总 token 数（未使用）
- `Log Type`：日志类型（例如 "Conversation log", "API log"）

示例：
```csv
Timestamp,Model,Request tokens,Response tokens,Total tokens,Log Type
5,ChatGPT,472,18,490,Conversation log
45,ChatGPT,1087,230,1317,Conversation log
118,GPT-4,417,276,693,Conversation log
```

## 输出格式

输出是一个 JSONL 文件，每行一个 JSON 对象：
```json
{"timestamp": 5000, "input_length": 472, "output_length": 18, "hash_ids": [123, 456, 789, ...]}
```

字段说明：
- `timestamp`：请求时间，单位毫秒（整数）
- `input_length`：输入 token 数量
- `output_length`：输出 token 数量
- `hash_ids`：用于模拟 KV 缓存（KV cache）块的随机哈希 ID 数组

## 用法

### 基本用法

```bash
python convert.py --input-file <BurstGPT CSV data>
```

如果未指定 `--output-file`，输出文件名将使用输入文件名并替换扩展名为 `.jsonl`。

### 命令行参数

#### 必填参数
- `--input-file`：输入 CSV 文件的路径

#### 可选参数

**过滤：**
- `--model`：按模型过滤（`ChatGPT` 或 `GPT-4`），None 表示不过滤
- `--log-type`：按日志类型过滤（`Conversation log` 或 `API log`），None 表示不过滤
- `--skip-num-prompt`：在过滤之后跳过前 N 行（默认：0）。在 `--num-prompt`
  **之前**应用。
- `--num-prompt`：限制最终输出中的行数，None 表示不过滤（在 `--skip-num-prompt`
  **之后**应用）

**时间戳调整：**
- `--speed-ratio`：调整请求时序（默认：1.0）
  - 大于 1：加速（例如 2.0 = 2 倍速）
  - 小于 1：减速（例如 0.5 = 0.5 倍速）
  - 公式：`new_timestamp = old_timestamp / speed_ratio`
  - 在过滤 / 跳过 / 截断以及 speed-ratio 调整之后，时间戳会整体平移，使保留下来
    的第一个请求从 `t=0` 开始。

**哈希生成：**
- `--block-size`：mooncake 轨迹中的 block 大小（默认：128）
- `--num-hash-blocks`：哈希 ID 的最大值（默认：10000）。每个 block 的哈希 ID 都
  会从 0 到该值之间随机选取。
**输出：**
- `--output-file`：输出 JSONL 文件路径（默认为：输入文件名替换扩展名为 .jsonl）

## 统计输出

转换完成后，脚本会输出有关生成的负载的统计信息：

```
============================================================
STATISTICS
============================================================

Input Length (ISL):
  Min: 37
  Max: 1528
  Avg: 705.89
  Std: 524.33

Output Length (OSL):
  Min: 18
  Max: 1656
  Avg: 494.67
  Std: 513.21

Sequence Length (ISL + OSL):
  Max: 3184

Request Rate:
  Total requests: 9
  Duration: 405.00 seconds
  Average RPS: 0.02
============================================================
```
