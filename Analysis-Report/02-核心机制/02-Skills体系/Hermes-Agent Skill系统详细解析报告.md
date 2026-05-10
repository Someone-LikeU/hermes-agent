# Hermes Agent Skill 系统详细解析报告

## 1. 分析范围

本文基于 `Analysis-Report/02-核心机制/02-Skills体系/Skills体系涉及文件清单.md` 中列出的实现文件阅读源码后整理，重点覆盖 Skill 的资产结构、发现加载、工具接口、安装同步、安全边界、入口集成和 Curator 生命周期。

本次按要求忽略 `.git`、工程配置、测试文件，以及大量具体技能文件。仅抽样查看了一个技能 `skills/research/arxiv/SKILL.md` 的开头作为格式例子。

## 2. 总体定位

Hermes 的 Skill 系统本质上是“可演进的过程性 Prompt 资产层”。它不把某类任务的最佳实践硬编码到 `run_agent.py` 或工具实现里，而是把它们沉淀为可扫描、可加载、可修改、可安装、可归档的 `SKILL.md` 文档和支持文件。

从职责上看，Skill 同时承担四个角色：

1. **轻量知识索引**：系统提示只放技能名称、分类和短描述，降低默认上下文成本。
2. **按需指令注入**：真正执行任务时通过 `/skill-name`、`skill_view()`、`--skills` 或 Cron 附加技能，把完整 `SKILL.md` 注入当前任务上下文。
3. **过程性记忆**：复杂任务经验可以通过 `skill_manage()` 写回技能库，用于下一次复用。
4. **可治理资产**：通过 Skills Hub、安全扫描、使用统计、Curator 和备份回滚维护技能生命周期。

核心链路可以概括为：

```mermaid
flowchart TD
    A["skills/、optional-skills/、external_dirs、plugin skills"] --> B["frontmatter 解析与平台/禁用过滤"]
    B --> C["build_skills_system_prompt 生成 <available_skills> 索引"]
    B --> D["scan_skill_commands 生成 /skill-name 映射"]
    C --> E["AIAgent 系统提示"]
    D --> F["CLI/Gateway/TUI slash 调用"]
    F --> G["skill_view 加载完整 SKILL.md"]
    G --> H["任务上下文消费技能指令"]
    H --> I["skill_manage 更新或创建技能"]
    I --> J["usage sidecar + Curator 生命周期治理"]
```

## 3. 技能资产模型

### 3.1 存储位置

`hermes_constants.py` 定义了 profile-aware 路径：

- `get_skills_dir()` 返回当前 profile 下的 `~/.hermes/skills`。
- `get_optional_skills_dir()` 支持 `HERMES_OPTIONAL_SKILLS` 覆盖，否则使用打包或 profile 的 `optional-skills`。

运行时的单一技能安装目录是 `~/.hermes/skills/`。仓库内置 `skills/` 通过 `tools/skills_sync.py` 同步到 profile 目录；agent 新建、hub 安装、用户放入的技能都在这个 profile 目录下共存。

### 3.2 标准目录结构

`tools/skills_tool.py` 对 Skill 的结构定义是：

```text
skill-name/
  SKILL.md
  references/
  templates/
  scripts/
  assets/
```

其中 `SKILL.md` 是必需入口，其余目录按需提供长文档、模板、脚本和资产。`skill_view(name)` 首次加载主文件时会返回 `linked_files`，提示模型按需再次调用 `skill_view(name, file_path)` 读取支持文件。

### 3.3 SKILL.md frontmatter

抽样查看的 `skills/research/arxiv/SKILL.md` 使用如下字段：

```yaml
---
name: arxiv
description: "Search arXiv papers by keyword, author, category, or ID."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [Research, Arxiv, Papers, Academic, Science, API]
    related_skills: [ocr-and-documents]
---
```

系统重点读取：

- `name`：技能名，也是 slash 命令和查找的主要标识。
- `description`：系统提示和列表中的短描述。
- `platforms`：限制 `macos`、`linux`、`windows`。
- `metadata.hermes.tags`、`related_skills`：列表和文档辅助信息。
- `metadata.hermes.config`：声明技能需要的配置项。
- `required_environment_variables` / `setup.collect_secrets`：声明运行所需环境变量。
- `required_credential_files`：声明远端执行环境需要挂载的凭据文件。

## 4. 发现与索引机制

### 4.1 轻量公共解析层

`agent/skill_utils.py` 是核心轻量工具模块，刻意避免导入工具注册表、CLI 配置或 provider 解析链。它提供：

- `parse_frontmatter()`：解析 YAML frontmatter，失败时退化为简单 `key:value`。
- `skill_matches_platform()`：按当前 `sys.platform` 过滤技能。
- `get_disabled_skill_names()`：读取 `skills.disabled` 和 `skills.platform_disabled`。
- `get_external_skills_dirs()`：读取并规范化 `skills.external_dirs`。
- `iter_skill_index_files()`：递归扫描 `SKILL.md` 或 `DESCRIPTION.md`，排除 `.git`、`.github`、`.hub`、`.archive`。
- `extract_skill_conditions()`：读取 `requires_tools`、`requires_toolsets`、`fallback_for_tools`、`fallback_for_toolsets` 等条件。

这个模块是避免循环依赖的关键：prompt builder、skills tool、slash command 都能安全复用它。

### 4.2 系统提示索引

`agent/prompt_builder.py::build_skills_system_prompt()` 负责生成系统提示里的 `## Skills (mandatory)` 块。它只放分类、名称和短描述，不放完整技能内容。

它有两层缓存：

1. 进程内 LRU 缓存，key 包含本地 skills 路径、external dirs、可用 tools/toolsets、平台、禁用列表。
2. 磁盘快照 `~/.hermes/.skills_prompt_snapshot.json`，用 `SKILL.md` 和 `DESCRIPTION.md` 的 mtime/size manifest 校验。

本地 skills 会写入快照，external dirs 每次直接扫描。名字冲突时，本地技能优先，外部目录技能跳过。

索引构建时会过滤：

- 当前 OS 不兼容的技能。
- 全局或平台禁用的技能。
- 当前工具集合不满足 `requires_*` 的技能。
- 当前工具集合已具备目标能力时的 `fallback_for_*` 技能。

最终提示明确要求：任务相关时必须先用 `skill_view(name)` 加载技能；Hermes 自身问题优先加载 `hermes-agent`；发现技能问题时用 `skill_manage(action='patch')` 修复。

### 4.3 Agent 系统提示接入

`run_agent.py` 的系统提示构造顺序中，Skill 位于记忆之后、项目上下文文件之前。只有当前 agent 可用工具里包含 `skills_list`、`skill_view` 或 `skill_manage` 时，才会调用 `build_skills_system_prompt()`。

这使 Skill 系统是 toolset 驱动的：工具不可用时，不注入 Skill 索引，也不会要求模型加载技能。

## 5. Slash 技能调用

### 5.1 动态命令发现

`agent/skill_commands.py::scan_skill_commands()` 扫描本地 `~/.hermes/skills/` 和 `skills.external_dirs`，把每个可用技能映射成 `/skill-name`：

- 技能名小写。
- 空格和下划线转成 hyphen。
- 删除 Telegram 等平台不接受的特殊字符。
- 本地目录优先，重复技能名跳过。
- 遵守平台兼容和禁用配置。

缓存中记录 `name`、`description`、`skill_md_path`、`skill_dir`。

### 5.2 调用消息构造

`build_skill_invocation_message()` 会：

1. 通过 `skill_view(..., preprocess=False)` 读取技能。
2. 记录 `bump_use()`。
3. 生成一个带有重要标记的消息块，说明用户显式调用了该技能。
4. 调用 `_build_skill_message()` 注入完整内容、技能目录、配置值、setup 提示和支持文件列表。

这条路径把 Skill 作为“用户侧增强输入”送进会话，而不是静默修改系统提示。好处是消息来源清晰，也保持 provider 的消息交替规则。

### 5.3 预加载技能

`build_preloaded_skills_prompt()` 用于 CLI/TUI 启动参数 `--skills`。它和 slash 调用类似，但激活语义是“本 session 持续生效”，会把结果拼到启动时的 `system_prompt` 中。

## 6. 技能内容预处理

`agent/skill_preprocessing.py` 对 `SKILL.md` 支持两类预处理：

- 模板变量：`${HERMES_SKILL_DIR}`、`${HERMES_SESSION_ID}`。
- inline shell：`!` 反引号命令片段，由配置 `skills.inline_shell` 控制。

默认 `template_vars=True`，`inline_shell=False`。inline shell 会用 skill 目录作为 cwd 执行 `bash -c`，输出限制 4000 字符，超时默认 10 秒。

这个设计允许技能引用自身脚本或注入动态上下文，但 inline shell 是高风险能力，所以默认关闭，并在 `hermes_cli/config.py` 注释中明确只应对可信技能源开启。

## 7. Agent 工具接口

### 7.1 Toolset 暴露

`toolsets.py` 把三个 Skill 工具放入核心工具列表和独立 `skills` toolset：

- `skills_list`
- `skill_view`
- `skill_manage`

`model_tools.py` 的 legacy 映射也把 `skills_tools` 映射到这三个工具，兼容旧配置。

### 7.2 skills_list：低成本目录

`tools/skills_tool.py::skills_list()` 返回 JSON：

- `skills`: 每项只有 `name`、`description`、`category`。
- `categories`: 可用分类列表。
- `count` 和提示信息。

它使用 `_find_all_skills()`，扫描本地和 external dirs，读取 `SKILL.md` 前 4000 字符用于 frontmatter 和首段描述提取。

### 7.3 skill_view：按需加载完整技能

`skill_view(name, file_path=None)` 是核心加载器，支持：

- 直接路径：如 `mlops/axolotl`。
- 按目录名扫描：如 `axolotl`。
- legacy flat `.md` 文件。
- 插件命名空间：如 `plugin:skill`。
- 支持文件读取：如 `references/api.md`。

安全边界包括：

- `file_path` 禁止 `..`，并用 `validate_within_dir()` 确保不逃逸技能目录。
- 对不在可信 skills 目录或 external dirs 内的文件记录 warning。
- 检测常见 prompt injection 字样并记录 warning，但本地技能仍会返回内容。
- 平台不兼容或技能禁用时直接拒绝。

加载主技能时，`skill_view()` 会：

- 返回 tags、related_skills、linked_files。
- 检查 `required_environment_variables` 和 legacy `prerequisites.env_vars`。
- 在 CLI/TUI 支持时触发 secret capture。
- 将已经存在的技能声明环境变量注册到 `tools.env_passthrough`。
- 注册 `required_credential_files`，供 Docker/Modal 等远端环境挂载。
- 根据缺失依赖返回 `setup_needed`、`setup_note` 和 `readiness_status`。

工具注册层 `_skill_view_with_bump()` 会在成功加载后 best-effort 调用 `bump_view()` 和 `bump_use()`。

### 7.4 skill_manage：过程性记忆写回

`tools/skill_manager_tool.py::skill_manage()` 支持：

- `create`：新建技能目录和 `SKILL.md`。
- `edit`：完整替换 `SKILL.md`。
- `patch`：对 `SKILL.md` 或支持文件做模糊 find-and-replace。
- `delete`：删除技能。
- `write_file`：写入支持文件。
- `remove_file`：移除支持文件。

关键校验：

- 技能名和分类必须是安全的单段路径。
- `SKILL.md` 必须有 frontmatter，并包含 `name` 和 `description`。
- `SKILL.md` 内容上限 100000 字符。
- 支持文件必须位于 `references/`、`templates/`、`scripts/`、`assets/` 下。
- 支持文件大小上限 1 MiB。
- 支持文件路径禁止 traversal。
- pinned 技能拒绝 `delete`，但允许 patch/edit。

成功修改后会清理 Skill 系统提示缓存和磁盘快照，使新技能能进入后续索引。它还会更新 usage sidecar：

- `create` 且当前写入来源为 background review 时，标记 agent-created。
- `patch/edit/write_file/remove_file` 时 bump patch。
- `delete` 时 forget usage。

注意：`delete` 的实现语义是删除整个技能目录；Curator 的自动治理不会直接删除，而是归档。

## 8. 内置技能同步

`tools/skills_sync.py` 负责把仓库内置 `skills/` 同步到 `~/.hermes/skills/`。它维护 `~/.hermes/skills/.bundled_manifest`，manifest v2 格式是：

```text
skill_name:origin_hash
```

同步策略：

- 新内置技能：复制到用户 skills 目录并写入 manifest。
- 用户未修改、上游变更：自动更新。
- 用户修改过：跳过，不覆盖用户内容。
- 用户删除过：尊重删除，不重新添加。
- 上游移除：清理 manifest 项。
- `DESCRIPTION.md` 会补充复制到分类目录。

CLI 启动、gateway 启动、更新流程和 profile 初始化都会触发同步。

## 9. Skills Hub 安装体系

`tools/skills_hub.py` 是库模块，CLI/TUI slash 命令通过 `hermes_cli/skills_hub.py` 调用它。核心抽象是 `SkillSource`：

- `search(query, limit)`
- `fetch(identifier)`
- `inspect(identifier)`
- `source_id()`
- `trust_level_for(identifier)`

当前 source router 包含：

- `OptionalSkillSource`：官方 `optional-skills/`。
- `HermesIndexSource`：集中索引。
- `SkillsShSource`
- `WellKnownSkillSource`
- `UrlSource`
- `GitHubSource`
- `ClawHubSource`
- `ClaudeMarketplaceSource`
- `LobeHubSource`

`OptionalSkillSource` 把 repo 的 `optional-skills/` 暴露为 `official/...`，trust level 是 `builtin`，不会默认激活，用户通过 Hub 安装后才复制到 `~/.hermes/skills/`。

### 9.1 安装流程

`hermes_cli/skills_hub.py::do_install()` 的主流程：

1. 根据 identifier 选择 source。
2. fetch 成 `SkillBundle`。
3. URL 来源缺少 name 时要求显式 `--name` 或交互输入。
4. official 来源自动从 identifier 推导 category。
5. 写入 `.hub/quarantine/<skill>`。
6. 调用 `scan_skill()` 安全扫描。
7. 根据 trust policy 决定 allow/block/force。
8. 交互确认。
9. 从 quarantine 移到 `~/.hermes/skills/<category>/<skill>`。
10. 写 `skills/.hub/lock.json` 和 `audit.log`。
11. 清理 Skill prompt cache。

### 9.2 安装状态记录

`HubLockFile` 记录：

- source
- identifier
- trust_level
- scan_verdict
- content_hash
- install_path
- files
- metadata
- installed_at / updated_at

这让 `hermes skills list/check/update/audit/uninstall` 能区分 hub、builtin 和 local 技能。

## 10. 安全模型

### 10.1 外部技能扫描

`tools/skills_guard.py` 对外部来源技能做静态扫描，覆盖：

- 文件结构、大小、二进制、symlink。
- secret exfiltration。
- prompt injection。
- destructive command。
- persistence。
- network 行为。
- obfuscation 和不可见字符。

trust level：

- `builtin`：内置或 official。
- `trusted`：`openai/skills`、`anthropics/skills`。
- `community`：其他来源。
- `agent-created`：agent 自己写的技能，在配置开启时扫描。

策略矩阵：

| trust level | safe | caution | dangerous |
| --- | --- | --- | --- |
| builtin | allow | allow | allow |
| trusted | allow | allow | block |
| community | allow | block | block |
| agent-created | allow | allow | ask |

外部 Hub 安装总会扫描；agent 通过 `skill_manage` 创建/修改的技能只有在 `skills.guard_agent_created=true` 时扫描。

### 10.2 环境变量透传

`tools/env_passthrough.py` 解决技能运行脚本时所需环境变量被 sandbox 清理的问题。`skill_view()` 读取 `required_environment_variables` 后，会把已存在的变量注册到 session-scoped allowlist。

重要限制：Hermes provider 凭据在 `_HERMES_PROVIDER_ENV_BLOCKLIST` 中，技能不能把 `OPENAI_API_KEY`、`ANTHROPIC_TOKEN` 等主模型凭据注册为 passthrough。这是为了避免恶意技能绕过执行沙箱的凭据清理。

### 10.3 Cron 运行时二次扫描

Cron 任务可以附加 skills。`cron/scheduler.py::_build_job_prompt()` 会在运行前逐个 `skill_view()`，把技能内容拼到最终 prompt。随后 `_scan_assembled_cron_prompt()` 对“用户 prompt + skill 内容”整体做 prompt injection 扫描。

这是必要的，因为 Cron 非交互运行且可能自动批准工具调用；仅在创建任务时扫描用户 prompt 不足以覆盖运行时加载的技能内容。

## 11. 插件技能

`hermes_cli/plugins.py::PluginContext.register_skill()` 允许插件注册只读技能：

```text
<plugin_name>:<skill_name>
```

插件技能特征：

- 不进入 `~/.hermes/skills/`。
- 不出现在系统提示 `<available_skills>` 索引中。
- 必须用 qualified name 显式 `skill_view("plugin:skill")`。
- `skill_view()` 会附加 bundle context，提示同插件的 sibling skills。
- 插件禁用时，插件技能也不可用。

这种设计避免插件技能污染普通技能目录，也避免系统提示默认膨胀。

## 12. CLI / Gateway / TUI / Dashboard 集成

### 12.1 CLI

`cli.py` 在启动时扫描 slash skill commands，并支持：

- `/skill-name` 动态技能调用。
- `/skills ...` 委托给 `hermes_cli.skills_hub`。
- `/reload-skills` 重新扫描技能命令。
- `--skills` 启动预加载。
- Cron 子命令中的 `--skills`、`--add-skills`、`--remove-skills`、`--clear-skills`。

`/reload-skills` 特别注意不清理系统提示缓存。它只刷新 slash command 映射，并把 added/removed diff 排成一个 one-shot note，等下一次用户消息时 prepend 给模型。

### 12.2 Gateway

`gateway/run.py` 复用 `agent.skill_commands`：

- 平台消息中的 `/skill-name` 会构造技能消息并落入正常 agent turn。
- 平台级禁用会在执行前二次检查。
- `/reload-skills` 会刷新内存映射，并调用 adapter 的 `refresh_skill_group()`。
- 新 session 可根据 `MessageEvent.auto_skill` 自动注入一个或多个技能。
- gateway 启动时会同步内置技能。

`gateway/platforms/base.py::resolve_channel_skills()` 支持 `channel_skill_bindings`，按 channel/thread id 自动绑定技能。Discord 和 Slack adapter 会把解析结果放入 `MessageEvent.auto_skill`。

### 12.3 Discord 特化

Discord 使用单个 `/skill` 命令和 autocomplete，而不是为每个技能注册一个 slash 命令。原因是 Discord slash command payload 有大小限制；flat autocomplete 可以随着技能数量增长而不膨胀注册 payload。

`/reload-skills` 后，Discord adapter 的 `refresh_skill_group()` 重新扫描本地技能并更新 autocomplete 内存状态。

### 12.4 TUI

`tui_gateway/server.py` 暴露：

- `skills.manage`
- `skills.reload`

TUI 启动可从环境读取预加载技能，调用 `build_preloaded_skills_prompt()` 拼入系统提示。slash dispatch 发现技能命令时返回 `{type: "skill", message, name}`，前端再把 message 发送到会话。

`ui-tui/src/components/skillsHub.tsx` 提供 TUI 内置 Skills Hub 列表、inspect、install 操作。

### 12.5 Dashboard

`hermes_cli/web_server.py` 提供：

- `GET /api/skills`：返回所有技能并标注 `enabled`。
- `PUT /api/skills/toggle`：写入 enabled/disabled 状态。

`web/src/pages/SkillsPage.tsx` 是前端技能管理页，负责搜索、分类和启用/禁用。

## 13. Cron 与任务系统

`cron/jobs.py` 规范化 `skill` 和 `skills` 字段，保留 legacy 单技能字段，同时以 `skills` 列表为规范形态。

`cron/scheduler.py` 运行时加载技能：

1. 读取 job 的 `skills`，兼容 legacy `skill`。
2. 对每个技能调用 `skill_view()`。
3. 成功时 bump use。
4. 将完整技能内容拼入 prompt。
5. 找不到技能时加入提示，要求最终回复开头告知用户。
6. 对完整 assembled prompt 做注入扫描。

Curator 合并或归档技能后，`cron.jobs.rewrite_skill_refs()` 可以重写 Cron job 中的技能引用：合并到 umbrella 的技能替换为新技能，真正 pruning 的技能从列表中移除。

## 14. Curator 生命周期治理

Skill 的生命周期治理由三部分组成：

- `tools/skill_usage.py`：使用统计和状态 sidecar。
- `tools/skill_provenance.py`：写入来源 ContextVar。
- `agent/curator.py` / `agent/curator_backup.py`：后台维护、归档和回滚。

### 14.1 Usage sidecar

`.usage.json` 位于 `~/.hermes/skills/.usage.json`，记录：

- `use_count`
- `view_count`
- `patch_count`
- `last_used_at`
- `last_viewed_at`
- `last_patched_at`
- `created_by`
- `state`
- `pinned`
- `archived_at`

设计上它是 sidecar，不写进 `SKILL.md`，避免污染用户内容或制造上游同步冲突。

### 14.2 Provenance

`tools/skill_provenance.py` 用 `ContextVar` 区分前台用户请求和后台 review fork。`run_agent.py` 每次 conversation 开始会设置当前 write origin；后台 review agent 会设置为 `background_review`。

只有在 background review fork 中 `skill_manage(action="create")` 的技能，才会通过 `mark_agent_created()` 标记为 Curator 管理对象。用户前台要求创建的技能归用户所有，不会自动治理。

需要注意命名细节：`skill_usage.is_agent_created()` 在一些路径里实际表示“不是 bundled/hub installed”，而真正进入 Curator 管理还需要 `.usage.json` 中 `created_by=agent` 或 `agent_created=true`。

### 14.3 自动状态迁移

`agent/curator.py::apply_automatic_transitions()` 按配置执行：

- `stale_after_days` 后标记 stale。
- `archive_after_days` 后归档到 `.archive/`。
- pinned 技能跳过所有自动迁移。
- 最近有活动的 stale 技能可恢复 active。

默认配置中：

- Curator enabled。
- interval 7 天。
- min idle 2 小时。
- stale 30 天。
- archive 90 天。
- backup 保留 5 个快照。

### 14.4 LLM 维护 pass

Curator 会在需要时 fork 一个辅助 agent，使用 `skills` toolset 做治理。它的目标不是制造大量细粒度技能，而是把经验合并成 class-level umbrella skills。

核心不变量：

- 不碰 bundled 或 hub-installed skills。
- 不自动删除，只归档。
- pinned 技能完全跳过。
- 合并时必须声明 `absorbed_into`，便于 Cron 引用迁移。
- 运行前用 `curator_backup.snapshot_skills()` 建立快照。

## 15. 与 Memory 的边界

源码注释中反复强调：

- Memory 记录“用户是谁、偏好是什么、事实是什么”。
- Skill 记录“这类任务该怎么做、步骤是什么、坑在哪里、如何验证”。

`run_agent.py` 的后台 review prompt 会同时考虑 memory 和 skill，但写入路径不同。复杂任务、用户纠错、反复迭代出的 workflow、错误修复经验，都应优先进入 Skill。

## 16. 关键设计取舍

### 16.1 渐进式披露降低上下文成本

默认系统提示只含短索引，不含完整技能。这能让几十到上百个技能可见，而不会把上下文耗尽。模型需要时再 `skill_view()`。

### 16.2 reload 不破坏 prompt cache

`/reload-skills` 不清理系统提示缓存，只刷新 slash command 映射并在下一轮塞入 diff note。这样新增技能即使尚未进入 cached system prompt，也可通过 slash 或 `skills_list` 使用。

Hub 安装和 `skill_manage` 修改则会清理缓存，因为资产内容确实发生变化，需要后续索引更新。

### 16.3 本地优先，外部只读

`skills.external_dirs` 便于团队共享技能，但创建新技能始终写入本地 profile skills。索引冲突时本地优先，降低外部共享目录覆盖个人技能的风险。

### 16.4 插件技能显式加载

插件技能不进全局索引，必须 `plugin:skill` 显式访问。这让插件能提供专有技能，又不让默认 prompt 规模失控。

### 16.5 Curator 只治理 agent-created

通过 provenance 和 usage sidecar，系统避免误动用户手写、内置或 hub 安装技能。这个边界是 Skill 生命周期治理能自动运行的前提。

## 17. 主要风险点

1. **技能内容本身是 prompt 权限面**：本地技能被视为可信，prompt injection 只记录 warning，不阻止加载。
2. **inline shell 风险高**：开启后技能作者的命令会在主机执行，因此默认关闭。
3. **外部技能安装依赖静态扫描**：regex 扫描只能覆盖已知模式，community 来源仍需要人工审阅。
4. **`skill_manage delete` 是目录级删除能力**：虽然有 pinned 和 Curator 归档策略，但前台工具调用仍可删除技能目录，使用时应要求明确意图。
5. **环境变量透传需要谨慎**：系统已阻断 Hermes provider credential，但第三方 API key 仍会按技能声明进入沙箱。
6. **缓存一致性依赖清理点**：`skill_manage` 和 Hub 安装会清缓存；直接手工改文件后需要 `/reload-skills` 或进程重启/快照失效才能完全反映。

## 18. 一句话总结

Hermes 的 Skill 系统不是简单的“读取一堆 Markdown”，而是一套围绕 `SKILL.md` 建立的可发现、按需加载、可写回、可安装、可审计、可治理的过程性知识系统。它把 agent 的任务经验从一次性上下文升级为 profile 级资产，并通过工具、入口适配、安全扫描和 Curator 形成闭环。
