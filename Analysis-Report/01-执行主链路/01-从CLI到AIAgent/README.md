# 第一部分：执行主链路

## 第一章：从 CLI/TUI/Gateway 到 AIAgent

### 1.1 主链路总览
可用如下抽象描述一次请求：

`入口事件 -> 规范化命令/消息 -> 组装上下文 -> 调用 LLM -> 触发工具 -> 工具结果回注 -> LLM 收敛输出`

这一链路由不同入口触发，但收敛于 `run_agent.py::AIAgent.run_conversation`。

### 1.2 AIAgent.run_conversation 的关键阶段
结合源码可拆为 8 个阶段：
1. **会话与运行态准备**：初始化 DB session、session context、task_id、重试计数器。
2. **输入净化**：用户消息的 surrogate 清洗，避免序列化异常。
3. **预算重置**：每轮构造 `IterationBudget(self.max_iterations)`，避免上一轮污染。
4. **消息装配**：合并 history + 当前用户消息 + system context。
5. **模型调用**：发起非流式或流式调用。
6. **工具回路**：若存在 tool call，则执行并将结果追加到 messages。
7. **终止判定**：无工具调用时输出最终文本。
8. **回合后处理**：记忆同步、预取、日志和状态收束。

### 1.3 工具分发真实链路
`run_agent.py -> model_tools.handle_function_call -> tools.registry.dispatch -> tool handler`

其中 `handle_function_call` 的关键价值：
- 参数类型强制转换（字符串到 schema 类型）。
- 插件 pre_tool_call/post_tool_call hook。
- execute_code 的 sandbox 工具集约束。
- transform_tool_result 二次变换。

### 1.4 主链路中的“工程护栏”
- 迭代预算：防止无限 tool loop。
- 异常 fail-open/fail-soft：局部工具失败不必立即崩整个会话。
- Hook 单次触发契约：避免重复 side effects。
- task_id 隔离：并发任务间 VM/状态互不污染。

### 1.5 为什么这条链路最先读
因为 Memory、Skills、Context compression、Delegation、Cron 本质都是“插入或改造这条主链路”的增强器。先理解主链路，其他模块才能快速定位作用点。
