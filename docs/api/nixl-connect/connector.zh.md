---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Connector
---

用于在分布式环境中管理 worker 间连接的核心类。
使用该类来创建可读/可写操作，或在远端 worker 上读写数据。

该类基于 NIXL 库为 Dynamo 图中由不同 worker 承载的模型之间的数据传输提供"Pythonic"接口；可用时会启用 GPU Direct RDMA 加速。
该 connector 提供两种 worker 之间数据搬移方式：

  - 准备本地内存以接受远端 worker 的写入。

  - 准备本地内存以供远端 worker 读取。

两种方式中，本地内存均通过 [`Descriptor`](#descriptor) 类注册到基于 NIXL 的 I/O 子系统并提供给 connector。
当 RDMA 可用时，connector 会配置 RDMA 子系统以面向所请求的操作暴露内存，并返回一个操作控制对象；
否则 connector 会选择最优的可用 RDMA 替代方案。
该操作控制对象（[`ReadableOperation`](readable-operation.md) 或 [`WritableOperation`](writable-operation.md)）通过其 `.metadata()` 方法提供 NIXL 元数据（[RdmaMetadata](rdma-metadata.md)），同时支持查询当前操作状态以及在完成前取消该操作。

NIXL 元数据必须提供给预期完成该操作的远端 worker。
该元数据中包含必要的信息（标识符、密钥等），使远端 worker 能够操作所提供的内存。

> [!Warning]
> NIXL 元数据包含 worker 的地址以及访问特定已注册内存描述符的安全密钥。
> 这些数据可在 worker 之间提供直接内存访问能力，应作为敏感信息谨慎处理。


## 用法示例

```python
    @async_on_start
    async def async_init(self):
      self.connector = dynamo.nixl_connect.Connector()
```

> [!Tip]
> 更多示例参见 [`ReadOperation`](read-operation.md#example-usage)、[`ReadableOperation`](readable-operation.md#example-usage)、
> [`WritableOperation`](writable-operation.md#example-usage) 和 [`WriteOperation`](write-operation.md#example-usage)。


## 方法

### `begin_read`

```python
async def begin_read(
    self,
    remote_metadata: RdmaMetadata,
    local_descriptors: Descriptor | list[Descriptor],
) -> ReadOperation:
```

创建一个 [`ReadOperation`](read-operation.md)，用于从远端 worker 传输数据。

创建该操作时，必须提供从远端 worker 的 [`ReadableOperation`](readable-operation.md) 序列化得到的请求，以及一组与之匹配的本地内存描述符（指向接收远端数据的内存）。
该序列化请求必须通过另一信道（很可能是 HTTP 或 TCP+NATS）从远端传到本地 worker。

创建后，数据传输会立即开始。

释放该对象会指示 NIXL 子系统取消操作；因此除非确实希望取消，否则应等待该操作完成。

使用 [`.wait_for_completion()`](read-operation.md#wait_for_completion) 阻塞调用方直到操作完成或出错。

### `begin_write`

```python
async def begin_write(
    self,
    local_descriptors: Descriptor | list[Descriptor],
    remote_metadata: RdmaMetadata,
) -> WriteOperation:
```

创建一个 [`WriteOperation`](write-operation.md)，用于向远端 worker 传输数据。

创建该操作时，必须提供从远端 worker 的 [`WritableOperation`](writable-operation.md) 序列化得到的请求，以及一组与之匹配的本地内存描述符（指向待传输到远端 worker 的内存）。
该序列化请求必须通过另一信道（很可能是 HTTP 或 TCP+NATS）从远端传到本地 worker。

创建后，数据传输会立即开始。

释放该对象会指示 NIXL 子系统取消操作；因此除非确实希望取消，否则应等待该操作完成。

使用 [`.wait_for_completion()`](write-operation.md#wait_for_completion) 阻塞调用方直到操作完成或出错。

### `create_readable`

```python
async def create_readable(
    self,
    local_descriptors: Descriptor | list[Descriptor],
) -> ReadableOperation:
```

创建一个 [`ReadableOperation`](readable-operation.md)，用于将数据传输到远端 worker。

创建该操作时，必须提供一组本地内存描述符，指向打算传输到远端 worker 的内存。
创建完成后，描述符所引用的内存对持有所需元数据的远端 worker 立即可读。
访问这些内存所需的元数据可通过该操作的 `.metadata()` 方法获得。
获取后，需要通过另一信道（很可能是 HTTP 或 TCP+NATS）将元数据提供给远端 worker。

释放该对象会指示 NIXL 子系统取消操作；因此除非确实希望取消，否则应等待该操作完成。

使用 [`.wait_for_completion()`](readable-operation.md#wait_for_completion) 阻塞调用方直到操作完成或出错。

### `create_writable`

```python
async def create_writable(
    self,
    local_descriptors: Descriptor | list[Descriptor],
) -> WritableOperation:
```

创建一个 [`WritableOperation`](writable-operation.md)，用于从远端 worker 接收数据。

创建该操作时，必须提供一组本地内存描述符，指向用于接收远端 worker 数据的内存。
创建完成后，描述符所引用的内存对持有所需元数据的远端 worker 立即可写。
访问这些内存所需的元数据可通过该操作的 `.metadata()` 方法获得。
获取后，需要通过另一信道（很可能是 HTTP 或 TCP+NATS）将元数据提供给远端 worker。

释放该对象会指示 NIXL 子系统取消操作；因此除非确实希望取消，否则应等待该操作完成。

使用 [`.wait_for_completion()`](writable-operation.md#wait_for_completion) 阻塞调用方直到操作完成或出错。


## 属性

### `hostname`

```python
@property
def hostname(self) -> str:
```

获取当前 worker 主机的名称。

### `is_cuda_available`

```python
@cached_property
def is_cuda_available(self) -> bool:
```

当 CUDA 可用于所选数组模块（很可能是 CuPy）时返回 `True`；否则返回 `False`。

### `name`

```python
@property
def name(self) -> str | None:
```

获取该 connector 使用的 Dynamo 组件名称。


## 相关类

  - [Descriptor](descriptor.md)
  - [Device](device.md)
  - [OperationStatus](operation-status.md)
  - [RdmaMetadata](rdma-metadata.md)
  - [ReadOperation](read-operation.md)
  - [ReadableOperation](readable-operation.md)
  - [WritableOperation](writable-operation.md)
  - [WriteOperation](write-operation.md)
