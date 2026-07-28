---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Descriptor
---

内存描述符，用于确保内存已注册到基于 NIXL 的 I/O 子系统中。
内存必须先注册到 NIXL 子系统，才能与之交互。

Descriptor 对象仅作管理用途，不会复制、移动或以其他方式修改已注册的内存。

创建描述符有四种方式：

 1. 从一个 `torch.Tensor` 对象创建。设备信息将从所提供的对象中推导。

 2. 从一个 `tuple` 创建，其中包含一个 NumPy 或 CuPy `ndarray` 以及描述内存所在位置的信息（Host/CPU 还是 GPU）。

 3. 从一个 Python `bytes` 对象创建。假定内存位于 CPU 可寻址的主机内存中。

 4. 从一个由内存地址、字节大小和设备信息组成的 `tuple` 创建。
    可以提供对一个 Python 对象的可选引用，以避免被垃圾回收。


## 方法

### `register_memory`

```python
def register_memory(self, connector: Connector) -> None:
```

指示该描述符将其内存缓冲区注册到基于 NIXL 的 I/O 子系统中。

在同一个描述符上多次调用此方法不会产生额外效果。

当描述符被分配给一个 NIXL 操作时，如果尚未显式注册，将自动注册。


## 属性

### `device`

```python
@property
def device(self) -> Device:
```

获取该描述符所代表的缓冲区所在的 [`Device`](device.md) 引用。

### `size`

```python
@property
def size(self) -> int:
```

获取该描述符所代表的内存分配的大小。

## 相关类

  - [Connector](connector.md)
  - [Device](device.md)
  - [OperationStatus](operation-status.md)
  - [RdmaMetadata](rdma-metadata.md)
  - [ReadOperation](read-operation.md)
  - [ReadableOperation](readable-operation.md)
  - [WritableOperation](writable-operation.md)
  - [WriteOperation](write-operation.md)
