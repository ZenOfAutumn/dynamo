<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# 源代码中的 Linear 工单引用

本仓库中的源代码不得包含对内部 Linear 工单（`DIS-XXXX`、`DYN-XXXX`，或任何形如 `<PROJECT>-NNNN` 的 Linear ID）的引用。Linear 是 NVIDIA 内部系统，外部贡献者和公共用户无法解析这些 ID。

## 当被标记时该怎么做

1. **先查找已存在的 GitHub issue**，看是否已有对应工作的跟踪 —— 通常在仓库的 *Issues* 标签页中搜索就足够了。
2. **如果没有，则新建一个 GitHub issue**，使用面向公众友好的摘要。不要在公共 issue 正文中包含 Linear URL 或 ID；提交时把 issue 分配给自己。
3. **将源码中的 Linear 引用替换为 GitHub issue 编号**（在标识符和代码注释中使用 `GH-NNNN`，在普通文字中使用 `#NNNN`）。
4. **在 PR 描述中同时携带两种引用**（`Closes #NNNN`，并附一句简短说明 "存在对应的内部 Linear 工单"），并在 Linear 工单上反向链接到该 GitHub issue，使两者保持关联。

## 范围

适用于工作树（working tree）—— 即所有会进入 `main` 的内容。**不**适用于 commit 信息正文、分支名或 PR 标题/描述。

Markdown 文档（`*.md`）允许引用 Linear 工单以提供内部上下文，但若存在对应的 GitHub 链接，建议同时附上。
