# 多模态 JSONL 请求生成器

为 [aiperf](https://github.com/ai-dynamo/aiperf) 生成单轮多模态请求（文本 + 图像）的
`.jsonl` 基准测试文件。

## 关键概念：图像池复用

每个请求都从一个固定的图像池中采样图像。相对于总图像位数（image slots），
更小的图像池会带来更多跨请求的图像复用 —— 这对衡量嵌入缓存（embedding cache）
命中率非常有用。

例如：500 个请求 x 每个请求 3 张图 = 1500 个图像位。当 `--images-pool 200` 时，
许多请求会共享同一组图像。

## 图像模式

| 模式 | `--image-mode` | JSONL 中存放的内容 | 由谁拉取图像 |
|------|---------------|------------------------|----------------------|
| base64（默认） | `base64` | 本地 PNG 文件的绝对路径 | 由 aiperf 读取并在发送前进行 base64 编码 |
| HTTP | `http` | COCO test2017 的 URL | 由 LLM 服务器自己下载图像 |

`http` 模式下需要先下载 COCO 标注：
```bash
mkdir -p annotations && cd annotations
wget http://images.cocodataset.org/annotations/image_info_test2017.zip
unzip image_info_test2017.zip
```

## 用法

```bash
# 默认值：500 个请求，每个请求 3 张图，全部唯一，base64 模式
python main.py

# 使用 COCO URL 的 HTTP 模式
python main.py --image-mode http

# 控制复用：200 个请求，池中 100 张唯一图像
python main.py -n 200 --images-pool 100

# 每个请求更多图像
python main.py -n 100 --images-per-request 20 --images-pool 500
```

输出文件名会编码各项参数，例如 `500req_3img_200pool_300word_http.jsonl`。

## 配合 aiperf 运行

```bash
aiperf profile \
  --model Qwen/Qwen3-VL-30B-A3B-Instruct-FP8 \
  --input-file 500req_3img_200pool_300word_http.jsonl \
  --custom-dataset-type single_turn \
  --shared-system-prompt-length 1000 \
  --extra-inputs "max_tokens:500" \
  --extra-inputs "min_tokens:500" \
  --extra-inputs "ignore_eos:true"
```

注意：JSONL 中包含的是真实内容（文本 + 图像引用），并非 token 数量。请勿传入
`--isl` —— 它仅适用于合成数据生成。
