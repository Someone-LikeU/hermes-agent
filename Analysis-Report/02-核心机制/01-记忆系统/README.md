# 第二部分：核心机制

## 第一章：记忆系统（Memory）

### 2.1 设计目标
Memory 机制在 Hermes 里承担三重任务：
1. 跨回合持久化事实/偏好。
2. 在新回合提供相关 recall context。
3. 通过工具接口允许模型读写可控记忆。

### 2.2 抽象契约：MemoryProvider
`agent/memory_provider.py` 提供统一接口，核心包含：
- `is_available()`：配置和依赖可用性判断。
- `initialize()`：会话级初始化。
- `prefetch()` / `queue_prefetch()`：召回与异步预取。
- `sync_turn()`：回合同步写入。
- `get_tool_schemas()` / `handle_tool_call()`：记忆工具暴露与执行。
- `shutdown()`：收尾。

并提供 `on_turn_start`、`on_session_end`、`on_session_switch` 等可选 hook。

### 2.3 编排器：MemoryManager
`agent/memory_manager.py` 采用“内置 + 至多一个外部 provider”的策略：
- builtin provider 始终允许。
- 外部 provider 只允许一个，防止 schema 冲突与行为竞态。
- 提供 `build_system_prompt()`、`prefetch_all()`、`sync_all()` 聚合调用。

这是一种典型的“能力统一入口 + 后端可替换”架构。

### 2.4 与主循环耦合点
Memory 与 AIAgent 的连接点通常是：
- 回合前：`prefetch` 注入 context。
- 回合后：`sync_turn` 写入本轮关键信息。
- 特殊模式：如 cron 会默认 `skip_memory=True`，避免自动任务污染用户长期记忆。

### 2.5 你在 fork 中最容易踩坑的点
1. **多 provider 并行启用**：会导致工具名冲突或写入冲突。
2. **慢写入阻塞主线程**：`sync_turn` 应尽量异步。
3. **会话切换遗漏清理**：`on_session_switch` 未实现会导致跨会话串写。

### 2.6 建议的源码阅读顺序
1. `agent/memory_provider.py`（接口契约）
2. `agent/memory_manager.py`（编排策略）
3. `plugins/memory/*`（具体实现）
4. `run_agent.py`（调用时机）
