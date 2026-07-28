# 生成 aiperf 源图像

aiperf 内置的图像生成器自带的源图像非常少。在使用 `--image-mode base64`
进行基准测试时，aiperf 会从它的 `assets/source_images/` 目录中挑选图像 ——
图像数量太少意味着每个请求都会发送几乎相同的图像，这无法真实地对多模态
流水线施加压力。

本脚本会向该目录填充 200 张随机噪声 PNG 图像，使 aiperf 拥有更大的采样池。

## 用法

```bash
python main.py
```

图像会直接写入到 aiperf 已安装的 `source_images/` 目录。
