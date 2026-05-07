# 第二部分：核心机制

## 第七章：Prompt 编排机制详解

### 7.1 Prompt 编排不是单一模板
Hermes 的 Prompt 是“消息序列编排系统”，而非单文件 prompt：
- system block
- user request
- tool call trace
- memory recall
- skill instruction

### 7.2 编排与工具回路的耦合
每次工具调用都会把 tool result 回注到 messages，再触发下一轮模型推理。因此 Prompt 在运行中是“动态生长”的。

### 7.3 编排质量的四个决定因素
1. system 约束清晰且稳定。
2. tool 返回结构化、低噪声。
3. history 裁剪/压缩策略合理。
4. memory/skills 注入有配额与优先级。

### 7.4 面向 fork 的优化建议
- 将高噪声工具输出摘要化。
- 按任务类型区分 system prompt 策略。
- 对关键任务记录 prompt 片段与结果质量，形成可回归数据集。
