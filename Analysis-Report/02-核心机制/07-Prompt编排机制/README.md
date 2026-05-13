# Prompt 编排机制

本报告基于源码阅读，已忽略 `Analysis-Report/`、`.git/` 以及工程配置目录。Hermes Agent 的运行时 prompt 不是单一模板，而是“稳定 system prompt 快照 + 当前会话消息 + API 调用时临时注入 + 独立 tool schemas”的组合。

## 总体流程

1. `AIAgent.run_conversation()` 在新会话首轮调用 `_build_system_prompt()` 构建完整 system prompt。
2. 构建结果缓存到 `self._cached_system_prompt`，并写入 `SessionDB.sessions.system_prompt`。
3. 后续轮次优先复用 session DB 中的 system prompt 快照，避免因记忆或技能文件变化破坏 prompt cache。
4. 每次真正发 API 请求前，再把临时 system prompt、prefill messages、插件上下文、外部记忆预取等拼到 API 消息里。
5. 工具调用结果追加到 `messages`，成为后续模型推理的上下文；子目录提示也通过 tool result 注入，不改 system prompt。

## 固定部分

固定部分指一次会话内首轮构建后保持稳定的 system prompt 快照。主要来源：

- Agent 身份：优先 `~/.hermes/SOUL.md`，否则使用 `DEFAULT_AGENT_IDENTITY`。
- Hermes 自助提示：遇到 Hermes 自身配置/使用问题时加载 `hermes-agent` skill。
- 工具相关行为约束：按已启用工具注入 memory、session_search、skills、kanban 等指导。
- 工具使用强制规则：按 `agent.tool_use_enforcement` 和模型名注入通用工具执行纪律，以及 GPT/Codex/Gemini 专项指导。
- 内置记忆快照：`MEMORY.md`、`USER.md` 在 agent 初始化时冻结，mid-session 写入不会改变本轮 system prompt。
- 外部 memory provider 的 `system_prompt_block()`。
- Skills 索引：存在 skills 工具时生成 `<available_skills>` 列表，按平台、禁用列表、工具/工具集条件过滤。
- 项目上下文文件：按优先级只加载一种项目规则：`.hermes.md/HERMES.md`、`AGENTS.md`、`CLAUDE.md`、`.cursorrules/.cursor/rules/*.mdc`。
- 会话启动时间、session id、model、provider。
- 运行环境提示：例如 WSL 路径提示。
- 平台格式提示：CLI、Telegram、Slack、Discord、WhatsApp、API Server 等平台的输出格式规则。

这些内容由 `run_agent.py::_build_system_prompt()` 统一拼接，使用空行连接。

## 动态添加部分

动态部分不会直接改写稳定 system prompt，通常只在 API 调用时或当前消息链中出现：

- `ephemeral_system_prompt`：来自 `HERMES_EPHEMERAL_SYSTEM_PROMPT`、`agent.system_prompt`、gateway channel/context prompt、API Server 入参 system messages 等；每次 API 调用时附加到 cached system prompt 后，不写入 session DB。
- `prefill_messages`：JSON 数组形式的 few-shot/context priming，插入在 system prompt 之后、conversation history 之前，不持久化。
- 用户消息与历史消息：`conversation_history` 加当前 `user_message`，持续增长并可被压缩。
- 工具调用轨迹：assistant tool call 与 tool result 被追加到 `messages`。
- 外部 memory provider 预取：`prefetch_all()` 的结果包装为 `<memory-context>`，注入当前 user message。
- 插件 `pre_llm_call` 返回的上下文：注入当前 user message，不进入 system prompt。
- Slash skill 调用：`/skill-name` 会把完整 skill 内容构造成一条用户消息；`--skills` 预加载则拼入 ephemeral system prompt。
- `/reload-skills` 和模型切换提示：作为一次性 note prepend 到下一条 user message。
- 子目录 context hints：读取工具访问路径附近的 `AGENTS.md/CLAUDE.md/.cursorrules`，追加到 tool result。
- Context compression：压缩历史后会使 system prompt 失效并重建，用新的稳定快照继续后续轮次。

## 编排顺序

稳定 system prompt 的实际拼接顺序：

1. `SOUL.md` 或 `DEFAULT_AGENT_IDENTITY`
2. `HERMES_AGENT_HELP_GUIDANCE`
3. 工具相关 guidance：memory、session_search、skills、kanban
4. Nous subscription 能力提示
5. tool-use enforcement 及模型专项执行纪律
6. 调用方传入的 `system_message`
7. 内置记忆快照与用户画像快照
8. 外部 memory provider system block
9. skills 索引
10. 项目上下文文件
11. conversation started 时间、session/model/provider
12. provider 特例提示，例如 Alibaba model identity
13. 环境提示
14. 平台提示

API 请求前的最终消息形态：

```text
system/developer: cached_system_prompt + ephemeral_system_prompt
prefill:         可选 few-shot messages
history:         历史 user/assistant/tool messages
current user:    原始用户输入 + 可选 memory/plugin 临时上下文
tools:           独立 tool schemas，不拼入文本 prompt
```

GPT-5/Codex 类模型在 Chat Completions transport 中会把首条 `system` role 改成 `developer` role；Codex Responses transport 会把首条 system message 提取为 Responses API 的 `instructions`。

## 关键结论

- Hermes 把 system prompt 设计成“会话级稳定快照”，目的是最大化 prompt cache 命中。
- 运行时高频变化的内容被放到 user message、tool result、prefill 或 ephemeral system prompt，避免污染稳定前缀。
- Skills 有两种注入形态：系统提示里只放索引；实际 skill 全文通过 `skill_view` 或 slash command 动态加载。
- Memory 有两层：内置 `MEMORY.md/USER.md` 作为首轮冻结快照；外部 provider 的 recall/prefetch 作为当前轮临时上下文。
- 项目规则首轮只加载 CWD 的最高优先级规则文件；工具访问新目录时再通过子目录 hint 渐进注入。
