# 转换规则（Conversion Rules）

这些规则适用于 `deploy/operator/api` 下的 API 转换代码与测试。任意版本中的任何 API 类型变更都必须更新转换代码/测试，或显式说明为何不影响转换。

## 结构

顶层 API 对象实现 `sigs.k8s.io/controller-runtime/pkg/conversion.Convertible`：

```go
type Convertible interface {
	runtime.Object
	ConvertTo(dst Hub) error
	ConvertFrom(src Hub) error
}
```

顶层方法仅接受 `Convertible` 的参数：

```go
func (src *DynamoWidget) ConvertTo(dstRaw conversion.Hub) error
func (dst *DynamoWidget) ConvertFrom(srcRaw conversion.Hub) error
```

它们的调用栈是：

1. 将 Kubernetes metadata 从 `src` 拷贝到 `dst`。
2. 将稀疏的 spec/status 载荷解码成有类型的 `restored` 值。
3. 从 `dst` 上清除私有的 conversion 注解。
4. 调用有类型的 spec/status 转换函数。
5. 从 `restored` 还原仅在目标版本中存在的字段。
6. 在有类型的 `save` 值中收集仅在源版本中存在的字段。
7. 将非空的 `save` 值编码为稀疏 spec/status 载荷写到 `dst`。

示例：

```text
(*DynamoWidget).ConvertTo(dstRaw)
  ConvertFromDynamoWidgetSpec(&src.Spec, &dst.Spec, restored.Spec, &save.Spec, ctx)
    ConvertFromDynamoWidgetNestedSpec(&src.Spec.Nested, &dst.Spec.Nested, restoredNested, saveNested, ctx)
  ConvertFromDynamoWidgetStatus(&src.Status, &dst.Status, restored.Status, &save.Status)
  saveDynamoWidgetAnnotations(save, dst)
```

转换算法对 API Go 类型进行归纳式遵循：

- 转换函数从 `v1alpha1` 导出。
- 转换函数以 v1alpha1 类型命名：

```go
func ConvertFromDynamoWidgetSpec(
	src *DynamoWidgetSpec,
	dst *v1beta1.DynamoWidgetSpec,
	restored *v1beta1.DynamoWidgetSpec,
	save *DynamoWidgetSpec,
	ctx DynamoWidgetSpecConversionContext,
) error

func ConvertToDynamoWidgetSpec(
	src *v1beta1.DynamoWidgetSpec,
	dst *DynamoWidgetSpec,
	restored *DynamoWidgetSpec,
	save *v1beta1.DynamoWidgetSpec,
	ctx DynamoWidgetSpecConversionContext,
) error
```

- `ConvertFrom` 表示 v1alpha1 -> hub。
- `ConvertTo` 表示 hub -> v1alpha1。
- 转换函数名中不要包含 `V1alpha1`。
- 不要添加另起名字的 wrapper 转换函数。
- 参数顺序固定：`src`、`dst`、`restored`、`save`、`ctx`。
- 仅在需要时才包含 `restored`、`save`、`ctx` 与 `error`。
- `ctx` 始终放在最后。
- 对不会失败的转换不返回值。
- 当转换函数会解析 JSON、校验保留载荷或可能拒绝输入时返回 `error`。
- `dst` 由调用方拥有且非 nil。
- 调用方显式分配嵌套 `dst` 对象。
- 转换函数不接受 `**T`。
- 不要通过把 `dst` 赋为 nil 来表达"缺失"。
- 转换函数只完成自身 API 结构的转换。
- 每个嵌套 API 结构都拥有自己一对系统化的 `ConvertFrom<SubStruct>` / `ConvertTo<SubStruct>` 转换函数。
- 父级转换函数应调用这些嵌套转换函数，而不是把嵌套结构的转换内联展开。

允许的局部辅助函数：

- 用于解码稀疏载荷的 `restore*` 读取函数。
- 用于稀疏载荷的 `save*` 写入函数。
- 用于稀疏 save 载荷的 `ensure*` 分配函数。
- 无副作用的谓词。
- 字段组的投影/分解，例如 pod template 的组合/拆解。

## 不变量（Invariants）

- `src` 的活字段始终是真相来源（source of truth）。
- 一个被表达的字段总是来自 `src`，包括 nil、空、false 与零值。
- `restored` 数据仅用于 `src` 无法表达的目标字段。
- `save` 数据仅包含 `dst` 无法表达的源字段。
- 不可表达的数据只能通过该类型的私有稀疏 spec 与 status 载荷保留。
- 新的转换风格允许每种类型最多两个私有 conversion 注解：一个 spec 载荷注解、一个 status 载荷注解。
- 不要添加按字段、按列表、按子对象或其他形式的转换注解。
- 不要将保留数据覆盖到已转换的活字段上。
- 不要将保留数据用作被表达字段的 fallback。
- 不要修改 `src`。

## 稀疏载荷（Sparse Payloads）

- 载荷是源版本中有类型的 API 结构。
- 载荷在结构上必须是稀疏的。
- 仅保存目标版本无法表达的源字段。
- 仅保存把保留字段映射回真实源结构所需的最小上下文。
- 跳过空的 save 对象。
- 除非嵌套 save struct 至少包含一个被保存的字段，否则不要分配它。
- 载荷注解常量保持非导出，并仅出现在 conversion 文件中。
- 仅 API 转换代码与转换测试可以知晓载荷注解或载荷形态。
- controller、reconciler、webhook、内部 helper 与一般 API helper 不得读取、写入、删除、过滤、解码、编码、判断、暴露载荷注解或载荷。
- 非转换代码可以不透明地拷贝 Kubernetes metadata，但不得识别 conversion 注解。
- 如果非转换代码需要稀疏载荷中的数据，应添加或调用真正的转换 helper。
- restore 代码可以从稀疏载荷读取被表达的字段，但只能作为匹配上下文。
- restore 代码不得从稀疏载荷还原被表达的字段。

## 复合值（Compound Values）

- 按叶子（leaf-by-leaf）还原复合值。
- 不要先还原整个 pod template、container、job、resource requirements 之类的复合对象，再把被表达字段打补丁覆盖回去。
- 对于投影字段，将活的目标对象与投影对象进行比较，仅保存不可表达的剩余部分。
- 把稀疏匹配/分解的上下文保留在 spec/status 载荷之内。
- 稀疏上下文不是真相来源。

## 命名列表（Named Lists）

- 用声明的 list-map key 来匹配保留的 list-map 条目，而不是用切片索引。
- 忽略其 key 已不存在于活 `src` 中的保留条目。
- 新的活条目不会获得保留数据，除非稀疏载荷里有相同的 key。
- 命名列表中保存的条目必须包含 list-map key，例如 DGD 组件中的 `ComponentName`。

## 旧 DGDR 例外

- 不要新增遗留格式。
- 已有的 legacy DGDR 注解只是临时的降级垫片。
- 保留 legacy 键命名为 `legacyAnn*`。
- 保留它们的 `TODO(sttts)` 移除注释。
- 将 legacy 读写隔离在 legacy helper 中。
- 将 legacy 数据解码到与结构化转换相同的有类型 `restored` 模型。
- legacy 数据只是旧值缓存。
- legacy 数据不得覆盖活 `src` 可表达的字段。
- 在仍需要降级兼容期间，把旧的 converter 保留作为测试 oracle。
- legacy-vs-structural 的 fuzz 测试可以将输入归一化为旧 converter 可表达的行为。
- 主 round-trip fuzz 测试不得为已修复的结构化转换 bug 保留 workaround。

## API 变更

对每一个新增、删除、重命名或语义变化的 API 字段，明确选择一种转换策略：

- 原生映射：从活 `src` 转换。
- 仅源端保留：保存到 `dst` 的源版本稀疏载荷。
- 仅目标端还原：在活字段转换后从 `restored` 还原。
- 派生或有损映射：记录映射并添加针对性测试。
- 故意丢弃：说明为什么超出转换契约，并在可观察时添加针对性测试。

如果对方版本之后为同一概念新增了原生字段：

- 活 `src` 优先于过期的稀疏载荷数据；
- 稀疏载荷停止保留这个现已可表达的值。

`TestV1Beta1ConversionFieldSetIsAcknowledged` 是针对所列 v1beta1 根类型的 schema 变更绊线（tripwire）：

- 不要先更新 `knownV1Beta1ConversionFieldSet`。
- 先实现转换决策。
- 添加或更新针对性测试。
- 然后再根据失败测试的 diff 刷新快照。

## 可变性（Mutability）

- 转换不得修改 `src`。
- 在赋值后将共享底层数据视为只读时，可以在 `src` 与 `dst` 之间共享。
- 不要仅因为某个值来自 `src` 就深拷贝。
- 在编辑、追加、排序、归一化、清空、合并或以其他方式修改来自 `src` 的数据之前必须先深拷贝。
- 在从 `src` 共享赋值后再通过 `dst` 修改，仍违反此规则。

## 测试

转换变更需要的覆盖：

- 为每条新增或变化的转换策略添加针对性测试。
- 发现的转换回归在相关 `*_bugs_test.go` 中以针对性测试覆盖。
- 常规 round-trip fuzz：

```text
hub -> spoke -> hub
spoke -> hub -> spoke
```

- 针对过期稀疏载荷行为的可变性 fuzz：

```text
fuzz in
convert to other
mutate other without deleting private spec/status annotations
convert other -> in -> other2
compare other and other2, ignoring only private spec/status annotations
```

- 变更必须触及嵌套已有对象、数组与切片。

## 评审检查清单

- 每个被表达字段都来自活 `src`。
- 每个被还原字段都不可由活 `src` 表达。
- 每个被保存字段都不可由 `dst` 表达。
- save 载荷是稀疏的。
- 稀疏上下文保留在 spec/status 载荷之内。
- 稀疏上下文仅用于匹配。
- 命名列表按 list-map key 匹配。
- 复合值按叶子还原。
- 在测试需要的位置保留 nil 与空形态。
- 不修改 `src`。
- 除 spec/status 之外没有新增 conversion 注解。
- 非转换代码不解释稀疏载荷。

## 验证

转换变更需运行：

```sh
GOCACHE=/tmp/dynamo-go-cache go test ./api/v1alpha1 -count=1
GOCACHE=/tmp/dynamo-go-cache go test ./api/... -count=1
GOCACHE=/tmp/dynamo-go-cache go test ./api -run TestFuzzRoundTrip -roundtrip-fuzz-iters=3000 -count=1 -v
```
