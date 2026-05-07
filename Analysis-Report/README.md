# Hermes Agent 源码深度解读报告（增强版）

> 面向 fork 维护者：本报告在“结构化总览”的基础上，补充了更细的机制级解读、关键源码锚点、运行时数据流与推荐阅读顺序。

## 报告目录

### 第零部分：总体架构
- 第零章：项目概览：`00-总体架构/00-项目概览/README.md`

### 第一部分：执行主链路
- 第一章：从 CLI/TUI/Gateway 到 AIAgent：`01-执行主链路/01-从CLI到AIAgent/README.md`

### 第二部分：核心机制
- 第一章：记忆系统（Memory）：`02-核心机制/01-记忆系统/README.md`
- 第二章：Skills 体系：`02-核心机制/02-Skills体系/README.md`
- 第三章：MCP 机制：`02-核心机制/03-MCP机制/README.md`
- 第四章：工具调用机制：`02-核心机制/04-工具调用机制/README.md`
- 第五章：Sandbox 与执行环境：`02-核心机制/05-Sandbox与执行环境/README.md`
- 第六章：Context 管理机制：`02-核心机制/06-Context管理机制/README.md`
- 第七章：Prompt 编排机制：`02-核心机制/07-Prompt编排机制/README.md`
- 第八章：多 Agent 与 Delegation：`02-核心机制/08-多Agent与Delegation/README.md`

### 第三部分：扩展面与平台化
- 第一章：Plugin 架构：`03-扩展面与平台化/01-Plugin架构/README.md`
- 第二章：Gateway 与多平台接入：`03-扩展面与平台化/02-Gateway与多平台/README.md`

### 第四部分：源码阅读路径
- 第一章：推荐阅读顺序：`04-源码阅读路径/01-推荐阅读顺序/README.md`

### 第九十九部分：总结
- 第九十九章：总结与 fork 演进策略：`99-总结/99-总结/README.md`


### 附录：QA
- 记忆系统 QA：`QA/记忆系统QA.md`
- 上下文 Context 系统 QA：`QA/上下文context系统QA.md`
