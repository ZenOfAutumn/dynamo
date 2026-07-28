# Python 指南

本仓库 Python 代码的规则与约定。

## 关键规则

这些是对代码质量与可维护性最为重要的约定。
存在例外，但应当少见，且要在代码注释中说明理由。

### 把 import 写在文件顶部

**始终标记**任何出现在函数体、方法或类内部的 `import` 语句。函数内部的 import 会隐藏依赖、让模块难以一眼看懂，并且会把缺失的包问题掩盖到运行时某条具体路径才暴露。

**始终标记**包裹 import 的 `try/except ImportError`（除下文记录的例外情况）。这种模式会形成两种执行模式 —— 一种装了库、一种没装 —— 把测试面翻倍，并在依赖意外缺失时产生混乱行为。

```python
# BAD -- import inside function; dependency is invisible until this path runs
# ALWAYS FLAG THIS PATTERN
def process_data():
    import json
    return json.loads(data)

# BAD -- try/except hides a missing dependency behind a flag
try:
    import requests
    HAS_REQUESTS = True
except ImportError:
    HAS_REQUESTS = False

# GOOD -- direct import at module top; fails immediately if missing
import json
import requests
```

让缺失依赖立即报错，胜过把问题藏起来直到用户在生产环境踩到一条隐蔽的代码路径。如果某个包是必需的，请把它加进 `requirements.txt` 或 `pyproject.toml`，确保安装一步到位。

**例外：可选依赖与 pytest 收集。** 本仓库存在多个可选后端（vLLM、SGLang、TRT-LLM、cupy 等），并不会在每种环境中都安装。以下情况下 `try/except ImportError` 是正确做法：

- 一个可选后端依赖（如 `tritonclient.grpc`、`vllm_omni`、`torch_memory_saver`）可能未安装，代码提供了回退或将 import 设为 `None`。
- 测试文件需要在缺少可选包时跳过收集（如 `except ImportError: pytest.skip(..., allow_module_level=True)`）。
- 标准库模块在不同 Python 版本上可用性不同（如 `tomllib` 在 3.11+ 上 vs 回退到 `tomli`）。

```python
# OK -- optional backend, graceful fallback to None
try:
    from vllm_omni.diffusion.data import DiffusionParallelConfig
except ImportError:
    DiffusionParallelConfig = None

# OK -- skip test collection when optional deps are missing
try:
    from dynamo.profiler.rapid import WorkloadSpec
except ImportError as e:
    pytest.skip(f"Skip (missing dependency): {e}", allow_module_level=True)

# OK -- stdlib version compatibility
try:
    import tomllib
except ImportError:
    import tomli as tomllib
```

### 偏向快速失败，而非掩盖错误

来自 PEP 20（The Zen of Python）：

> *"Errors should never pass silently. Unless explicitly silenced."*
> *"Explicit is better than implicit."*

出错时立即失败，胜过带着错误状态默默继续，因为：

- 触发错误的人可以立刻看到，上下文还很新鲜。
- 调用栈直接指向根本原因，而不是三层调用之外的下游表象。
- 隐藏的错误会复合 —— 一处吞掉的异常会在别处产生让人困惑的行为，调试成本随距原始失败的距离指数级增长。

```python
# BAD -- all of these hide errors
except Exception:
    pass

except Exception as e:
    logging.error(e)        # logs but silently continues!
    return []               # returns a fake default

# BAD -- bare except catches KeyboardInterrupt and SystemExit too,
# making the process impossible to kill with Ctrl-C or sys.exit()
try:
    do_work()
except:
    log_error()

# GOOD -- just let it crash
result = something()

# GOOD -- catch SPECIFIC exceptions you can actually handle
try:
    result = json.loads(text)
except json.JSONDecodeError:
    result = {}

# GOOD -- if you must catch broad, catch Exception (not bare except)
# and re-raise after logging
try:
    result = something()
except Exception as e:
    logger.error(f"Failed: {e}")
    raise
```

**三条规则：**
1. 能去掉 try/except 就去掉 —— 让它直接崩。
2. 只捕获**具体的**异常（`FileNotFoundError`、`ValueError`、`json.JSONDecodeError` 等）。
3. 如果必须宽泛捕获，使用 `except Exception:`（绝不用裸 `except:`），并在日志后**始终**重新 raise。

### 不要在已知类型上做防御式 `getattr()`

**始终标记** `getattr(obj, "attr", default)`，当对象类型已知、属性又是该类型定义的一部分时（类属性、`__init__` 参数、dataclass 字段等）。带默认值的 `getattr()` 会在属性本应始终存在时悄悄返回回退值，从而掩盖 bug。直接属性访问会在类型契约改变时大声失败 —— 这正是你想要的。

```python
# BAD -- cfg is a ServiceConfig with host/port; getattr hides AttributeError
# ALWAYS FLAG THIS PATTERN
cfg = ServiceConfig(host="0.0.0.0", port=8080)
host = getattr(cfg, "host", "localhost")
port = getattr(cfg, "port", 9999)

# GOOD -- direct access, fails loudly if something is wrong
host = cfg.host
port = cfg.port
```

---

## 必须在评审中标记的反模式

下面每一项都是**强制性的评审检查**。如果 PR 中出现这些模式，请标记并要求修改。这不是风格偏好 —— 它们是真实 bug、资源泄漏与 CI 不稳定的来源。

### 可变默认参数

**始终标记**任何默认参数为可变对象（`[]`、`{}`、`set()`）的函数。默认值在函数定义时只求值一次，并被所有调用共享，于是修改会在多次调用之间悄悄累积。

```python
# BAD -- the list is shared across all calls; flag this
def add_item(item, items=[]):
    items.append(item)
    return items

add_item("a")  # ["a"]
add_item("b")  # ["a", "b"] -- not ["b"]!

# GOOD -- use None sentinel, create a new list each call
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

### 文件句柄泄漏 —— 始终使用上下文管理器

**始终标记**任何未被 `with` 包裹的 `open()` 调用。文件、网络连接、子进程与锁都必须用 `with` 打开，这样即使发生异常也会被释放。裸 `open()` 后再手动 `.close()`，一旦在两次调用之间抛出异常，就会泄漏句柄。

```python
# BAD -- file handle leaks if json.load raises; flag this
f = open("data.json")
data = json.load(f)
f.close()

# GOOD
with open("data.json") as f:
    data = json.load(f)
```

在本项目中尤其重要，因为测试要管理子进程、etcd/NATS 连接与临时目录。请使用 `ManagedProcess`、`tempfile.TemporaryDirectory` 等上下文管理器，而不是手动 setup/teardown。

### 遮蔽内置名

**始终标记**任何名为 `list`、`dict`、`id`、`type`、`input`、`open`、`format`、`set`、`map`、`filter`、`range`、`str`、`int`、`float`、`bool`、`bytes`、`tuple`、`hash`、`len`、`min`、`max`、`sum`、`any`、`all`、`zip`、`enumerate`、`sorted`、`reversed` 或 `next` 的变量。给这些名字赋值会覆盖内置，并在同一作用域内之后引发让人困惑的 `TypeError`。

```python
# BAD -- shadows built-in list(); flag this
list = get_items()
filtered = list(some_gen)   # TypeError: 'list' object is not callable

# GOOD
items = get_items()
filtered = list(some_gen)
```

### 与 None / True / False 比较使用 `is`

**始终标记** `== None`、`== True`、`== False`、`!= None`、`!= True`、`!= False`。这些会调用 `__eq__`，可被覆盖并产生意外结果。单例比较请使用 `is` / `is not`。

```python
# BAD -- flag these
if result == None:
if flag == True:
if done == False:

# GOOD
if result is None:
if flag is True:     # or just: if flag:
if not done:
```

### 不要在迭代中修改集合

**始终标记**任何在迭代过程中向被迭代集合中增、删元素的循环。这会跳过元素、对 dict 触发 `RuntimeError`，或导致死循环。请构造新集合或在副本上迭代。

```python
# BAD -- RuntimeError on dict, skips elements on list; flag this
for item in items:
    if item.is_stale():
        items.remove(item)

# GOOD
items = [item for item in items if not item.is_stale()]
```

### 循环中字符串拼接，优先用 `join()`

**始终标记**循环内对字符串变量的 `+=`。在字符串上反复 `+=` 每次都会创建新字符串对象，对大循环而言是 O(n^2)。

```python
# BAD -- O(n^2) string building; flag this
result = ""
for line in lines:
    result += line + "\n"

# GOOD -- O(n) with join
result = "\n".join(lines)
```

### 循环中的延迟绑定闭包

**始终标记**在循环中创建、引用循环变量但未通过默认参数绑定的 lambda 或内嵌函数。闭包捕获的是变量引用，而非创建时的值，因此所有闭包最终都会拿到循环结束时的值。

```python
# BAD -- all lambdas return 4 (the final value of i); flag this
fns = [lambda: i for i in range(5)]
[f() for f in fns]  # [4, 4, 4, 4, 4]

# GOOD -- default argument captures current value
fns = [lambda i=i: i for i in range(5)]
[f() for f in fns]  # [0, 1, 2, 3, 4]
```

### 不要把 `assert` 用于运行时校验

**始终标记**用于校验函数参数、请求负载、配置或任何来自函数外部数据的 `assert`。Python 在 `-O`（optimize）下运行时会去掉 assert，悄无声息地移除校验。必须始终执行的检查请使用显式的 `if/raise`。

```python
# BAD -- silently skipped under python -O; flag this
assert user_id is not None, "user_id required"

# GOOD
if user_id is None:
    raise ValueError("user_id required")
```

---

## 代码风格

- 遵循 PEP 8。
- 变量与函数使用 `snake_case`。
- 类使用 `PascalCase`。
- 在能提升可读性的地方加上类型注解。
- 公共函数与类使用 docstring。
- 当结构有 4 个以上字段时使用 `dataclass` 而非纯 dict（更好的类型推断与 IDE 支持）。

## 文件组织

- `__init__.py` 用于包初始化。
- 模块之间清晰分离。
- 测试放在 `tests/` 目录。

## 格式化与 Lint

### 推荐工作流

先自动修复格式，再 lint：

```bash
ruff format <touched_paths>
ruff check --fix <touched_paths>
```

或使用 pre-commit（会运行 isort、black、flake8、ruff 等）：

```bash
pre-commit run --files <touched_files>
pre-commit run --all-files    # for broad changes
```

### Pre-commit 钩子

仓库的 `.pre-commit-config.yaml` 会运行下列 Python 钩子：

- **isort** —— import 排序（`profile = "black"`，在 `pyproject.toml` 中配置）
- **black** —— 代码格式化
- **flake8** —— 风格检查（`max-line-length=88`）
- **ruff** —— 带自动修复的快速 lint
- **codespell** —— 拼写检查
- **trailing-whitespace**、**end-of-file-fixer**、**check-yaml**、**check-json**、**check-toml**

### 提交前

始终运行：

```bash
pre-commit run --files <changed_files>
```

变更范围更大或不确定时：

```bash
pre-commit run --all-files
```

### 缩进验证

缩进错误常见且不易肉眼发现。编辑 Python 文件后请机械地校验：

```bash
ruff format <touched_paths>                     # auto-fix (preferred)
python3 -m compileall -q <touched_paths>        # fast parse-only check
```

修复缩进错误时，始终读 20–30 行的上下文并修整整个块 —— 邻近行常常犯同样的错。

## 错误处理

完整策略见上文 **关键规则** 一节。摘要：

- 默认让异常向上传播。
- 仅捕获你确实能处理的具体异常。
- 如果你捕获了 `Exception`，必须在日志后重新 raise。
- 永远不要 `except Exception: pass`。

### 正则注意事项

注意 raw string 中的转义（`\s` 与 `\\s`）。修改关键正则时，加一行测试以证明它能匹配。

## Import 顺序

import 顺序由 isort 按 `profile = "black"` 排序（在 `pyproject.toml` 中配置）。
顺序为：

1. 标准库
2. 第三方（已知：`vllm`、`tensorrt_llm`、`sglang`、`aiconfigurator`）
3. 第一方（`dynamo`、`deploy`）

运行 `isort` 或 `pre-commit run isort` 进行自动排序。
