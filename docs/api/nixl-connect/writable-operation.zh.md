---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Writable Operation
---

一个允许远端 worker 向本地 worker 写入数据的操作。

要创建该操作，必须提供一组本地 [`Descriptor`](descriptor.md) 对象，引用用于接收来自远端 worker 数据的内存。
一旦创建，所提供描述符所引用的内存即可被持有相应元数据的远端 worker 写入。
访问这些内存所需的 NIXL 元数据（[RdmaMetadata](rdma-metadata.md)）可通过该操作的 `.metadata()` 方法获得。
获取后，元数据需要通过辅助通道（最有可能是 HTTP 或 TCP+NATS）发送给远端 worker。

对该对象的销毁将指示 NIXL 子系统取消该操作，
因此除非有取消意图，否则应等待该操作完成。
取消是异步处理的。


## 示例用法

```python
    async def recv_data(
      self,
      local_tensor: torch.Tensor
    ) -> None:
      descriptor = dynamo.nixl_connect.Descriptor(local_tensor)

      with await self.connector.create_writable(descriptor) as write_op:
        op_metadata = write_op.metadata()

        # Send the metadata to the remote worker via sideband communication.
        await self.request_remote_data(op_metadata)
        # Wait the remote worker to complete its write operation to local_tensor.
        # AKA receive data from remote worker.
        await write_op.wait_for_completion()
```


## 方法

### `metadata`

```python
def metadata(self) -> RdmaMetadata:
```

生成并返回远端 worker 向该操作写入数据所需的 NIXL 元数据（[RdmaMetadata](rdma-metadata.md)）。
获取后，元数据需要通过辅助通道（最有可能是 HTTP 或 TCP+NATS）发送给远端 worker。

### `wait_for_completion`

```python
async def wait_for_completion(self) -> None:
```

阻塞调用方，直到该操作收到来自远端 worker 的完成信号。


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
  - [WriteOperation](write-operation.md)
