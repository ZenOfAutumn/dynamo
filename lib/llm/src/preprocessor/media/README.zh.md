# 前端中的媒体解码

本组件负责媒体下载、base64 解码、媒体解码以及 NIXL 注册。当前它被用在 OpenAI
预处理器（preprocessor）中，将多模态输入（image_url、video_url、audio_url）
转换为完全解码后的数据（pixel values 等），供后端（backend）通过 NIXL 访问。

## 用法

媒体解码在注册 MDC 时启用：

设置 HTTP 下载选项：

```python
from dynamo.llm import MediaFetcher
fetcher = MediaFetcher()
fetcher.user_agent("dynamo")
fetcher.timeout_ms(15000)
fetcher.allow_direct_ip(True)
fetcher.allow_direct_port(False)
fetcher.allowed_media_domains(["google.com"])
```

设置媒体解码的默认选项与限制：

```python
from dynamo.llm import MediaDecoder
decoder = MediaDecoder()
decoder.enable_image({"limits": {"max_image_width": 4096, "max_image_height": 4096, "max_alloc": 16*1024*1024}})
decoder.enable_video({"fps": 2.0, "max_frames": 128, "limits": {"max_alloc": 1024*1024*128*3}})
```

如果未调用 `enable_image` 或 `enable_video`，包含相应模态的请求将被拒绝。

按常规方式注册 LLM，并附加媒体相关配置：

```python
register_model(
  ...,
  media_decoder=decoder,
  media_fetcher=fetcher,
)
```


## 已知限制

> [!WARNING]
> **与 `Dockerfile.frontend` 不兼容**：使用 `Dockerfile.frontend` 时不支持前端
> 媒体解码。基于 `Dockerfile.frontend` 构建出的前端镜像不包含所需的 NIXL/UCX
> 依赖。

> [!WARNING]
> **需要 GPU 节点**：前端必须运行在具备 GPU 访问权限的节点上。在媒体处理过程
> 中，解码后的张量会通过 NIXL 写入 GPU 内存，这需要系统提供 `libcuda.so.1`。
> 在仅 CPU 的节点上运行前端会失败，错误信息形如：
> `Failed to initialize required backends: [UCX: No UCX plugin found]`。

> [!WARNING]
> **视频解码**：视频解码需要通过 `dynamo-llm/media-ffmpeg` Rust feature 启用。
> 系统上需提供以下 ffmpeg 动态库：`libavcodec`、`libavdevice`、`libavfilter`、
> `libavformat`、`libswresample`、`libswscale`。这些库会在
> `container/context.yaml` 中将 `enable_media_ffmpeg` 设为 true 时渲染出的
> dynamo dockerfiles 中提供。

## 图像解码选项

### 限制项（不可在运行时通过 `media_io_kwargs` 覆盖）
- **limits.max_image_width**（uint32，> 0）：若图像宽度超过该值，则中止解码。
- **limits.max_image_height**（uint32，> 0）：若图像高度超过该值，则中止解码。
- **limits.max_alloc**（uint64，> 0）：解码器允许的最大总分配（RAM）字节数。

## 视频解码选项
### 采样
有两种方式可配置视频采样：固定帧数采样，或按 FPS 采样。两种模式下，被采样的帧
都会均匀分布。

- **num_frames**（uint32，> 0）：尝试从输入视频中精确解码这么多帧。
- **fps**（float32，> 0）和可选的 **max_frames**（uint32，> 0）：尝试按指定帧率
  解码，并可对解码总帧数设置上限。

### 其他
- **strict**（bool）：开启严格模式时，任何被请求帧解码失败都会中止整个视频解码
  并报错。关闭严格模式时，某些被请求帧可能解码失败，最终得到的解码帧集合可能
  少于预期。

### 限制项（不可在运行时通过 `media_io_kwargs` 覆盖）
- **limits.max_alloc**（usize，> 0）：若解码帧的总字节数会超过该值，则中止解码。


## 运行时媒体解码选项（`media_io_kwargs`）

解码器的参数也可以通过对 OpenAI chat completions API 的扩展，在运行时进行设置。
MDC 中定义的限制（如最大图像尺寸、最大 RAM 分配）不可在运行时被覆盖。

例如，可以为某个请求指定与 MDC 中默认值不同的视频采样策略：

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": ...,
    "messages": ...,
    "media_io_kwargs": {
        "video": {
            "fps": 1.0,
            "max_frames": 16
        }
    }
  }'
```

## TODO

### 模态

- [x] 图像解码
- [x] 视频解码
- [ ] 音频解码

### 性能

- [x] 图像 SW 解码
- [ ] 视频 HW 解码（NVDEC）
- [ ] JPEG HW 解码（nvJPEG）
- [x] 稀疏视频采样（seek-forward）
- [ ] 内存 slab 预分配 / 注册

### 内存管理
- [ ] 内存向更低存储层级溢出
- [ ] 在客户端通知后及早释放内存

### 杂项
- [ ] 在性能、内存使用、输入分布上的可观测性（observability）
- [x] 每请求级别的解码选项
