# 多模态 Profile 系统

本目录维护按后端（backend）划分的多模态服务测试列表（每个后端一个文件 —— 当前是 `vllm.py`）。每个文件都声明一个
`<BACKEND>_MULTIMODAL_PROFILES` 列表，
`tests/serve/test_<backend>.py` 会通过 `make_multimodal_configs(...)`
将其展开为 pytest 的配置项。

## 为什么要这样做

在引入 profile 系统之前，每个多模态测试都是 `tests/serve/test_vllm.py` 中手写的一个
`VLLMConfig(...)` 块。新增一个模型、一种新拓扑或一种新的 payload 变体，都需要复制约 40 行代码，
并手动让 marks / timeouts / `script_args` /
`request_payloads` 保持一致。Profile 系统只声明
*形状*（模型、拓扑、payload），并由一个生成器构建出全部
配置项。

## 三层结构

```
MultimodalModelProfile           ← one HF model
  └─ topologies: dict[str, TopologyConfig]
       └─ TopologyConfig         ← one deployment shape (agg / e_pd / epd / p_d / ...)
            └─ tests: list[MmCase]
                 └─ MmCase       ← one concrete test case (payload + per-case overrides)
```

每一层都补充了上层无法表达的内容：

| 层 | 拥有 | 为什么需要这一层？ |
|-------|------|-----------------|
| `MultimodalModelProfile` | `name`、`short_name`、`gpu_marker`、`extra_vllm_args`、`gated`、`marks` | 模型粒度的、不随拓扑变化的事实（HF id、默认 GPU 类别、模型级的 pytest 标记，例如 `gated`）。 |
| `TopologyConfig` | `marks`、`timeout_s`、`delayed_start`、`profiled_vram_gib`、`requested_vllm_kv_cache_bytes`、`single_gpu` / `two_gpu`、`gpu_marker` 覆盖、`env` | 每种拓扑的启动形状（部署如何连线 —— 例如 `agg` 是单进程，`epd` 则是 encode + prefill + decode 三 GPU）。形状决定 GPU 数、显存边界与超时时间。 |
| `MmCase` | `payload`、`suffix`、`extra_script_args`、用例级的 `marks` / `timeout_s` / `env` 覆盖 | 同一拓扑下不同测试的差异 —— 启动形状相同，但 payload 不同（HTTP-URL vs base64 内联 vs 视频）或启动参数不同（例如 `--frontend-decoding`）。 |

拓扑键（`"agg"`、`"e_pd"` 等）是 `make_multimodal_configs`
在 `<BACKEND>_TOPOLOGY_SCRIPTS` 中查找启动脚本时用的键。
**多个拓扑键可以映射到同一个脚本** —— 见 `vllm.py` 中的
`agg`/`agg_video` 与 `epd`/`epd_video` 配对。之所以拆开，
是因为视频测试相比图像测试需要不同的显存边界 / 超时 /
`delayed_start`，尽管二者都运行
`agg_multimodal.sh`。

## 生成的内容

对每一个 `(profile, topology, case)` 三元组，`make_multimodal_configs`
会输出一个 `EngineConfig`，其键为：

```
mm_{topology}_{profile.short_name}[_{case.suffix}]
```

因此，`qwen3-vl-2b` 在 `agg` 拓扑下、配两个 MmCase
（`""` 与 `"b64"`）会产生：

```
mm_agg_qwen3-vl-2b
mm_agg_qwen3-vl-2b_b64
```

附加在每个 EngineConfig 上的 marks 是按顺序分层叠加的
（在 pytest 关心的冲突上以靠后的为准 —— pytest
只是收集所有 marks）：

1. GPU marker（`gpu_1` / `gpu_2` / `gpu_4`）—— 来自 `topology.gpu_marker`，
   缺省时回退到 `profile.gpu_marker`
2. `pytest.mark.timeout(...)` —— 设置了 `case.timeout_s` 时取该值，否则取
   `topology.timeout_s`
3. 用例级 marks（`case.marks`）若非空，则使用之；否则使用拓扑 marks
   （`topology.marks`）—— 用例完全覆盖拓扑
4. 拓扑上若设置了 `profiled_vram_gib(...)` 与 `requested_vllm_kv_cache_bytes(...)`
   则使用之
5. 若 `profile.gated` 为真，则附加 `skipif(DYN_HF_GATED_MODELS_ENABLED unset)`
6. profile 级别的 marks（`profile.marks`）

## 端到端示例

```python
# tests/serve/multimodal_profiles/vllm.py

VLLM_TOPOLOGY_SCRIPTS = {
    "agg": "agg_multimodal.sh",
    "agg_video": "agg_multimodal.sh",   # same script, different config row
    "e_pd": "disagg_multimodal_e_pd.sh",
    "epd": "disagg_multimodal_epd.sh",
    "epd_video": "disagg_multimodal_epd.sh",
    "p_d": "disagg_multimodal_p_d.sh",
}

VLLM_MULTIMODAL_PROFILES = [
    MultimodalModelProfile(
        name="Qwen/Qwen3-VL-2B-Instruct",
        short_name="qwen3-vl-2b",
        topologies={
            "agg": TopologyConfig(
                marks=[pytest.mark.post_merge],
                timeout_s=220,
                tests=[
                    MmCase(payload=make_image_payload(["green"])),
                    MmCase(suffix="b64", payload=make_image_payload_b64(["green"])),
                    MmCase(
                        suffix="frontend_decoding",
                        payload=make_image_payload(["green"]),
                        extra_script_args=["--frontend-decoding"],
                    ),
                ],
            ),
            "agg_video": TopologyConfig(
                marks=[pytest.mark.pre_merge],
                timeout_s=600,
                delayed_start=60,
                profiled_vram_gib=8.2,
                requested_vllm_kv_cache_bytes=1_719_075_000,
                tests=[MmCase(payload=make_video_payload(["red", "static", "still"]))],
            ),
            ...
        },
    ),
    ...
]
```

会展开为：

```
mm_agg_qwen3-vl-2b                           # post_merge, gpu_1, t=220, image
mm_agg_qwen3-vl-2b_b64                       # post_merge, gpu_1, t=220, b64-image
mm_agg_qwen3-vl-2b_frontend_decoding         # post_merge, gpu_1, t=220, image, --frontend-decoding
mm_agg_video_qwen3-vl-2b                     # pre_merge,  gpu_1, t=600, video, kv-bounded
...
```

每一项最终都会变成一个 `VLLMConfig(...)` 实例，带有正确的
`script_name`、`script_args`、`marks`、`request_payloads` 与 `env`，
并被串接进 `tests/serve/test_vllm.py` 中的 `vllm_configs`。

## 新增变体

| 想要新增...                                              | 应当放到哪里                                                                                                |
|----------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| 一个新模型                                               | 在 `VLLM_MULTIMODAL_PROFILES` 中新增一个 `MultimodalModelProfile(...)` 条目                                   |
| 一种新的拓扑形状（例如张量并行）                          | 在 `VLLM_TOPOLOGY_SCRIPTS` 新增一项 + 在 `examples/backends/vllm/launch/` 新增对应的启动脚本                  |
| 已有拓扑下的新测试用例（例如同一模型用不同 payload 测试） | 在该拓扑的 `tests` 列表中新增一个 `MmCase(...)`                                                              |
| 一个新的拓扑边界（例如视频需要更长超时）                  | 用一个不同的拓扑键新增一份 `TopologyConfig`（如 `agg_video`）—— 多个键可以映射到同一启动脚本                  |
| 仅对单个用例生效的启动参数                                | `MmCase(extra_script_args=[...], ...)`                                                                     |

## `tests/utils/multimodal.py` 中的分层辅助工具

`tests/utils/multimodal.py` 中包含数据类（`MmCase`、
`TopologyConfig`、`MultimodalModelProfile`）、生成器
（`make_multimodal_configs`），以及在 profile
之间共享的 payload 工厂：

- `make_image_payload(expected_colors, *, max_attempts=1)` —— 使用标准测试图像的
  HTTP-URL 颜色识别 payload
- `make_image_payload_b64(expected_colors)` —— 同样的 prompt，但将
  LFS 跟踪的 PNG 以 `data:image/png;base64,...` 形式内联进 URL。
  其物化（materialization）被推迟到第一次读取 `.body` 时，
  这样在 LFS 还没拉取该图像时 pytest 收集阶段不会失败
- `make_video_payload(expected_phrases)` —— 本地测试视频
- `make_audio_payload(expected_phrases)` —— 远端测试 WAV

后端专用文件（`vllm.py`，将来的 `trtllm.py` 等）只是把
这些工厂与 `MmCase` 组合起来，并不会自行定义工厂。
