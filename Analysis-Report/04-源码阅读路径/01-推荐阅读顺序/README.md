# 第四部分：源码阅读路径

## 第一章：推荐阅读顺序（强化版）

### 10.1 第一阶段：建立执行主链（1-2 天）
1. `run_agent.py`
   - 目标：看懂 `run_conversation` 的阶段与终止条件。
2. `model_tools.py`
   - 目标：看懂工具分发与 hook 流程。
3. `toolsets.py`
   - 目标：看懂工具“注册”与“暴露”差异。

输出物：画出一张 `user -> model -> tools -> model -> response` 时序图。

### 10.2 第二阶段：建立核心机制模型（2-4 天）
1. `agent/memory_provider.py` + `agent/memory_manager.py`
2. `tools/delegate_tool.py`
3. `agent/context_engine.py` 与压缩相关模块

输出物：整理每个模块的“输入/输出/失败模式/恢复策略”。

### 10.3 第三阶段：建立扩展与平台模型（2-3 天）
1. `hermes_cli/commands.py`
2. `hermes_cli/plugins.py`
3. `gateway/run.py` + `gateway/platforms/*`
4. `ui-tui/` + `tui_gateway/`

输出物：整理“能通过插件做什么、必须改 core 才能做什么”。

### 10.4 第四阶段：测试反向阅读（持续）
- 从 `tests/` 里找与你改动相关的行为约束。
- 对 flaky/慢测试标注依赖条件，构建最小回归集。

### 10.5 fork 维护者每次 rebase 后必做清单
- 核查 `run_agent.py` 主循环是否新增护栏逻辑。
- 核查 `model_tools.py` hook 契约是否变更。
- 核查 `toolsets.py` 默认暴露面是否变化。
- 核查 `commands.py` 命令注册是否新增冲突别名。
