# DeepSeek R1 SGLang 部署示例

本示例用于以解耦（disaggregated）模式使用 SGLang 运行 DeepSeek R1。它基于 SGLang 团队提供的 WideEP 示例。

## 容器

使用 `build.sh` 脚本构建容器：

```bash
./container/build.sh --framework SGLANG
```

需要使用 `1b3eed4b6a0e735d4ecec6681f4c0b89f2112167`（2025 年 9 月 18 日）之后的 Dynamo commit。

## 硬件

提供两套部署示例：16x H200（disagg-8gpu）以及 32x H200（disagg-16gpu）。文件夹名称中的数字（8 或 16）表示每种 worker 所使用的 GPU 数量，prefill（预填充）与 decode（解码）worker 各使用对应数量的 GPU。该示例同样适用于其他 GPU SKU，请根据 GPU 容量相应调整 TP 和 EP 大小。

如果在向引擎发送请求时看到 NCCL 错误，通常是由 OOM（内存不足）引起的。可以尝试降低 prefill 与 decode 引擎中的 `--mem-fraction-static`。

