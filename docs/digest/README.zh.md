# 添加 Digest 帖子

按照以下步骤在 Dynamo 文档站点上发布一篇新的 Digest 帖子。

## 步骤 1：撰写 Digest 帖子

在该目录（`docs/digest/`）下新建一个 Markdown 文件：

```text
docs/digest/my-post-slug.md
```

文件名使用 kebab-case。文件名将作为 URL 的 slug
（例如，`my-post-slug.md` 提供于 `/dynamo/dev/digest/my-post-slug`）。

在文件顶部添加 frontmatter：

```yaml
---
title: Your Digest Post Title
description: A one-sentence summary shown in search results and social previews.
---
```

## 步骤 2：将帖子加入导航

打开 `docs/index.yml` 并在 **Digest** 章节下添加一条 page 条目：

```yaml
  - section: Digest
    path: digest/index.mdx
    slug: digest
    contents:
      - page: Your Digest Post Title
        path: digest/my-post-slug.md
```

每篇新帖在 `contents` 列表中获得一个 `- page:` 条目。`page`
的值是侧边栏中显示的名称；`path` 指向你的文件。

## 步骤 3：在着陆页中添加一张卡片

打开 `docs/digest/index.mdx`，在已有的 `<CardGroup>` 中添加一个 `<Card>`：

```mdx
<Card
  title="Your Digest Post Title"
  icon="regular newspaper"
  href="/dynamo/dev/digest/my-post-slug"
>
  A brief summary of what this post covers (1-2 sentences).
</Card>
```

### 卡片字段

| 字段  | 必填 | 描述                                              |
|--------|----------|----------------------------------------------------------|
| title  | 是      | 卡片上显示的帖子标题                         |
| icon   | 否       | Font Awesome 图标 class（例如 `regular bolt`）           |
| href   | 是      | 以 `/dynamo/dev/digest/` 开头的 URL 路径             |
| (body) | 是      | Card 标签内的简短摘要文本                  |

`<CardGroup cols={2}>` 包裹器已在 `index.mdx` 中存在。只需把你的
`<Card>` 加在其中即可。

可在 https://fontawesome.com/icons （免费版）浏览图标。

## 步骤 4（可选）：将帖子加入导航栏下拉菜单

打开 `fern/docs.yml`，在 `navbar-links` 的 Digest 下拉菜单下添加一条链接：

```yaml
navbar-links:
  - type: dropdown
    text: Digest
    links:
      - text: All Posts
        href: /dynamo/dev/digest
      - text: Your Digest Post Title
        href: /dynamo/dev/digest/my-post-slug
```

只在此处加入精选 / 近期帖子。下拉菜单应保持精简（最多 5 项）。
不再精选的帖子可以从下拉菜单中移除，但仍可通过着陆页和侧边栏访问。

## 速查清单

- [ ] 在 `docs/digest/` 下创建 Digest 帖子的 `.md` 文件
- [ ] 在 `docs/index.yml` 的 Digest 章节下添加 page 条目
- [ ] 在 `docs/digest/index.mdx` 中添加 Card
- [ ] （可选）在 `fern/docs.yml` 的 Digest 下拉菜单中添加链接
