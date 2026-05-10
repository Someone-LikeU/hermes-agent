# Hermes Agent Skills 体系涉及文件清单

本文档基于仓库当前文件系统梳理，已忽略 `Analysis-Report/`、`.git/`、`.github/`、`node_modules/` 等分析产物与工程配置相关目录。

## 结论概览

Skills 体系不是单一模块，而是由以下几层组成：

1. 技能资产层：`skills/`、`optional-skills/`、插件自带 `SKILL.md`。
2. 发现与加载层：扫描 `SKILL.md`、解析 frontmatter、过滤平台/禁用项、构造系统提示与 `/skill-name` 调用消息。
3. Agent 工具层：`skills_list`、`skill_view`、`skill_manage`、安装同步、安全扫描、使用统计。
4. 入口层：CLI、TUI、Gateway、Dashboard、Cron、Kanban 等把技能加载到会话或任务里。
5. 生命周期层：Curator 负责维护 agent 创建的技能，包含归档、合并、快照与回滚。

## 核心运行链路文件

| 文件 | 作用 |
| --- | --- |
| `run_agent.py` | Agent 主循环集成 Skills 系统提示、技能工具可用性判断、复杂任务后的技能复盘/创建提示，以及后台 review fork 的 skill 写入来源。 |
| `agent/prompt_builder.py` | 构建系统提示中的 `<available_skills>` 索引；扫描 `SKILL.md` 与 `DESCRIPTION.md`；维护进程内和磁盘快照缓存；处理外部技能目录。 |
| `agent/skill_utils.py` | Skills 轻量公共工具：frontmatter 解析、平台匹配、禁用技能读取、外部技能目录、命名空间/条件/配置项提取等。 |
| `agent/skill_commands.py` | `/skill-name` 动态命令发现与调用消息构造；支持技能重载、预加载技能、技能配置注入、模板变量和 inline shell 预处理。 |
| `agent/skill_preprocessing.py` | `SKILL.md` 内容预处理：`${HERMES_SKILL_DIR}`、`${HERMES_SESSION_ID}` 替换，以及可选 inline shell 展开。 |
| `model_tools.py` | 将 `skills_tools` 工具组映射到 `skills_list`、`skill_view`、`skill_manage`。 |
| `toolsets.py` | 定义 `skills` toolset，并把 `skills_list`、`skill_view`、`skill_manage` 放入核心/平台工具集合。 |
| `hermes_constants.py` | 提供 `get_skills_dir()`、`get_optional_skills_dir()` 等 profile-aware 路径。 |

## Agent 可调用工具文件

| 文件 | 作用 |
| --- | --- |
| `tools/skills_tool.py` | 实现 `skills_list` 与 `skill_view`；读取技能元数据和正文；处理 linked files、平台兼容、setup/env 需求、插件技能解析、使用计数等。 |
| `tools/skill_manager_tool.py` | 实现 `skill_manage`：创建、编辑、patch、删除技能，以及写入/移除支持文件；包含大小限制、frontmatter 校验、安全扫描入口、pinned 保护。 |
| `tools/skills_sync.py` | 启动或更新时把仓库内置 `skills/` 同步到 profile 的 `~/.hermes/skills/`，维护 `.bundled_manifest`，尊重用户修改和删除。 |
| `tools/skills_hub.py` | Skills Hub 底层库：官方 optional skills、GitHub、well-known、ClawHub 等来源适配；安装锁文件、quarantine、audit、tap、index cache。 |
| `tools/skills_guard.py` | 外部技能安全扫描器：检测 prompt injection、密钥外传、破坏性命令、持久化、混淆等风险，并按 trust level 给出安装策略。 |
| `tools/skill_usage.py` | 技能使用遥测和 Curator 依据：`use/view/patch` 计数、agent-created 标记、pinned、active/stale/archived 状态。 |
| `tools/skill_provenance.py` | 使用 `ContextVar` 标记技能写入来源，区分前台用户请求和后台 review fork 创建的技能。 |
| `tools/env_passthrough.py` | 支持技能 frontmatter 中的 `required_environment_variables`，把声明且存在的环境变量加入安全透传白名单。 |

## CLI 与配置入口文件

| 文件 | 作用 |
| --- | --- |
| `cli.py` | 交互式 CLI 中处理 `/skills`、`/reload-skills`、`--skills` 预加载、技能 slash 命令、Cron 技能参数等。 |
| `hermes_cli/main.py` | argparse 入口；注册 `hermes skills`、`hermes curator`、`--skills`、Cron 技能参数；启动/更新/profile 创建时同步技能。 |
| `hermes_cli/_parser.py` | 顶层 CLI parser 中声明 `--skills` 和 `--ignore-rules` 等与预加载技能相关参数。 |
| `hermes_cli/commands.py` | 中央 slash command 注册表；声明 `/skills`、`/reload-skills`，并生成 Telegram/Discord 可见技能命令列表。 |
| `hermes_cli/banner.py` | CLI/TUI 欢迎信息中展示可用技能分类和数量。 |
| `hermes_cli/config.py` | 默认配置中定义 `skills`、`skills_hub`、`curator`、技能配置项扫描和 `skills.config.*` 注入。 |
| `hermes_cli/skills_hub.py` | `hermes skills ...` 与 `/skills ...` 的共享命令实现：browse/search/install/inspect/list/check/update/audit/uninstall/reset/publish/snapshot/tap。 |
| `hermes_cli/skills_config.py` | 交互式启用/禁用技能，支持全局和按平台的 `skills.disabled` / `skills.platform_disabled`。 |
| `hermes_cli/profiles.py` | profile 创建/克隆/统计时处理 `skills/`，支持 `--no-skills` 和 `seed_profile_skills()`。 |
| `hermes_cli/curator.py` | `hermes curator ...` 命令：status/run/pause/resume/pin/unpin/archive/prune/backup/rollback/list-archived。 |
| `hermes_cli/cron.py` | CLI Cron 子命令中规范化、显示、添加、编辑任务附加的 skills。 |
| `hermes_cli/kanban_db.py` | Kanban 任务可携带强制加载 skills，并在 worker 启动命令中传递 `--skills`。 |
| `hermes_cli/webhook.py` | Webhook 入口支持把请求中的 skills 传给任务。 |
| `hermes_cli/plugins.py` | 插件 API 支持 `ctx.register_skill(...)`，注册命名空间技能如 `plugin:skill`。 |
| `hermes_cli/setup.py`、`hermes_cli/claw.py` | 设置/迁移流程中引用 optional skills 目录与技能迁移目标。 |

## Gateway、TUI、Dashboard 集成文件

| 文件 | 作用 |
| --- | --- |
| `gateway/run.py` | 网关侧 `/skill-name`、`/reload-skills`、不可用技能提示、optional skill 安装提示、自动加载 channel/topic 绑定技能、启动时同步内置技能、定期触发 Curator。 |
| `gateway/session.py` | 会话 reset 后保留重新注入技能的上下文。 |
| `gateway/config.py` | 平台配置桥接，包含 `channel_skill_bindings` 等技能绑定配置。 |
| `gateway/platforms/base.py` | `MessageEvent.auto_skill` 和 `resolve_channel_skills()`，为各平台提供频道/线程自动技能绑定。 |
| `gateway/platforms/discord.py` | Discord `/skill` flat autocomplete、`/reload-skills`、频道技能绑定解析和动态刷新。 |
| `gateway/platforms/slack.py` | Slack 频道技能绑定解析并传入 `auto_skill`。 |
| `tui_gateway/server.py` | TUI 后端暴露 `skills.manage`、`skills.reload`；处理 TUI 启动技能、slash skill dispatch、Skills Hub 操作、secret capture。 |
| `ui-tui/src/components/skillsHub.tsx` | TUI 内置 Skills Hub 交互界面。 |
| `ui-tui/src/app/slash/commands/ops.ts` | TUI `/skills`、`/reload-skills` 本地命令逻辑，调用 `skills.manage` / `skills.reload`。 |
| `ui-tui/src/app/createSlashHandler.ts` | TUI slash 处理器把技能命令结果发送到会话。 |
| `ui-tui/src/components/branding.tsx` | TUI 品牌/欢迎区展示技能分类和数量。 |
| `ui-tui/src/lib/rpc.ts`、`ui-tui/src/gatewayTypes.ts` | TUI RPC 类型中包含 skill dispatch 和 skill count。 |
| `hermes_cli/web_server.py` | Dashboard API：`GET /api/skills`、`PUT /api/skills/toggle`，以及 profile skill count。 |
| `web/src/pages/SkillsPage.tsx` | Dashboard 技能管理页：搜索、分类、启用/禁用技能。 |
| `web/src/lib/api.ts` | 前端 API client 中的 `getSkills()`、`toggleSkill()` 类型和请求。 |
| `web/src/App.tsx` | Dashboard 路由中挂载 `/skills` 页面。 |
| `web/src/i18n/en.ts`、`web/src/i18n/zh.ts`、`web/src/i18n/types.ts` | Dashboard Skills 页面文案和类型。 |

## Cron、任务与执行环境相关文件

| 文件 | 作用 |
| --- | --- |
| `cron/jobs.py` | Cron job 中保存 `skills` / legacy `skill` 字段；Curator 合并/归档技能后重写 job 引用。 |
| `cron/scheduler.py` | Cron 运行前加载附加技能，拼接到有效 prompt，并扫描包含技能内容的最终 prompt。 |
| `tools/code_execution_tool.py` | 执行代码时尊重技能声明的 env passthrough 白名单。 |
| `tools/environments/local.py` | 本地终端环境变量过滤时查询 `tools.env_passthrough`。 |
| `tools/environments/docker.py` | Docker 环境把技能声明/配置允许的 env passthrough 转发到容器。 |

## Curator 生命周期文件

| 文件 | 作用 |
| --- | --- |
| `agent/curator.py` | 后台技能维护 orchestrator：读取配置、判断 idle/interval、标记 stale/archived、启动 review agent。 |
| `agent/curator_backup.py` | Curator 执行前为 `~/.hermes/skills/` 和 Cron job skill 引用创建快照，并支持回滚。 |
| `tools/skill_usage.py` | Curator 的数据来源和状态写入点。 |
| `tools/skill_provenance.py` | 确保只有后台 review fork 创建的技能被标记为 agent-created。 |
| `hermes_cli/curator.py` | Curator 的用户命令入口。 |

## 技能资产目录

| 路径 | 作用 |
| --- | --- |
| `skills/` | 内置 bundled skills 资产目录；当前发现 89 个 `SKILL.md`。包含 `references/`、`templates/`、`scripts/`、`assets/`、`workflows/`、`tests/` 等支持文件。 |
| `optional-skills/` | 官方可选技能目录；当前发现 71 个 `SKILL.md`。通过 Skills Hub 的 `official/...` 来源安装，不默认激活。 |
| `plugins/google_meet/SKILL.md` | 插件自带技能示例。 |
| `skills/**/DESCRIPTION.md`、`optional-skills/**/DESCRIPTION.md` | 分类描述，会进入技能索引/文档生成。 |

内置技能一级分类包括：

`apple`、`autonomous-ai-agents`、`creative`、`data-science`、`devops`、`diagramming`、`dogfood`、`domain`、`email`、`gaming`、`gifs`、`github`、`index-cache`、`inference-sh`、`mcp`、`media`、`mlops`、`note-taking`、`productivity`、`red-teaming`、`research`、`smart-home`、`social-media`、`software-development`、`yuanbao`。

可选技能一级分类包括：

`autonomous-ai-agents`、`blockchain`、`communication`、`creative`、`devops`、`dogfood`、`email`、`finance`、`health`、`mcp`、`migration`、`mlops`、`productivity`、`research`、`security`、`web-development`。

## 文档与索引生成文件

| 文件 | 作用 |
| --- | --- |
| `scripts/build_skills_index.py` | 构建技能索引。 |
| `website/scripts/extract-skills.py` | 从技能目录抽取网站文档所需数据。 |
| `website/scripts/generate-skill-docs.py` | 生成 bundled/optional skills 文档页。 |
| `website/docs/user-guide/features/skills.md` | 用户侧 Skills 功能说明。 |
| `website/docs/user-guide/features/curator.md` | Curator 功能说明。 |
| `website/docs/developer-guide/creating-skills.md` | 技能作者指南。 |
| `website/docs/guides/work-with-skills.md` | 使用 Skills 的指南。 |
| `website/docs/reference/skills-catalog.md` | 内置技能目录文档。 |
| `website/docs/reference/optional-skills-catalog.md` | 可选技能目录文档。 |
| `website/docs/user-guide/skills/**.md` | 由技能资产生成或维护的具体技能文档页。 |
| `website/src/pages/skills/index.tsx`、`website/src/pages/skills/styles.module.css` | 文档网站 Skills 页面。 |

## 测试覆盖文件

核心测试集中在以下路径：

| 路径/文件 | 覆盖点 |
| --- | --- |
| `tests/tools/test_skills_tool.py` | `skills_list` / `skill_view` 基础行为。 |
| `tests/tools/test_skill_manager_tool.py` | `skill_manage` 创建、编辑、删除等行为。 |
| `tests/tools/test_skills_sync.py` | bundled skills 同步和 manifest 逻辑。 |
| `tests/tools/test_skills_hub.py`、`tests/tools/test_skills_hub_clawhub.py` | Skills Hub 来源和安装逻辑。 |
| `tests/tools/test_skills_guard.py` | 外部技能安全扫描。 |
| `tests/tools/test_skill_usage.py` | 使用统计和 Curator 数据。 |
| `tests/tools/test_skill_provenance.py` | 写入来源 ContextVar。 |
| `tests/tools/test_skill_size_limits.py` | 技能内容和支持文件大小限制。 |
| `tests/tools/test_skill_view_path_check.py`、`tests/tools/test_skill_view_traversal.py` | `skill_view` 路径安全。 |
| `tests/tools/test_skill_env_passthrough.py` | 技能声明环境变量透传。 |
| `tests/tools/test_skill_improvements.py` | 技能改进/维护相关行为。 |
| `tests/agent/test_skill_utils.py` | `agent.skill_utils`。 |
| `tests/agent/test_skill_commands.py`、`tests/agent/test_skill_commands_reload.py` | 技能 slash 命令和 reload。 |
| `tests/agent/test_external_skills.py` | `skills.external_dirs`。 |
| `tests/agent/test_curator*.py` | Curator 状态、分类、报告、备份、活动判断。 |
| `tests/hermes_cli/test_skills_*.py` | CLI `hermes skills` 参数、配置、Hub、安装确认等。 |
| `tests/hermes_cli/test_chat_skills_flag.py`、`tests/cli/test_cli_preloaded_skills.py`、`tests/cli/test_cli_reload_skills.py` | CLI 会话预加载和重载技能。 |
| `tests/gateway/test_*skill*.py` | Gateway 自动技能、reload、不可用技能提示、平台频道绑定。 |
| `tests/cron/test_cron_prompt_injection_skill.py`、`tests/cron/test_rewrite_skill_refs.py` | Cron 加载技能、prompt 扫描、Curator 后引用重写。 |
| `tests/test_plugin_skills.py` | 插件命名空间技能注册与解析。 |
| `tests/website/test_generate_skill_docs.py` | 技能文档生成。 |
| `tests/skills/*.py` | 具体技能包脚本/迁移/OAuth 等资产级测试。 |

## 可排除或弱相关项

以下文件虽然搜索到 `skills` 字样，但不是 Skills 体系本身的核心实现，通常只在特定上下文里间接相关：

| 文件/路径 | 说明 |
| --- | --- |
| `hermes_cli/tips.py` | 只是展示提示文案。 |
| `hermes_cli/doctor.py`、`hermes_cli/dump.py`、`hermes_cli/backup.py` | 健康检查、诊断导出、备份中统计或包含 skills 目录。 |
| `web/src/pages/AnalyticsPage.tsx` | 展示技能使用统计，不参与技能加载/安装。 |
| `plugins/*/dashboard/*` | 插件 Dashboard 可能显示或传递 skills，但不是核心技能机制。 |

