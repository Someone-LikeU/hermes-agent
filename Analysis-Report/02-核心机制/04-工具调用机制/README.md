# 第二部分：核心机制

## 第四章：工具调用机制详解

### 4.1 调用主链路
`run_agent.py -> model_tools.handle_function_call -> registry.dispatch -> tool handler`

其中核心调度点是 `model_tools.handle_function_call`。

### 4.2 handle_function_call 的关键执行阶段
从源码行为可归纳为：
1. 参数类型矫正（`coerce_tool_args`）。
2. 拦截 Agent loop 内建工具（防止路径错误）。
3. pre_tool_call hook（插件可阻断）。
4. 调用 registry.dispatch 执行真实工具。
5. post_tool_call hook（带 `duration_ms` 可观测数据）。
6. transform_tool_result hook（二次改写返回值）。
7. 统一异常捕获并 JSON 错误返回。

### 4.3 工具注册与可见性
- 注册来源：`tools/*.py` 顶层 `registry.register(...)`。
- 可见性控制：`toolsets.py` 决定“哪些注册工具暴露给模型”。
- 关键认知：**注册成功 != 对模型可见**。

### 4.4 execute_code 的特殊治理
`execute_code` 分支会携带当前 enabled tools 上下文，目的是让 sandbox 能力边界保持一致，防止子上下文越权调用。

### 4.5 维护建议
- 工具返回必须 JSON 字符串，保持主链路一致性。
- 为高风险工具加 pre_hook 策略审计。
- 监控工具耗时分布，识别慢工具与不稳定依赖。
