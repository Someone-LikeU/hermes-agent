# 第二部分：核心机制

## 第八章：多 Agent 与 Delegation

### 8.1 核心能力
`tools/delegate_tool.py::delegate_task` 支持单任务与批任务委派，让复杂问题可分解并并发处理。

### 8.2 关键护栏
- parent_agent 必须存在。
- spawn pause 可全局暂停新分支。
- `max_spawn_depth` 防递归爆炸。
- `max_concurrent_children` 控并发。
- 模型传入 max_iterations 会被配置覆盖，预算以配置为准。

### 8.3 角色治理
- `leaf`：不可继续委派。
- `orchestrator`：可继续委派，但受深度与配置约束。

### 8.4 使用边界
`delegate_task` 是同步分治机制，不是持久后台执行；长任务应转 `cronjob` 或后台 terminal 任务。
