---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Device
---

`Device` 类描述给定分配所在的设备。
通常是主机内存（`"cpu"`）或 GPU 内存（`"cuda"`）。

当系统包含多个 GPU 设备时，可以通过包含其序数索引来标识具体的 GPU 设备。
例如，要引用系统中的第二个 GPU，可以使用 `"cuda:1"`。

默认情况下，当传入 `"cuda"` 时，假定其代表 `"cuda:0"`，即系统枚举的第一个 GPU。


## 属性

### `id`

```python
@property
def id(self) -> int:
```

获取该设备的标识或序号。

当设备为 [`HOST`](device-kind.md#host) 时，该值始终为 `0`。

当设备为 [`GPU`](device-kind.md#cuda) 时，该值标识具体的 GPU。

### `kind`

```python
@property
def kind(self) -> DeviceKind:
```

获取该实例所引用设备的 [`DeviceKind`](device-kind.md)。


## 相关类

  - [Connector](connector.md)
  - [Descriptor](descriptor.md)
  - [OperationStatus](operation-status.md)
  - [ReadOperation](read-operation.md)
  - [ReadableOperation](readable-operation.md)
  - [RdmaMetadata](rdma-metadata.md)
  - [WritableOperation](writable-operation.md)
  - [WriteOperation](write-operation.md)
