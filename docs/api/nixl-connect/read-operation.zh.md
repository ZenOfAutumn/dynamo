---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Read Operation
---

一个将数据从远端 worker 传输到本地 worker 的操作。

要创建该操作，必须提供来自远端 worker 上 [`ReadableOperation`](readable-operation.md) 的 NIXL 元数据（[RdmaMetadata](rdma-metadata.md)），以及一组与之匹配的、引用了用于接收远端 worker 数据的内存的本地 [`Descriptor`](descriptor.md) 对象。
NIXL 元数据必须通过辅助通道（最有可能是 HTTP 或 TCP+NATS）从远端传输到本地 worker。

一旦创建，数据传输将立即开始。
对该对象的销毁将指示 NIXL 子系统取消该操作，
因此除非有取消意图，否则应等待该操作完成。


## 示例用法

```python
    async def read_from_remote(
      self,
      remote_metadata: dynamo.nixl_connect.RdmaMetadata,
      local_tensor: torch.Tensor
    ) -> None:
      descriptor = dynamo.nixl_connect.Descriptor(local_tensor)

      with await self.connector.begin_read(remote_metadata, descriptor) as read_op:
        # Wait for the operation to complete writing data from the remote worker to local_tensor.
        await read_op.wait_for_completion()
```


## 方法

### `cancel`

```python
def cancel(self) -> None:
```

指示 NIXL 子系统取消该操作。
已完成的操作无法被取消。

### `wait_for_completion`

```python
async def wait_for_completion(self) -> None:
```

阻塞调用方，直到来自远端 worker 的内存已传输到所提供的缓冲区。


## 属性

### `status`

```python
@property
def status(self) -> OperationStatus:
```

返回 [`OperationStatus`](operation-status.md)，提供该操作当前的状态。


## 相关类

  - [Connector](connector.md)
  - [Descriptor](descriptor.md)
  - [Device](device.md)
  - [OperationStatus](operation-status.md)
  - [RdmaMetadata](rdma-metadata.md)
  - [ReadableOperation](readable-operation.md)
  - [WritableOperation](writable-operation.md)
  - [WriteOperation](write-operation.md)
