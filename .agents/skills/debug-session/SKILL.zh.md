---
name: debug-session
description: Start a debugging session with worklog file
user-invocable: true
disable-model-invocation: true
---

# 启动调试会话

为 Dynamo 生态中的某个问题创建一个结构化的调试会话。

## 步骤 1：获取缺陷报告

询问用户希望以何种方式提供缺陷信息：

**选项 A：Linear 工单**
- 用户提供工单 ID（例如 "DYN-123"）
- 通过 Linear MCP 工具拉取
- 提取：标题、描述、复现步骤

**选项 B：GitHub issue**
- 用户提供 issue URL
- 通过 `gh issue view <url>` 拉取
- 提取：标题、描述、复现步骤

**选项 C：直接粘贴**
- 让用户直接粘贴缺陷报告
- 从中解析出关键信息

## 步骤 2：探查环境

收集环境信息：

!`nvidia-smi --query-gpu=name,count --format=csv,noheader 2>/dev/null || echo "No GPU detected"`

!`uname -a`

!`which python && python --version`

通过这些命令可以了解：
- GPU 型号与数量（L40s、H100 等）
- 操作系统/平台
- Python 环境

**注意**：用户的 `~/.claude/CLAUDE.md` 中可能包含其开发环境的更多细节（路径、别名、偏好设置等），可在此查找额外上下文。

## 步骤 3：创建工作日志

创建一份工作日志文件以追踪整个排查过程：

- 文件名：当前目录下的 `<issue-slug>.md`
- 模板：

```markdown
# Debug: [Issue Title]

**Date**: [today's date]
**Source**: [Linear ticket / GitHub issue / user report]
**Status**: investigating
**Environment**: [GPU type/count from nvidia-smi]

## Problem
[Description of the issue]

## Reproduction Steps
1. [Step to reproduce]
2. ...

## Expected vs Actual
- **Expected**:
- **Actual**:

## Investigation Log

### [timestamp]
[Notes on what you tried/found]

## Root Cause
[Fill in when found]

## Fix
[Fill in when implemented]
```

## 步骤 4：搭建测试环境

### 构建命令

修改代码后重新构建 Dynamo：
```bash
cd lib/bindings/python && maturin develop --uv && cd ../../.. && uv pip install -e .
```

如果需要修改某个框架（sglang、vllm、trtllm），请查看用户的 `~/.claude/CLAUDE.md` 以获取该框架专属的重新构建说明。

### 运行示例

示例位于：`/home/ubuntu/dynamo/examples/backends/`

可用后端：
- `sglang/launch/` - SGLang 后端示例
- `vllm/launch/` - vLLM 后端示例
- `trtllm/launch/` - TensorRT-LLM 后端示例

请根据缺陷报告判断相关后端：
- 如果不明确，**主动询问用户**应运行哪个后端/示例
- 在后台运行示例
- 等待模型就绪

### 验证模型已启动

```bash
curl localhost:8000/v1/models
```

### 通过请求进行测试

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "<model-name-from-above>",
    "messages": [{"role": "user", "content": "Hello"}],
    "max_tokens": 50
  }'
```

## 步骤 5：开始排查

### Dynamo 基础设施调试

**KV 缓存（KV cache）与路由问题：**
- 查看 `lib/llm/src/block_manager/kv_consolidator/tracker.rs` 中的 KV 事件日志
- 关注 block manager 的状态与合并行为
- 检查 KV 感知路由器（router）中的路由决策

**ZMQ / 网络问题：**
- 检查 ZMQ socket 配置以及端点绑定
- 关注连接超时或消息丢失
- 验证 nats/etcd 的连通性以确保服务发现正常

**多节点 / 解耦（disaggregated）问题：**
- 检查 prefill/decode worker 的分配
- 验证 DGD（disaggregated）状态上报
- 在每个节点上通过 `nvidia-smi` 检查节点间通信
- 检查 NCCL 与 GPU Direct RDMA 状态

**进程检查：**
- `ps aux | grep dynamo` - 查看运行中的进程
- `nvidia-smi` - GPU 利用率与显存使用
- `ss -tlnp | grep 8000` - 检查端口绑定
- `journalctl -u dynamo` - 如适用，查看 systemd 日志

### 通用调试流程

1. **先复现** - 在尝试修复前确认能稳定触发该缺陷
2. **边查边记** - 把发现实时记录到工作日志
3. **最小变更** - 只修复缺陷，不顺手重构周边代码
4. **验证修复** - 确认复现用例如今已通过

性能关键代码 - 避免不必要的抽象与注释。
