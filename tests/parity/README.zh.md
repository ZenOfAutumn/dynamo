# 跨实现 Parity（一致性）测试套件

用于在 Dynamo、vLLM 与 SGLang 之间比对 parser / preprocess / postprocess 行为差异的共享测试基础设施。当前仅 parser 阶段已落地（`parser/`）；其他阶段会作为同级目录陆续加入。

## 目录布局

```
tests/parity/
├── README.md                       (this file)
├── conftest.py                     ← session-scoped fixtures (server boots, etc.)
├── common.py                       ← ParseResult, canonical-JSON diff, decode_arguments
└── parser/
    ├── fixtures/                   ← static YAML, generated from Dynamo as oracle
    │   └── <family>/PARSER.batch.yaml         (and per-top-level-case files like PARSER.batch.8.yaml; see Fixture file schema)
    ├── regenerate_fixtures.py      ← (re-)build fixtures by running Dynamo's parser
    │
    ├── dynamo.py                   ← M2 in-process wrapper (PyO3 binding)
    ├── vllm.py                     ← M2 in-process wrapper (ToolParserManager)
    ├── sglang.py                   ← M2 in-process wrapper (per-module detectors)
    ├── test_parity_parser.py       ← M2 harness (parser-class parity)
    │
    ├── server.py                   ← M3 subprocess boot helper
    ├── client.py                   ← M3 HTTP client (vllm + sglang)
    └── test_parity_e2e.py          ← M3 harness (server-stack parity over HTTP)
```

## 三种方法（M1、M2、M3）—— 各自的真正含义

驱动同一份 parser 逻辑的三种**真实程度递增**的方式。它们的区别在于**哪些被替换、哪些是真实的**，这决定了每种方法能捕获哪一类 bug：

```
                                                  ┌─ engine ──────────┐
                                                  │                   │
   client ─request→ chat-template ─→ tokenize ─→ engine ─→ detokenize ─→ text ─→ PARSER ─→ tool_calls JSON ─→ client
              ↑                                                              ↑
              └── M1 / M3 exercise this (real)                               └── M2 starts here (skips everything left)
                  M2 skips it (substituted by direct call)
```

### Method 1 —— 回退路径测试 *（即将到来，下一步）*

**运行内容：** `python -m dynamo.frontend --dyn-chat-processor <vllm|sglang>`。Dynamo 前端 chat processor 把工具解析委托给上游的 Python parser，而不是 Dynamo 的 Rust parser。参考路径是默认的 `python -m dynamo.frontend`（Dynamo 自家 Rust parser）。

**揭示什么：** 被回退掩盖的 Dynamo Rust parser 的 gap；以及 Dynamo 前端用于包装上游 parser 的代码 bug。

**状态：** 尚未实现。落地后会有自己的 harness 文件（M1 测的是 *Dynamo 对上游 parser 的包装*，而不是不同实现*之间*的一致性）。

### Method 2 —— parser 类测试（本 PR 的主要 harness）

**运行内容：** 进程内 Python import，三种实现共存于同一进程 —— 没有 HTTP、没有模型、没有 tokenizer、不做 chat-template 物化：

```python
# Dynamo Rust side (via PyO3 binding)
from dynamo._core import parse_tool_call
result = await parse_tool_call("kimi_k2", text, tools_json)

# vLLM side (native Python class)
from vllm.tool_parsers import ToolParserManager
parser = ToolParserManager.get_tool_parser("kimi_k2")(tokenizer=stub)
info = parser.extract_tool_calls(text, request)

# SGLang side (native Python class)
from sglang.srt.function_call.kimik2_detector import KimiK2Detector
result = KimiK2Detector().detect_and_parse(text, tools)
```

**揭示什么：** Dynamo Rust parser 类与上游 Python parser 类之间的 parser 逻辑差异 —— 把这一类 bug 与请求生命周期中的其他部分隔离开。

**文件：** `tests/parity/parser/test_parity_parser.py`。

### Method 3 —— 端到端 HTTP 测试 *（即将到来，下一步 —— 兄弟 PR #9189）*

**运行内容：** 真实的上游 serving 二进制；通过约束解码（constrained decoding）强制其输出 fixture 的 text：

```bash
vllm serve <model> --load-format dummy \
  --enable-auto-tool-choice --tool-call-parser kimi_k2 --port 8001 &
python -m sglang.launch_server --model-path <model> --load-format dummy \
  --tool-call-parser kimi_k2 --port 8002 &
```

两个服务都收到完全相同的 chat-completion 请求，并通过 `structured_outputs.regex`（vLLM）/ `regex`（SGLang）强制 assistant 轮逐字节地输出 fixture 的 `model_text`；harness 从每个响应中捕获 `tool_calls` JSON。

**揭示什么：** vLLM 与 SGLang HTTP 流水线（请求预处理、tokenizer 往返、流式 chunk 边界、响应整形）之间的服务栈差异 —— 这些是类级别测试（M2）看不到的。

**文件：** `tests/parity/parser/test_parity_e2e.py`（在 #9189 落地）。

### 对比

| | M1（即将） | M2（本 PR） | M3（即将，#9189） |
|---|---|---|---|
| **测试对象** | Dynamo 前端对上游 parser 的包装 | parser **类**（独立） | parser 在它的**服务**里（完整 HTTP 栈） |
| **调用方式** | `python -m dynamo.frontend` 子进程 | 进程内 Python import | HTTP 走 `/v1/chat/completions` |
| **真实引擎？** | 是 | 否 | 是（带 `--load-format dummy`） |
| **真实 tokenizer？** | 是 | 否 | 是 |
| **真实 chat template？** | 是 | 否 | 是 |
| **真实 HTTP？** | 是 | 否 | 是 |
| **成本** | （TBD） | 570 个测试约 3 秒 | 30 个测试约 60 秒（服务启动占大头） |
| **GPU** | 是 | 不需要 | 是（`--load-format dummy` 仍占用约 2.5 GiB） |
| **CI markers** | （TBD） | `unit, pre_merge, gpu_0` | `e2e, pre_merge, gpu_1` |

三种方法（实现后）共享同一套 fixture、`ParseResult` 结构以及 `KNOWN_DIVERGENCES` 注册表模式。它们是层叠诊断：

- M2 说："*parser 类不一致*"
- M3 说："*服务栈也不一致*"（更有用的是："*只有服务栈不一致 —— parser 类是一致的*"，这把 bug 定位到 chat-template / tokenizer / 响应整形）
- M1 说："*Dynamo 的包装层与它包装的上游 parser 不一致*"

## 当前 parity 状态（M2，batch 模式）

每行是一个 parser 家族；每列 `bN` 对应 [`PARSER.batch.N`](../../lib/parsers/PARSER_CASES.md)：

```
b1   =  PARSER.batch.1   "single happy-path call"
b2   =  PARSER.batch.2   "multiple calls"
b3   =  PARSER.batch.3   "no tool call (plain text)"
b4   =  PARSER.batch.4   "malformed JSON args"
b5   =  PARSER.batch.5   "missing end-token recovery"
b6   =  PARSER.batch.6   "empty args (no-arg call)"
b7   =  PARSER.batch.7   "complex args (nested JSON / arrays)"
b8   =  PARSER.batch.8   "interleaved normal text"
b9   =  PARSER.batch.9   "empty input"
b10  =  PARSER.batch.10  "duplicate calls (same name twice)"
```

单元格的值表示与 Dynamo（oracle）的差异：

- `✓` —— vLLM 与 SGLang 都与 Dynamo 期望输出一致。
- `V` —— vLLM 出现差异（通过 `KNOWN_DIVERGENCES` 标记为 xfail）。
- `S` —— SGLang 出现差异。
- `VS` —— 两者都出现差异。
- `n/a` —— 该家族在该实现中没有对等的 parser。

总共 19 个 parser 家族 —— 分为我们优先关注的 **Top-N 模型**（行末括号里写明模型名）以及非 top-N 但已接入 harness 以求完整性的 **Others**。两段内分别按字母序排列。

| family                          |  b1 |  b2 |  b3 |  b4 |  b5 |  b6 |  b7 |  b8 |  b9 | b10 |
|---------------------------------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Top-N models**                |     |     |     |     |     |     |     |     |     |     |
| deepseek_v4 § (DeepSeek V4)     | ✓   | ✓   | ✓   | ✓   | V   | ✓   | V   | ✓   | ✓   | ✓   |
| gemma4 § (Gemma 4)              | ✓   | ✓   | ✓   | ✓   | ✓   | ✓   | ✓   | V   | ✓   | ✓   |
| glm47 (GLM 5.1)                 | ✓   | ✓   | ✓   | ✓   | VS  | ✓   | ✓   | V   | ✓   | ✓   |
| harmony † (gpt-oss)             | S   | S   | ✓   | S   | S   | S   | S   | S   | ✓   | S   |
| kimi_k2 (Kimi K2.6)             | ✓   | ✓   | ✓   | S   | ✓   | ✓   | ✓   | VS  | ✓   | ✓   |
| minimax_m2 (MiniMax 2.7)        | ✓   | ✓   | ✓   | V   | VS  | ✓   | ✓   | ✓   | ✓   | ✓   |
| nemotron_deci †§ (Nemotron)     | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| qwen3_coder (Qwen 3.5)          | ✓   | ✓   | ✓   | ✓   | ✓   | ✓   | ✓   | ✓   | ✓   | ✓   |
| **Others**                      |     |     |     |     |     |     |     |     |     |     |
| deepseek_v3                     | ✓   | ✓   | ✓   | V   | VS  | ✓   | ✓   | S   | ✓   | ✓   |
| deepseek_v3_1                   | ✓   | ✓   | ✓   | V   | VS  | ✓   | ✓   | S   | ✓   | ✓   |
| deepseek_v3_2                   | ✓   | ✓   | ✓   | ✓   | V   | ✓   | V   | S   | ✓   | ✓   |
| hermes                          | ✓   | ✓   | ✓   | VS  | ✓   | ✓   | ✓   | V   | ✓   | ✓   |
| jamba §                         | ✓   | ✓   | ✓   | ✓   | V   | ✓   | ✓   | V   | ✓   | ✓   |
| llama3_json §                   | ✓   | V   | ✓   | V   | V   | ✓   | ✓   | ✓   | ✓   | V   |
| mistral                         | S   | S   | ✓   | VS  | S   | S   | S   | VS  | ✓   | S   |
| nemotron_nano †§                | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| phi4 §                          | ✓   | ✓   | ✓   | ✓   | V   | ✓   | V   | V   | ✓   | ✓   |
| pythonic                        | ✓   | ✓   | ✓   | ✓   | ✓   | ✓   | ✓   | VS  | ✓   | ✓   |
| qwen25 †                        | S   | S   | ✓   | S   | S   | S   | S   | S   | ✓   | S   |

### 脚注

† vLLM 没有对等的 parser（或运行时返回 `UNAVAILABLE`，例如 `harmony#vllm` 需要 token IDs 而非文本）。在接入 SGLang 时表中只显示 SGLang 状态；否则该行整行 `n/a`。
§ SGLang 没有对应的 detector。在接入 vLLM 时表中只显示 vLLM 状态；否则该行整行 `n/a`。

`nemotron_deci` 与 `nemotron_nano` 同时带两个标记（`†§`）—— 上游均无对等 parser，因此整行 `n/a`。这里列出仅为完整性；可以在后续工作中加入 Dynamo 自一致性校验，但不会产生跨实现信号。

合计（不含 `dynamo`，因为它是 oracle）：380 格对照 YAML 期望 —— 203 格一致、67 格差异（xfail）、110 格 n/a。

**热点列：**
- `PARSER.batch.4`（畸形 JSON）与 `PARSER.batch.5`（缺失结束 token）—— `PARSER_CASES.md` 规定其恢复行为是实现自定义的；差异仅记录而非断言。
- `PARSER.batch.8`（与正常文本交错）—— vLLM 与 SGLang 在 XML 风格家族里都丢掉了 wrapper 之后的尾部文本。
- `harmony/sglang` —— 10 个用例中 8 个有差异，因为 SGLang 的 `GptOssDetector` 严格要求 `<|start|>assistant<|channel|>commentary` 这种封套，而 Dynamo 接受裸的 commentary。

**尚未覆盖的：** `PARSER.fmt.*`、`PARSER.xml.*`、`PARSER.harmony.*`、`PARSER.stream.*` —— 这些用例类目前仅以 Rust 单元测试形式存在，会在后续 PR 中加入 YAML 语料。

## 运行 parity harness

在 Dynamo devcontainer 内一行命令：

```bash
PYTHONPATH=lib/bindings/python/src python3 -m pytest tests/parity/parser/test_parity_parser.py
```

`PYTHONPATH=lib/bindings/python/src` 必填 —— PyO3 binding（`dynamo._core`）在那里。没有它，所有 Dynamo 用例都会被 skip。vLLM / SGLang 包也必须安装，否则它们的用例会干净地 skip。

**常见过滤** —— 选要跑的：

```bash
# One family across every bucket
... -k "kimi_k2"

# One sub-case across every family + every impl
... -k "PARSER.batch.8.b"

# Exactly one (family, sub-case, impl)
... "tests/parity/parser/test_parity_parser.py::test_parity[kimi_k2/PARSER.batch.8.b#vllm]"
```

**从主机**（容器外）运行时，把命令包在 `docker exec <container> bash -c 'cd /workspace && …'` 中。容器名可用 `docker ps --format '{{.Names}}' | grep vsc-dynamo2 | head -1` 找到。

**基线：** 在 `main` post-#9338 上，预计约 5 秒内 ~378 passed / ~299 skipped / ~64 xfailed。**xfailed 的数量等于 `KNOWN_DIVERGENCES` 的大小** —— 完美 parity 即该计数为 0（三种实现在每条 fixture 上字节级一致）。

## 解决差异（8 步）

上表中每个非 `✓` 单元格都对应 `KNOWN_DIVERGENCES`（`tests/parity/parser/test_parity_parser.py`）中的一个条目。目标：把这个数推到 0。每次走以下流程，处理一个单元格。示例：`kimi_k2 / PARSER.batch.8.b → V`（vLLM 丢弃 wrapper 之后的尾部文本）。

### 1. 选一个单元格

矩阵中任意 `V` / `S` / `VS` 的格。

### 2. 查阅原因与并排 diff

```bash
grep -A 1 '"vllm", "kimi_k2", "PARSER.batch.8.b"' \
  tests/parity/parser/test_parity_parser.py
```

然后打开 `tests/parity/parser/fixtures/kimi_k2/PARSER.batch.8.yaml`，找到 `PARSER.batch.8.b:` 块。`expected:` 之后的 `TODO(research)` 注释就是并排 diff —— Dynamo 的输出 vs 该实现的输出。**不需要跑 harness 就能读到。**

```yaml
PARSER.batch.8.b:
  description: Narration after tool call only
  ref: originated from https://github.com/vllm-project/vllm/.../test_kimi_k2_tool_parser.py#L435
  model_text: |-
    <|tool_calls_section_begin|>...
  expected:
    calls: [...]
    normal_text: <Dynamo's output>
  # TODO(research): vllm diverges — <reason from KNOWN_DIVERGENCES>
  # vllm produces:
  #   calls: [...]
  #   normal_text: <vLLM's actual output>
```

如果原因是泛化的（如 `"sglang diverges on this sub-case (CI-surfaced)"`），那么 `TODO(research)` 块就是*唯一*能看到实际 diff 的地方 —— 批量注册的条目跳过了逐用例分析。

如果存在 `ref: originated from <url>`，URL 指向了上游 vLLM 测试，是有用的上下文。

### 3. 本地复现

```bash
PYTHONPATH=lib/bindings/python/src python3 -m pytest \
  "tests/parity/parser/test_parity_parser.py::test_parity[kimi_k2/PARSER.batch.8.b#vllm]" \
  -v --tb=short
```

实现 tag 是 `#vllm` 或 `#sglang`。

### 4. 判断谁对

- **Dynamo 错、上游对** → 修 Dynamo（步骤 5）。
- **Dynamo 对、上游故意错** → 保留条目，去 `vllm-project/vllm` 或 `sgl-project/sglang` 提上游 bug，并把链接加到差异原因里。
- **两边都错 / 规范模糊** → 动代码前先讨论。

### 5. 修 parser

编辑 `lib/parsers/src/tool_calling/<family>/...rs`。重建 PyO3 binding，让 fixture 重生器能跑你的改动：

```bash
cd lib/bindings/python && maturin develop --uv && cd -
```

前置条件参见 [构建指南](../../docs/getting-started/building-from-source.md)。

### 6. 重新生成 fixture

```bash
PYTHONPATH=lib/bindings/python/src python3 \
  -m tests.parity.parser.regenerate_fixtures --overwrite-if-exists
```

Dynamo 是 oracle —— 你的改动改变了 Dynamo 的输出，因此 `expected:` 块会被重写以匹配新输出。如果跳过这一步，你的修复看起来就像是*新的*差异。

### 7. 重跑 pytest，注意 XPASS-strict

```bash
PYTHONPATH=lib/bindings/python/src python3 -m pytest \
  tests/parity/parser/test_parity_parser.py -q --tb=no
```

如果三种实现现在都同意，harness 会把注册表标记为陈旧：

```text
FAILED ...test_parity[kimi_k2/PARSER.batch.8.b#vllm]
       XPASS-strict: known divergence (vllm,kimi_k2,PARSER.batch.8.b)
       now matches expected — remove from KNOWN_DIVERGENCES.
```

### 8. 删除条目 + 添加 spec_ref + 刷新注释 + 提交

把这一行从 `KNOWN_DIVERGENCES` 中删除：

```python
("vllm", "kimi_k2", "PARSER.batch.8.b"): "...",   # ← delete
```

在 `regenerate_fixtures.py` 的 INPUTS 项里给该用例加上 `spec_ref:` 字段，这样追溯链条就能保留 —— 指向证明 V/S 是对的依据（规范章节、模型卡、GH issue、上游 PR、或团队决策文档）：

```python
("kimi_k2", "PARSER.batch.8.b"): {
    "description": "Narration after tool call only",
    "ref": "originated from https://github.com/vllm-project/vllm/.../test_kimi_k2_tool_parser.py#L435",
    "spec_ref": "https://platform.moonshot.ai/docs/tool-call-spec#L42  (or GH issue / PR url)",
    "text": ...,
    "tools": [...],
},
```

然后再跑一次 regen 让该字段流到 fixture YAML，并 embed，把已经过时的 `TODO(research)` 块去掉：

```bash
PYTHONPATH=lib/bindings/python/src python3 \
  -m tests.parity.parser.regenerate_fixtures --overwrite-if-exists
PYTHONPATH=lib/bindings/python/src python3 \
  -m tests.parity.parser.embed_divergence_comments
```

也可以在 YAML 中给该用例旁留一条人读注释（注意：当前在下次 `regenerate_fixtures` 时会被丢掉 —— 保留这部分是后续工具改进项）：

```yaml
PARSER.batch.8.b:
  description: Narration after tool call only
  ref: originated from https://...
  spec_ref: https://...
  # aligned with V+S+spec on YYYY-MM-DD — was a Dynamo-only divergence;
  #   parser updated to drop trailing text after wrapper end.
  model_text: ...
```

pytest 现在应当全绿；矩阵下次重生时该格翻为 `✓`。

```bash
git add lib/parsers/ tests/parity/parser/
git commit -s -m "fix(parser): align <family> with <impl> on PARSER.batch.N.x"
git push
```

## 何时**不应**修复 —— 永久差异

注册表应该缩小，而不是膨胀。但少数几类会永远存在：

- **`PARSER.batch.4`（畸形 args）与 `PARSER.batch.5`（缺失结束 token）** —— 按 `PARSER_CASES.md` 规定，恢复策略由各实现定义。每个 parser 自选行为（drop / 恢复 / 回退到字符串 / 报错）。不期望一致。
- **schema 驱动的强制类型转换 vs parser 层保留** —— 例如 `{"celsius": "20"}` 对应 `celsius: integer`。某些 parser 在 parser 层就强转，另一些保留原值并把强转推给下游。两者都站得住脚。

其他都属于可修复候选。

## Fixture 文件 schema

每个家族有两种文件布局并存。loader 会合并它们：

```
<family>/PARSER.batch.yaml          ← legacy flat: holds top-level cases (1, 2, ..., 10)
<family>/PARSER.batch.<n>.yaml      ← per-top-level-case: holds sub-cases <n>.a, <n>.b, ...
```

仅当一个顶层用例长出子用例时才会创建 per-case 文件。一旦引入了任意子用例 `PARSER.batch.<n>.<sub>`，平铺文件中那个裸 `PARSER.batch.<n>` 就会迁移到 per-case 文件。两种文件之间用例 ID 必须唯一，这是合并不变量。

两种文件都使用相同的 schema：

```yaml
family: kimi_k2
mode: batch
cases:
  PARSER.batch.1:
    description: Single tool call (happy path)
    model_text: |-
      <|tool_calls_section_begin|>...
    tools:
    - name: ...
      parameters: {...}
    expected:
      calls:
      - name: ...
        arguments: {...}
      normal_text: ''
  PARSER.batch.8.a:                    # sub-case keys also valid
    description: Narration before tool call only
    ref: https://github.com/vllm-project/vllm/blob/<sha>/tests/tool_parsers/test_<family>_tool_parser.py#L<line>
    model_text: |-
      ...
```

`ref` 字段在 per-sub-case 文件（`PARSER.<mode>.<n>.yaml`）中必填，可取以下三种形式之一，区分 fixture 与上游的绑定强度：

- **`ref: inspired-by <url>`** —— 上游有针对同一家族同一形态的测试，但 fixture 的 `model_text` 是新写的（模板化的叙述、跨家族一致的 function/args），不是逐字拷贝。URL 用于回溯到上游测试。多数有上游对照的用例都属于这一种。
- **`ref: ported-from <url>`** —— fixture 的 `model_text` 与上游测试输入完全一致（或经过最小改动）。罕见；仅在线协议非常复杂、字节等价性重要、希望上游链接成为字面级真相时才用。
- **`ref: dynamo`** —— 在本仓库新写、没有上游对照。多数子用例分类填充（如 `.b` 仅 post、`.d` 在 calls 之间）属于这一种，因为 vLLM/SGLang 不测这些形态。

URL 形式（`inspired-by` / `ported-from`）的 URL 本身就指明了实现：`vllm-project/vllm` → vLLM，`sgl-project/sglang` → SGLang。

每个子用例都必须带这三种状态之一；不存在"无溯源"状态。旧式平铺 `PARSER.<mode>.yaml`（无子用例的）**不**带 `ref` —— 那些条目早于此约定。

#### 嵌入式的差异注释

每个登记了跨实现差异的 per-sub-case fixture（在 `test_parity_parser.py` 的 `KNOWN_DIVERGENCES` 中），其 `expected:` 后会带一段 YAML 注释块，展示发散方实际输出，并标记 `TODO(research)`：

```yaml
PARSER.batch.8.b:
  ...
  expected:
    calls: [...]
    normal_text: "Let me know if you need more."     # Dynamo's output
  # TODO(research): vllm diverges — drops trailing normal_text...
  # vllm produces:
  #   calls: [{'name': 'get_weather', 'arguments': {'location': 'Dallas'}}]
  #   normal_text: None
```

这样无需运行 harness，在 fixture 层即可看到差异，且每条都自带"是否切换"的明确提示。注释由 `embed_divergence_comments.py` 写入（必须在 regen 之后跑，因为 `yaml.dump` 在重写时会丢注释）：

```bash
PYTHONPATH=lib/bindings/python/src python3 -m tests.parity.parser.regenerate_fixtures --overwrite-if-exists
PYTHONPATH=lib/bindings/python/src python3 -m tests.parity.parser.embed_divergence_comments
```

用例 key 是 [`lib/parsers/PARSER_CASES.md`](../../lib/parsers/PARSER_CASES.md) 中的完整 ID（`PARSER.batch.1` … `PARSER.batch.10`，加上 `PARSER.batch.8.a` 这类子用例）。它们与 `KNOWN_DIVERGENCES` 的 key 以及 pytest parametrize ID 直接一致，因此一次 `grep PARSER.batch.8.a` 就能在文档、fixture 与 Rust 源码注释中找到该用例。

`model_text` 使用 YAML 字面块标量（`|-`），多行线协议（XML 风格家族、harmony）按模型实际输出读起来更直观，而不是带 `\n` 转义的单行。UTF-8 + `allow_unicode=True`，因此 DeepSeek 的特殊 token（`｜` U+FF5C，`▁` U+2581）以字面字符出现，而非转义序列。

## 为什么不同家族的 YAML 看起来很像（这正是关键）

并排打开任意两个家族文件，用例外形几乎一致：相同的 `description` 字符串、相同的 `tools` schema、相同的用例键 `"PARSER.batch.1"`–`"PARSER.batch.10"`。**这是有意为之** —— `PARSER.batch.N` 在每个家族里都是同一个逻辑场景（完整列表见上面的 [当前 parity 状态](#当前-parity-状态m2batch-模式)）。

如此一来，reviewer 可以 `grep PARSER.batch.4` 一次性看遍所有 10 个家族在同一场景下的处理。重复*就是* diff：它使逐用例的跨家族对比变得轻松。

### 不同家族会变化的部分

**1. `model_text`** —— 每个家族有自己的线协议。`case 1`（"single happy-path call"）在每个家族中分别长这样：

| family | model_text（截断） |
|---|---|
| `kimi_k2` | `<\|tool_calls_section_begin\|><\|tool_call_begin\|>functions.get_weather:0…` |
| `qwen3_coder` | `<tool_call>\n<function=get_weather>\n<parameter=location>\nNYC\n</parameter>…` |
| `glm47` | `<tool_call>get_weather<arg_key>location</arg_key><arg_value>NYC</arg_value>…` |
| `deepseek_v3_1` | `<｜tool▁calls▁begin｜><｜tool▁call▁begin｜>get_weather<｜tool▁sep｜>{…}…` |
| `harmony` | `<\|channel\|>commentary to=functions.get_weather <\|constrain\|>json<\|message\|>…` |
| `minimax_m2` | `<minimax:tool_call>\n<invoke name="get_weather">\n<parameter name="location">…` |
| `nemotron_deci` | `<TOOLCALL>[{"name": "get_weather", "arguments": {"location": "NYC"}}]</TOOLCALL>` |

**2. `expected`** —— 有时也会不同：当 Dynamo 各家族 parser 的具体行为差异让同一逻辑场景产生不同的解析输出时。例如 `case 4`（畸形输入）—— 各家族畸形方式本身不同（每种畸形都按该家族线协议自然出现），Dynamo 的恢复也不同：

```
kimi_k2/batch.4    expected.calls[0].arguments = "{\"location\":\"NYC\""
                   ↑ truncated raw string — Dynamo's kimi_k2 parser
                     surfaces the malformed bytes verbatim

qwen3_coder/batch.4 expected.calls[0].arguments = {"location": "NYC"}
                   ↑ recovered into a proper dict — Dynamo's
                     qwen3_coder parser is lenient with missing tags
```

两者都是各自家族下合法的 Dynamo 契约。跨实现差异（vLLM 与 SGLang 在同一用例上做出*不同*的事）以 `xfail` 形式记在每个测试的 `KNOWN_DIVERGENCES` 中，并附一句话原因。

**3. `tools`** —— 偶有参数命名差异（如 `city` vs `location`，是否有 `unit` 字段），是从最初塞入各家族 fixture 的 Rust 单元测试沿用下来的。

如果你在*新增*用例，请把用例形态在所有适用家族里镜像复制 —— harness 假设 case N 在各处含义一致。

## 重新生成 fixture

在已构建 `dynamo._core`（M2 的 PyO3 binding）的容器内，从仓库根目录运行：

```bash
# Default: non-destructive — new cases written, existing left alone.
python3 -m tests.parity.parser.regenerate_fixtures

# Refresh: re-run Dynamo for every case in INPUTS, overwrite on disk.
# Use this only when Dynamo's parser behavior intentionally changed.
python3 -m tests.parity.parser.regenerate_fixtures --overwrite-if-exists
```

（必须用 `-m` 调用 —— 直接运行脚本会把 `tests/parity/parser/` 放到 `sys.path`，让本地的 `dynamo.py` 包装影盖真正的 `dynamo` 包。）

重生后用 `git diff tests/parity/parser/fixtures/` 审核改动再 stage。磁盘上不在 `INPUTS` 里的用例无论是否带 flag 都始终保留，因此修改你的 `INPUTS` 段不会意外删除其他贡献者的用例。

## 未来阶段（同级目录）

```
tests/parity/
├── parser/         (today)
├── postprocess/    (future) — parser output → OpenAI wire response
└── preprocess/     (future) — request preprocessing, chat-template
```

每个阶段有自己的 fixture、wrapper 与测试文件，但复用本层级共享的 `common.py`（`ParseResult` 风格结构）与 `conftest.py`（session 级服务启动）。当前不在范围内；参见 `lib/parsers/PARSER_CASES.md`、`components/src/dynamo/frontend/tests/FRONTEND_CASES.md` 与 `lib/parsers/PIPELINE_CASES.md` 中围绕的分类法，它会指引何时增加哪个阶段。

## 最终目标：YAML fixture 作为单一事实来源

如今本 harness 的 fixture 与 `lib/parsers/src/tool_calling/*` 下手写的 Rust 单元测试有重叠 —— 对约 190 个黑盒 "给定输入 X，parser 返回 Y" 的测试，相同 input + expected 同时存在于两处（M2 fixture 最初是从那些 Rust 测试里手工提取出来的）。

预期的最终态是 **一套 fixture，多个轻 harness**，分布在后续 PR 中：

```
tests/parity/parser/fixtures/<family>/PARSER.batch.yaml
        │
        ├── Python harness (M2 / M3) — already reads it
        └── Rust harness (future)    — would read it too,
                                       at cargo-test speed,
                                       no Python required
```

这样的好处：

- **不再重复测试数据。** 在 YAML 中加一条用例，立即覆盖 Dynamo（Rust harness）、Dynamo-via-PyO3（M2）以及 vLLM/SGLang 服务（M3）。如今想要跨实现覆盖，新增 Rust 测试还得手工镜像到 M2 的 `INPUTS`。
- **Rust 开发者保留快速反馈循环。** `cargo test` 仍能在约 0.5 s 内完成；不需要 Python 构建。
- **每种实现都用其原生语言测试。** 比为了断言一个 Rust 契约而绕道 PyO3 更贴近生产语义。

迁移后保留为 Rust 专属测试的：

- 内部 helper 的白盒测试（`detect_tool_call_start_*`、`find_tool_call_end_position_*`、regex 回退路径）。它们测的是 parser 内部状态，不是 parity，也不通过 PyO3 暴露。约 498 个 Rust 测试中约 120 个属此类。
- tokenizer / config / panic 类测试。本质上是单实现。

工作量草图（M2 + M3 落地后的独立 PR）：

- **PR-X：** Rust harness 读取 `PARSER.batch.yaml`，分发到 `try_tool_call_parse_<family>(...)` 并断言 `expected`。约 1-2 天。
- **PR-Y：** 机械迁移 —— 删除约 70 个手写、与共享 fixture 冗余的黑盒 Rust 测试。约 6 小时。
- **PR-Z：** 同样形态扩展到格式条件式与客户事故回归（约多 150 个用例）。约 2-3 天。
- （延后）流式变体。需要新的 `PARSER.stream.*` schema + 流式 PyO3 + 流式 Rust harness。规模与最初的 M3 工作相当。

在那之前，M2 与 Rust 套件都存在；对约 100 个用例，它们通过不同表面测试同一个 Dynamo 契约。M2 真正的增量价值在于跨实现那一半（vLLM 与 SGLang）。

## 添加新的 parser 家族

1. 在 Dynamo 的 parser 注册表（Rust 侧）添加家族名。
2. 跑 M2 现有测试 —— 该家族的 `dynamo` wrapper 测试会因为还没 fixture 而失败。
3. 在 `regenerate_fixtures.py` 的 `INPUTS` 中为每个 `(family, "PARSER.batch.<n>")` 你想覆盖的用例加一段（从已有家族镜像用例形态）。
4. 跑重生器物化 `<family>/PARSER.batch.yaml`。
5. 把家族对应的 vLLM 与 SGLang 派发条目加入 `_FAMILY_TO_VLLM_KEY`（`vllm.py`）与 `_FAMILY_TO_SGLANG_DETECTOR`（`sglang.py`）。
6. 跑 pytest。任意跨实现差异都会以失败暴露 —— 分类、给 `KNOWN_DIVERGENCES` 加一句话原因，测试就转为 xfail。
