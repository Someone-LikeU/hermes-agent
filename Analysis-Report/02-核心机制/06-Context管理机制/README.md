# 第二部分：核心机制

## 第六章：Context 管理机制详解

### 6.1 Context 的组成面
Hermes 的上下文由多来源拼接：
- system 约束
- user 当前输入
- conversation history
- tool results
- memory prefetch
- skills 注入内容

### 6.2 run_conversation 的 Context 管理动作
从 `run_agent.py::run_conversation` 可以看到典型治理动作：
1. 用户输入净化（surrogate 清洗）。
2. history 拷贝，避免直接污染调用方消息。
3. task_id 设置，支撑并发任务隔离。
4. todo 状态从历史回填（特定模式下）。
5. 每轮预算对象重建，避免跨轮状态泄漏。

### 6.3 Context 管理的核心目标
- **正确性**：角色序列合法、工具结果可追溯。
- **隔离性**：不同会话/任务不串线。
- **可控性**：避免无限增长导致成本与延迟失控。

### 6.4 常见问题
- 工具输出过长导致历史爆炸。
- 多入口模式下会话信息不一致。
- 记忆/技能注入比例过高，稀释用户真实目标。
