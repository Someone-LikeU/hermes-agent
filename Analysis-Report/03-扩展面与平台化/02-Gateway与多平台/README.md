# 第三部分：扩展面与平台化

## 第二章：Gateway 与多平台接入

### 9.1 Gateway 的核心任务
Gateway 不是“转发器”而是“平台适配与会话编排层”：
- 处理平台事件模型差异。
- 映射统一命令系统。
- 管理会话身份与状态。
- 调用统一 AIAgent 内核。

### 9.2 命令体系一致性
`hermes_cli/commands.py` 的 `COMMAND_REGISTRY` 与 `resolve_command` 让 CLI/Gateway 共享命令语义，减少“同命令不同端行为不一致”问题。

### 9.3 TUI 与 Dashboard 的关系
项目明确倾向复用真实 TUI（PTY 嵌入），而非在 Web 端重写第二套聊天核心。这是降低行为分叉与维护成本的关键工程决策。

### 9.4 平台化扩展建议
- 新平台接入优先实现 adapter，不改核心 agent loop。
- 命令能力尽量从 registry 自动派生，避免平台私有分叉。
- 网关层增加可观测字段（platform, session_id, latency, tool_count）。
