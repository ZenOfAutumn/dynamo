# AGENTS.md

在修改本目录或其下任意版本化 API 包中的 API 代码或 API 测试之前，请先阅读
`CONVERSION.md`。

任何版本中的每一次 API 类型变更都必须同步更新对应的转换代码与转换测试，或显式
说明为什么不会影响转换。

请遵循其中关于 hub/spoke 转换、稀疏注解保留（sparse annotation preservation）、
live-source 优先级、结构性辅助函数命名以及 fuzz 覆盖率的不变量约束。
