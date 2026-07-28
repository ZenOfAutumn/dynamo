---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Write Operation
---

一个将数据从本地 worker 传输到远端 worker 的操作。

要创建该操作，必须提供来自远端 worker 上 [`WritableOperation`](writable-operation.md) 的 NIXL 元数据（[RdmaMetadata](rdma-metadata.md)），以及一组与之匹配的、引用了将传输到远端 worker 的内存的本地 [`Descriptor`](descriptor.md) 对象。
NIXL 元数据必须通过辅助通道（最有可能是 HTTP 或 TCP+NATS）从远端传输到本地 worker。

一旦创建，数据传输将立即开始。
对该对象的销毁将指示 NIXL 子系统取消该操作，
因此除非有取消意图，否则应等待该操作完成。
取消是异步处理的。


## 示例用法

```python
    async def write_to_remote(
      self,
      remote_metadata: dynamo.nixl_connect.RdmaMetadata,
      local_tensor: torch.Tensor
    ) -> None:
      descriptor = dynamo.nixl_connect.Descriptor(local_tensor)

      with await self.connector.begin_write(descriptor, remote_metadata) as write_op:
        # Wait for the operation to complete writing local_tensor to the remote worker.
        await write_op.wait_for_completion()
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

阻塞调用方，直到所有提供的缓冲区都已传输到远端 worker。


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
  - [ReadOperation](read-operation.md)
  - [ReadableOperation](readable-operation.md)
  - [WritableOperation](writable-operation.md)
