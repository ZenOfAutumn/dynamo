---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Operation Status
---

表示一个操作当前的状态或阶段。


## 取值

### `CANCELLED`

操作已被用户或系统取消。

### `COMPLETE`

操作已成功完成。

### `ERRORED`

操作遇到错误，无法完成。

### `IN_PROGRESS`

操作已初始化且正在进行中（既未完成、出错，也未取消）。

### `INITIALIZED`

操作已初始化，准备被处理。

### `UNINITIALIZED`

操作尚未初始化，处于无效状态。


## 相关类

  - [Connector](connector.md)
  - [Descriptor](descriptor.md)
  - [Device](device.md)
  - [RdmaMetadata](rdma-metadata.md)
  - [ReadOperation](read-operation.md)
  - [ReadableOperation](readable-operation.md)
  - [WritableOperation](writable-operation.md)
  - [WriteOperation](write-operation.md)
