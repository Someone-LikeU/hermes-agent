# Hermes Agent 多 Agent 与 Delegation 涉及文件清单

本文档梳理 Hermes Agent 项目中与“多 Agent / Delegation”直接相关的文件。已按要求忽略 `Analysis-Report/`、`.git/`、`.github/`、`node_modules/`、`ui-tui/node_modules/` 等分析报告、版本控制、依赖与工程配置目录。

## 结论概览

项目内多 Agent 机制主要分为两套：

1. **即时 Delegation 子 Agent**
   - 入口工具是 `delegate_task`。
   - 父 Agent 在同一进程内通过 `ThreadPoolExecutor` 生成一个或多个子 `AIAgent`。
   - 子 Agent 有独立上下文、独立 `task_id`、独立迭代预算、受限工具集和进度回传。
   - TUI 通过 `/agents`、`delegation.status`、`subagent.interrupt` 等接口展示和控制子 Agent 树。

2. **Kanban 持久化多 Agent 工作队列**
   - 通过 SQLite 看板、dispatcher、worker 进程、任务声明和 heartbeat 实现跨 profile / 跨进程协作。
   - Gateway 内嵌 dispatcher，周期性领取 ready 任务并启动 worker。
   - worker 通过 `kanban_*` 工具回写结果、阻塞原因、子任务和心跳。
   - Dashboard、CLI、Gateway slash command 共同操作同一套 Kanban DB。

## 即时 Delegation 核心运行链路

| 文件 | 作用 |
| --- | --- |
| `tools/delegate_tool.py` | Delegation 的核心实现。定义 `delegate_task`、子 Agent 构建、批量并行执行、最大并发、最大嵌套深度、角色 `leaf/orchestrator`、子 Agent 注册表、暂停/恢复、单个子 Agent 中断、超时诊断、进度事件和成本/文件变更汇总。 |
| `run_agent.py` | `AIAgent` 主循环集成 Delegation。拦截 `delegate_task` 这类 agent-loop tool，分发到 `tools.delegate_tool.delegate_task`；对子 Agent 传播 interrupt；限制同一轮里过量的 `delegate_task` 调用；把 Kanban worker 指引注入系统提示；维护父/子 Agent 的迭代预算、心跳、并行工具线程和回调传递。 |
| `model_tools.py` | 工具调度层。把 `delegate_task` 标记为 agent-loop tool，避免普通 registry dispatch 路径丢失父 Agent 状态；处理多线程工具调用中的事件循环和调用上下文。 |
| `toolsets.py` | 暴露 `delegation` toolset，并把 `delegate_task` 放入核心工具集合；同时定义 `kanban` toolset 中的 `kanban_*` 工具。 |
| `agent/tool_guardrails.py` | 工具安全策略中包含 `delegate_task`，影响工具选择和风险控制。 |
| `agent/display.py` | CLI/TUI 文本展示层对 `delegate_task` 做专门预览和可读消息渲染。 |
| `tools/terminal_tool.py` | 子 Agent 使用 terminal 工具时的线程本地 approval/sudo 回调依赖此文件；`delegate_tool.py` 会为子 Agent worker 线程安装非交互式 approval 回调，避免并行子 Agent 卡住父 UI。 |
| `tools/file_state.py` | 跨 Agent 文件状态协调。记录不同子 Agent 的读写时间戳，检测兄弟子 Agent 并发编辑同一文件导致的陈旧写入风险，并给 `delegate_tool.py` 提供子 Agent 完成后的“父 Agent 已读文件被子 Agent 修改”提醒。 |
| `tools/interrupt.py` | terminal 和 Agent 中断状态的底层事件；`run_agent.py` 与 terminal 工具依赖它把 interrupt 传播到并行工具线程和子 Agent。 |

## Delegation 配置、命令与入口

| 文件 | 作用 |
| --- | --- |
| `hermes_cli/config.py` | `DEFAULT_CONFIG` 中定义 `delegation` 配置，包括 `model`、`provider`、`base_url`、`api_key`、`max_concurrent_children`、`max_spawn_depth`、`max_iterations`、`child_timeout_seconds`、`reasoning_effort`、`subagent_auto_approve` 等；同时定义 `kanban` 配置。 |
| `cli.py` | 经典 CLI 的默认 delegation 配置、`/agents` 状态命令、`/kanban` slash command、并发 approval 锁，以及后台任务/并行 agent 工作相关逻辑。 |
| `hermes_cli/commands.py` | 中央 slash command registry。注册 `/agents`（别名 `/tasks`）和 `/kanban`，供 CLI、Gateway、TUI autocomplete/help 等统一复用。 |
| `hermes_cli/tools_config.py` | 工具配置 UI 中把 `delegation` 显示为 “Task Delegation”，映射到 `delegate_task`。 |
| `hermes_cli/tips.py` | CLI tips 中包含 `delegate_task` 并发、子 Agent model/provider、reasoning effort 等提示。 |
| `hermes_cli/web_server.py` | Dashboard 配置页面暴露 `delegation.reasoning_effort` 等配置字段，并把 `delegation` 作为配置分组之一。 |

## TUI 子 Agent 树与控制面

| 文件 | 作用 |
| --- | --- |
| `tui_gateway/server.py` | TUI JSON-RPC 后端。把 `subagent.*` 进度事件转换成 TUI 事件；提供 `delegation.status`、`delegation.pause`、`subagent.interrupt`、spawn tree 持久化/历史查询等 RPC；支持 `/agents` overlay。 |
| `ui-tui/src/gatewayTypes.ts` | 定义 `subagent.spawn_requested/start/thinking/tool/progress/complete` 事件 payload、`DelegationStatusResponse`、`SubagentInterruptResponse` 等类型。 |
| `ui-tui/src/app/createGatewayEventHandler.ts` | 接收 gateway 的 `subagent.*` 事件，刷新 delegation caps，更新当前 turn 的 subagent 状态，并把 spawn tree 归档到历史。 |
| `ui-tui/src/app/turnController.ts` | 当前 turn 状态控制器。维护 `subagents` 列表、按 `subagent_id` 合并事件、记录工具次数、状态、token、成本和文件 rollup。 |
| `ui-tui/src/app/turnStore.ts` | TUI 当前 turn store，包含 `subagents` 状态字段。 |
| `ui-tui/src/app/delegationStore.ts` | TUI delegation 状态 store，保存最大并发、最大深度、暂停状态和 overlay 折叠状态。 |
| `ui-tui/src/app/spawnHistoryStore.ts` | 保存已完成 turn 的子 Agent 树历史，用于 `/agents history` 和 overlay 回放。 |
| `ui-tui/src/lib/subagentTree.ts` | 从扁平事件列表重建子 Agent 树，计算子树汇总、宽度 sparkline、热度、成本、token、文件触达等聚合指标。 |
| `ui-tui/src/components/agentsOverlay.tsx` | `/agents` overlay 主界面。展示实时/历史子 Agent 树，支持 pause/resume、kill 单个 Agent、kill subtree、历史对比和详情查看。 |
| `ui-tui/src/components/thinking.tsx` | thinking 面板中展示 subagents 分区，把子 Agent 活动嵌入常规推理/工具活动视图。 |
| `ui-tui/src/components/appChrome.tsx` | 顶层 UI chrome 中展示 delegation HUD、并发/深度 caps、当前子 Agent 数等。 |
| `ui-tui/src/app/slash/commands/ops.ts` | TUI 侧 `/agents` slash command。调用 `delegation.status`、`delegation.pause`，打开 overlay 和历史视图。 |
| `ui-tui/src/domain/details.ts` | details section 中定义 `subagents` 分区的展开/折叠行为。 |
| `ui-tui/src/types.ts` | TUI 内部 `SubagentProgress`、section 名称等类型定义。 |

## Kanban 多 Agent 工作队列核心

| 文件 | 作用 |
| --- | --- |
| `hermes_cli/kanban_db.py` | Kanban 多 Agent 的核心持久化和调度层。定义 SQLite schema、board 路径、任务/依赖/评论/事件/run 模型、CAS claim、dispatcher `dispatch_once`、worker spawn、worker heartbeat、crash/reclaim/timeout 检测、worker 日志、worker 上下文构建、任务完成/阻塞/创建子任务等。 |
| `tools/kanban_tools.py` | dispatcher-spawned worker 和 orchestrator profile 的结构化工具面。注册 `kanban_show`、`kanban_complete`、`kanban_block`、`kanban_heartbeat`、`kanban_comment`、`kanban_create`、`kanban_link`；通过 `HERMES_KANBAN_TASK` 或 `kanban` toolset gating 限制普通聊天不可见。 |
| `hermes_cli/kanban.py` | `hermes kanban ...` CLI 和 `/kanban` slash command 的共享入口。覆盖 board 管理、任务创建/移动/阻塞/完成、dispatcher tick/daemon、worker heartbeat、日志查看、context 查看、diagnostics、specify 等操作。 |
| `hermes_cli/kanban_diagnostics.py` | Kanban worker/dispatcher 问题诊断规则。识别幻觉卡片、spawn crash-loop、任务卡住、worker 崩溃等，并生成可执行建议。 |
| `hermes_cli/kanban_specify.py` | 把 triage task 改写为可执行 worker spec，供 `hermes kanban specify` 和 dispatcher 前置整理任务使用。 |
| `gateway/run.py` | Gateway 内嵌 Kanban dispatcher 和 notifier。启动 `_kanban_dispatcher_watcher` 周期性调用 `kanban_db.dispatch_once` 生成 worker；启动 `_kanban_notifier_watcher` 把任务完成/阻塞/崩溃消息发回平台；处理 Gateway 侧 `/kanban`。 |
| `plugins/kanban/systemd/hermes-kanban-dispatcher.service` | 旧的/可选 systemd dispatcher 服务模板；当前代码更推荐 Gateway 内嵌 dispatcher。 |

## Kanban Dashboard 与 Web 集成

| 文件 | 作用 |
| --- | --- |
| `plugins/kanban/dashboard/plugin_api.py` | Kanban Dashboard 后端 API。包装 `kanban_db` 的任务、评论、事件、run、诊断和 WebSocket tail 能力。 |
| `plugins/kanban/dashboard/manifest.json` | Dashboard 插件声明，用于 dashboard 插件系统加载 Kanban 页面。 |
| `plugins/kanban/dashboard/dist/index.js` | Kanban Dashboard 前端构建产物，包含看板列、任务抽屉、dispatcher nudge、board switcher、worker log、run history 等 UI。 |
| `plugins/kanban/dashboard/dist/style.css` | Kanban Dashboard 前端样式。 |
| `web/src/pages/ConfigPage.tsx` | Dashboard 配置页面把 `delegation` 和 `kanban` 分组映射到图标/配置 UI。 |
| `web/src/i18n/en.ts` | Dashboard 英文文案中包含 `delegation`。 |
| `web/src/i18n/zh.ts` | Dashboard 中文文案中包含 `delegation`。 |
| `web/src/i18n/types.ts` | Dashboard i18n 类型中包含 `delegation` 字段。 |

## 多 Agent 技能与工作流资产

| 文件 | 作用 |
| --- | --- |
| `skills/software-development/subagent-driven-development/SKILL.md` | 以 `delegate_task` 为核心的子 Agent 驱动开发工作流。定义“每个任务一个新子 Agent + 两阶段 review”的实施规范。 |
| `skills/software-development/subagent-driven-development/references/context-budget-discipline.md` | 多子 Agent 编排中的上下文预算纪律，指导 orchestrator 避免把大量内容塞入自身上下文。 |
| `skills/software-development/subagent-driven-development/references/gates-taxonomy.md` | 子 Agent 工作流中的质量门和失败处理分类。 |
| `skills/devops/kanban-worker/SKILL.md` | Kanban worker 自动加载的深层技能说明，补充 dispatcher 注入的 worker 生命周期指引。 |
| `skills/devops/kanban-orchestrator/SKILL.md` | Kanban orchestrator profile 的任务拆分、路由和看板协作策略。 |
| `skills/autonomous-ai-agents/hermes-agent/SKILL.md` | Hermes 自身技能说明，包含 `delegate_task` 与 Kanban 多 Agent 工作队列的使用说明。 |
| `skills/autonomous-ai-agents/DESCRIPTION.md` | autonomous-ai-agents 分类说明，涵盖 spawning、delegating、multi-agent workflows。 |
| `skills/autonomous-ai-agents/claude-code/SKILL.md` | 外部 autonomous coding agent Claude Code 的集成技能，涉及其 subagent 能力和与 Hermes 的委托关系。 |
| `skills/autonomous-ai-agents/codex/SKILL.md` | 外部 Codex agent 工作流相关技能。 |
| `skills/autonomous-ai-agents/opencode/SKILL.md` | 外部 OpenCode agent 工作流相关技能。 |
| `skills/software-development/requesting-code-review/SKILL.md` | 使用独立 reviewer 子 Agent 做质量门和 auto-fix loop。 |
| `skills/software-development/test-driven-development/SKILL.md` | 说明如何在 `delegate_task` 子 Agent 中强制 TDD。 |
| `skills/software-development/systematic-debugging/SKILL.md` | 说明复杂调试中如何派发调查子 Agent。 |
| `skills/software-development/writing-plans/SKILL.md` | 计划阶段与 `subagent-driven-development` 的衔接说明。 |
| `skills/research/research-paper-writing/SKILL.md` | 研究写作中使用 `delegate_task` 并行撰写章节和验证引用。 |
| `optional-skills/security/oss-forensics/SKILL.md` | 多 Agent OSS 安全取证框架，显式要求用 `delegate_task` 批量派发调查员。 |
| `optional-skills/security/oss-forensics/references/*.md` | OSS 取证多 Agent 工作流的证据、模板、恢复和调查参考资料。 |
| `optional-skills/security/oss-forensics/templates/*.md` | 取证子 Agent 输出报告模板。 |
| `optional-skills/security/oss-forensics/scripts/evidence-store.py` | 取证工作流使用的证据存储辅助脚本。 |
| `optional-skills/creative/kanban-video-orchestrator/SKILL.md` | 基于 Kanban 的多 Agent 视频生产 orchestrator。 |
| `optional-skills/creative/kanban-video-orchestrator/scripts/bootstrap_pipeline.py` | 生成多 profile / 多 worker 视频管线启动脚本。 |
| `optional-skills/creative/kanban-video-orchestrator/scripts/monitor.py` | 监控 Kanban 视频生产管线。 |
| `optional-skills/creative/kanban-video-orchestrator/references/*.md` | 视频管线中的角色、工具矩阵、监控、Kanban setup 和示例资料。 |
| `optional-skills/creative/kanban-video-orchestrator/assets/*.tmpl` | 生成多 Agent 视频管线时使用的 setup、brief、soul 模板。 |

## 文档与参考资料

| 文件 | 作用 |
| --- | --- |
| `website/docs/user-guide/features/delegation.md` | 用户侧 Delegation 功能说明。 |
| `website/docs/guides/delegation-patterns.md` | Delegation 使用模式和编排建议。 |
| `website/docs/user-guide/features/kanban.md` | 用户侧 Kanban 多 Agent 工作队列说明。 |
| `website/docs/user-guide/features/kanban-tutorial.md` | Kanban 多 Agent 工作流教程。 |
| `website/docs/user-guide/configuration.md` | `delegation`、`kanban`、terminal sandbox 与并行子 Agent 配置说明。 |
| `website/docs/reference/toolsets-reference.md` | `delegation` 和 `kanban` toolset 参考。 |
| `website/docs/reference/tools-reference.md` | `delegate_task` 与 Kanban 工具参考。 |
| `website/docs/developer-guide/architecture.md` | 架构文档中标出 `delegate_tool.py` 等关键文件。 |
| `website/docs/developer-guide/agent-loop.md` | Agent loop 文档说明 `delegate_task` 和子 Agent 迭代预算。 |
| `website/docs/developer-guide/tools-runtime.md` | 工具 runtime 文档说明 agent-loop tools 中包含 `delegate_task`。 |
| `website/docs/developer-guide/provider-runtime.md` | provider runtime 文档说明子 Agent provider/model 继承和覆盖。 |
| `website/docs/user-guide/skills/bundled/software-development/software-development-subagent-driven-development.md` | `subagent-driven-development` 技能生成文档。 |
| `website/docs/user-guide/skills/bundled/devops/devops-kanban-worker.md` | `kanban-worker` 技能生成文档。 |
| `website/docs/user-guide/skills/bundled/devops/devops-kanban-orchestrator.md` | `kanban-orchestrator` 技能生成文档。 |
| `docs/hermes-kanban-v1-spec.pdf` | Kanban v1 设计规格文档。 |
| `docs/plans/2026-05-02-telegram-dm-user-managed-multisession-topics.md` | 提到 subagent-driven-development 的历史计划文档，属于弱相关背景。 |

## 测试覆盖文件

| 文件 | 覆盖点 |
| --- | --- |
| `tests/tools/test_delegate.py` | `delegate_task` 主测试。覆盖子 Agent 构建、并行、超时、中断、成本汇总、工具集裁剪、provider/model 覆盖、reasoning effort、nested orchestration、approval 回调等。 |
| `tests/tools/test_delegate_toolset_scope.py` | Delegation 子 Agent 工具集作用域和泄漏防护。 |
| `tests/tools/test_delegate_composite_toolsets.py` | 复合 toolset 与 Delegation 交互。 |
| `tests/tools/test_delegate_subagent_timeout_diagnostic.py` | 子 Agent 超时诊断日志。 |
| `tests/tools/test_file_state_registry.py` | 并发子 Agent 文件读写状态协调。 |
| `tests/agent/test_subagent_progress.py` | 子 Agent 进度事件回传。 |
| `tests/agent/test_subagent_stop_hook.py` | 子 Agent 停止 hook。 |
| `tests/run_agent/test_real_interrupt_subagent.py` | 真实 Agent 中断传播到子 Agent。 |
| `tests/cli/test_cli_interrupt_subagent.py` | CLI/TUI 侧中断子 Agent 行为。 |
| `ui-tui/src/__tests__/subagentTree.test.ts` | TUI 子 Agent 树重建和聚合逻辑。 |
| `ui-tui/src/__tests__/createGatewayEventHandler.test.ts` | TUI gateway event handler 对 `subagent.*` 事件的处理。 |
| `ui-tui/src/__tests__/details.test.ts` | `subagents` details section 展示模式。 |
| `tests/tools/test_kanban_tools.py` | Kanban worker/orchestrator 工具面。 |
| `tests/hermes_cli/test_kanban_db.py` | Kanban DB、任务状态、claim、run、worker 日志和上下文构建。 |
| `tests/hermes_cli/test_kanban_core_functionality.py` | Kanban 核心任务流。 |
| `tests/hermes_cli/test_kanban_cli.py` | `hermes kanban` CLI 行为。 |
| `tests/hermes_cli/test_kanban_boards.py` | 多 board 隔离和切换。 |
| `tests/hermes_cli/test_kanban_diagnostics.py` | Kanban 诊断规则。 |
| `tests/hermes_cli/test_kanban_specify.py` | Kanban specify 流程。 |
| `tests/hermes_cli/test_kanban_specify_db.py` | specify 与 DB 状态交互。 |
| `tests/hermes_cli/test_pin_kanban_board_env.py` | dispatcher/worker 使用 env 固定 board 的行为。 |
| `tests/plugins/test_kanban_dashboard_plugin.py` | Kanban dashboard plugin API。 |
| `tests/stress/_fake_worker.py` | Kanban/worker 压测辅助 fake worker。 |

## 弱相关或支撑性文件

| 文件 | 说明 |
| --- | --- |
| `batch_runner.py` | 批处理并行运行 Agent，但不是 `delegate_task` 子 Agent 树的核心实现。 |
| `mini_swe_runner.py` | SWE 风格任务运行器，可能与多 Agent 工作流并行使用，但不构成核心 Delegation 机制。 |
| `plugins/hermes-achievements/dashboard/plugin_api.py` | 统计 `delegate_task` 调用次数用于成就系统，不参与多 Agent 调度。 |
| `plugins/disk-cleanup/__init__.py` | 清理逻辑中考虑子 Agent 产生的文件/状态，属于外围维护。 |
| `plugins/memory/*` | 个别 memory provider 会识别 `agent_context == "subagent"` 或多 Agent 标识，但不是 Delegation 调度核心。 |
| `acp_adapter/*` | ACP 集成可承载 Agent 会话，但未直接实现 `delegate_task` 或 Kanban dispatcher。 |
