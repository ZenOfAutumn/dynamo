# ZMQ KV 事件线协议解析

本模块负责解码由引擎侧 ZMQ 发布器发出的 msgpack KV 事件。
它同时支持带标签（tagged）的 map/object 事件，以及 Python `msgspec`
带标签的 `array_like=True` 事件。

## Map 事件

Map/object 事件按字段名进行解析。字段顺序无关紧要，可选字段可以独立省略。

Rust 测试 fixture 必须使用 `rmp_serde::to_vec_named` 或者
`Serializer::with_struct_map()` 才能产生这种结构。普通的
`rmp_serde::to_vec` 会按位置编码 struct。

## 位置（Positional）事件

Array-like 事件是按位置组织的元组（tuple）。解析器把每个事件看作一段固定的位置前缀，
后面跟一个可选的元数据尾部。

`BlockStored` 固定前缀：

```text
0 tag
1 block_hashes
2 parent_block_hash
3 token_ids
4 block_size
5 old lora_id slot
6 medium
7 lora_name
8 extra_keys
```

`BlockStored` 可选尾部：

```text
block_mm_infos
group_idx
kv_cache_spec_kind
kv_cache_spec_sliding_window
```

`BlockRemoved` 固定前缀：

```text
0 tag
1 block_hashes
2 medium
```

`BlockRemoved` 可选尾部：

```text
group_idx
kv_cache_spec_kind
kv_cache_spec_sliding_window
```

对于位置事件，靠后的字段需要前面所有位置都有占位值。例如，一个想要携带
`group_idx` 的 `BlockStored` 事件必须同时包含位置 5 到 8，即使其值为 `None`。
只有当后续字段都不存在时，事件才能提前结束。

尾部尽可能按类型解析。由于 `group_idx` 与 `kv_cache_spec_sliding_window`
都是 `u32`，数值类型尾部字段是顺序敏感的：尾部的第一个数值是 `group_idx`，
第二个是 `kv_cache_spec_sliding_window`。

## 生产者说明

vLLM 的 `BlockStored` 元组在元数据尾部之前，包含与 vLLM 兼容的固定前缀，
因此 `group_idx` 与缓存元数据从尾部位置解析。

SGLang 目前发出的是更短的位置 `BlockStored` 形式，到 `lora_id` 为止，
不包含缓存分组（cache-group）元数据。由于该元组提前结束，因此可以正确解析。
如果 SGLang 之后增加位置元数据，必须在尾部前包含与 vLLM 兼容的占位字段，
或改用带命名字段的 map/object 事件。
