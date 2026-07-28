# 工作流模板（Workflow Templates）

本目录包含可复用的 GitHub Actions 工作流模板与工具。

## 文件

### akamai-eccu-flush.xslt

用于生成 Akamai ECCU（Edge Content Control Utility）XML 请求的 XSLT 模板。

**用途**：生成向 Akamai CDN 提交的缓存失效请求 XML。

**用法**：
```bash
xsltproc --stringparam target-path "path/to/flush" \
  akamai-eccu-flush.xslt akamai-eccu-flush.xslt > eccu-request.xml
```

**使用方**：`.github/workflows/publish-s3.yml`，在文档发布后用于刷新 CDN 缓存。

该模板生成一个嵌套的 XML 结构，使用 `match:recursive-dirs` 元素来表示需要在 Akamai 缓存中失效的目录路径。
