---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: RDMA Metadata
---

一种 Pydantic 类型，旨在为 [`ReadableOperation`](readable-operation.md) 或 [`WritableOperation`](writable-operation.md) 对象提供 JSON 序列化的 NIXL 元数据。
NIXL 元数据包含有关 worker 进程的详细信息，以及如何访问与对应 agent 注册的内存区域。
该数据是使用基于 NIXL 的 I/O 子系统执行数据传输所必需的。

> [!Warning]
> NIXL 元数据包含用于跨 agent 连接对应后端的信息，以及访问已注册的特定内存区域的标识密钥。
> 此数据在多个 worker 之间提供直接的内存访问，应被视为敏感数据并相应地进行处理。

使用对应类的 `.metadata()` 方法可为操作生成一个 `RdmaMetadata` 对象。

> [!Tip]
> 使用 `RdmaMetadata` 对象的类必须正确配对。
> [`ReadableOperation`](readable-operation.md) 与 [`ReadOperation`](read-operation.md) 配对，
> [`WritableOperation`](write-operation.md) 与 [`WriteOperation`](write-operation.md) 配对。
> 错误的配对将导致抛出错误。


## 相关类

  - [Connector](connector.md)
  - [Descriptor](descriptor.md)
  - [Device](device.md)
  - [OperationStatus](operation-status.md)
  - [ReadOperation](read-operation.md)
  - [ReadableOperation](readable-operation.md)
  - [WritableOperation](writable-operation.md)
  - [WriteOperation](write-operation.md)
