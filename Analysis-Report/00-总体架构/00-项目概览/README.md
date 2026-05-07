# 第零部分：总体架构

## 第零章：项目概览

### 0.1 当前版本规模快照（工作区统计口径）
- 文件数：`3038`
- 文本总行数（常见源码/配置/文档后缀）：`1,122,433`
- Python 代码行数（同口径）：`704,522`

> 结论：这是一个“超大体量 Agent 平台工程”，不是单一对话脚本。阅读时必须采用“主链路优先 + 模块递进”的方式。

### 0.2 项目定位：Agent Runtime + Agent Platform
Hermes Agent 的真实定位更接近 **Agent Runtime + Platform**：
- Runtime：AIAgent 对话循环、工具编排、上下文管理、预算与中断控制。
- Platform：多入口（CLI/TUI/Gateway）、多插件面（模型/记忆/工具）、多运行模式（交互/批处理/定时任务）。

### 0.3 分层架构（建议心智模型）

#### 0.3.1 入口层（Entry Layer）
- `cli.py`：命令行主控与 slash command 分发。
- `ui-tui/` + `tui_gateway/`：Ink 前端 + Python JSON-RPC 后端。
- `gateway/run.py`：外部平台消息入口。

#### 0.3.2 编排层（Orchestration Layer）
- `run_agent.py`：AIAgent 核心循环、状态、预算、重试与容错。
- `model_tools.py`：函数调用分发与插件 hook。
- `toolsets.py`：平台可见工具集合。

#### 0.3.3 能力层（Capability Layer）
- `tools/`：基础工具能力。
- `skills/` + `optional-skills/`：Prompt 资产与任务模板。
- `plugins/`：模型、记忆、观测等扩展能力。

#### 0.3.4 运行支撑层（Infra Layer）
- `hermes_state.py`：会话与索引。
- `cron/`：调度执行与周期任务。
- `tools/environments/`：执行环境后端。

### 0.4 特色能力（相较常见 Agent 项目）
1. **强插件化边界**：核心鼓励通过插件扩展，减少核心硬编码。
2. **多入口共享同一内核**：CLI/TUI/Gateway 最终收敛到 AIAgent。
3. **可运行于生产约束下**：预算、迭代上限、中断、日志、会话恢复。
4. **多 Agent 任务分治**：支持子 Agent 与并行批任务。
5. **技能资产化与治理**：Skills + Curator 形成知识沉淀链路。

### 0.5 程序入口与最短理解路径
- 入口：`cli.py` / `gateway/run.py` / `ui-tui`。
- 核心执行：`run_agent.py::AIAgent.run_conversation`。
- 工具执行：`model_tools.py::handle_function_call`。

建议首次阅读顺序：
1. `run_agent.py`
2. `model_tools.py`
3. `toolsets.py`
4. `hermes_cli/commands.py`
5. `agent/memory_manager.py`
6. `tools/delegate_tool.py`
