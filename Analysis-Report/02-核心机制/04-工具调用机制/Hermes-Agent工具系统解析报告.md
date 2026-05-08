# Hermes-Agent 工具系统解析报告

## 1. 扫描范围

本报告基于源码只读扫描整理，分析时忽略了 `Analysis-Report/`、`.git/`、`.github/`、`.idea/`、`node_modules/`、`__pycache__/` 等历史报告或工程配置相关目录。

这里的“工具系统”指 Hermes-Agent 暴露给模型调用的 function tools 体系，覆盖：

- 内建工具注册、发现、schema 生成、可用性过滤、执行分发。
- toolset 分组、平台默认工具集、用户配置开关。
- Agent 主循环中的工具调用校验、顺序/并发执行、结果写回。
- MCP、插件、memory provider、context engine 等动态工具来源。
- 安全审批、输出预算、schema 兼容处理、测试入口和平台接入。

## 2. 总体结论

Hermes-Agent 的工具系统是“注册表 + toolset 选择 + Agent 调度”的三层架构：

1. `tools/registry.py` 是工具注册中心。每个内建工具文件在模块顶层调用 `registry.register(...)`，把工具名、toolset、OpenAI function schema、handler、可用性检查函数等注册进去。
2. `model_tools.py` 是工具系统的公共门面。它触发内建工具发现和插件发现，向 Agent 提供 `get_tool_definitions(...)` 和 `handle_function_call(...)` 两个核心 API。
3. `toolsets.py` 是工具暴露策略。它定义基础 toolsets、场景 toolsets、平台默认 toolsets，并支持插件和 MCP 动态 toolsets。
4. `run_agent.py` 是运行期执行器。Agent 初始化时把 toolsets 解析成实际 schema；模型返回 tool calls 后，由 Agent 校验、修复、排序、并发或顺序执行，再把结果作为 `role=tool` 消息写回上下文。

内建注册工具当前通过 AST 扫描统计为 69 个。这个数字不包含运行时动态来源，例如 MCP server 工具、插件注册工具、外部 memory provider 工具、context engine 工具。

## 3. 关键文件总览

### 3.1 核心必读文件

| 文件 | 角色 | 关键职责 |
| --- | --- | --- |
| `tools/registry.py` | 工具注册中心 | 发现自注册工具文件；保存 `ToolEntry`；检查可用性；生成 OpenAI tool schema；执行 handler；维护动态 toolset alias。 |
| `model_tools.py` | 工具公共 API | 触发工具发现；解析 enabled/disabled toolsets；动态重写 schema；参数类型修正；统一分发工具调用；插件 hook 接入。 |
| `toolsets.py` | 工具集定义 | 定义 `_HERMES_CORE_TOOLS`、基础 toolsets、平台 toolsets、组合 toolsets；支持插件/MCP 动态 toolset。 |
| `run_agent.py` | Agent 工具调用主循环 | 初始化 `self.tools`；校验模型 tool calls；处理特殊 agent-level tools；顺序/并发执行；结果预算、guardrail、回调、消息写回。 |

### 3.2 内建工具实现文件

这些文件直接或间接提供模型可调用工具：

| 文件 | 注册工具 |
| --- | --- |
| `tools/browser_tool.py` | `browser_navigate`、`browser_snapshot`、`browser_click`、`browser_type`、`browser_scroll`、`browser_back`、`browser_press`、`browser_get_images`、`browser_vision`、`browser_console` |
| `tools/browser_cdp_tool.py` | `browser_cdp` |
| `tools/browser_dialog_tool.py` | `browser_dialog` |
| `tools/clarify_tool.py` | `clarify` |
| `tools/code_execution_tool.py` | `execute_code` |
| `tools/cronjob_tools.py` | `cronjob` |
| `tools/delegate_tool.py` | `delegate_task` |
| `tools/discord_tool.py` | `discord`、`discord_admin` |
| `tools/feishu_doc_tool.py` | `feishu_doc_read` |
| `tools/feishu_drive_tool.py` | `feishu_drive_list_comments`、`feishu_drive_list_comment_replies`、`feishu_drive_reply_comment`、`feishu_drive_add_comment` |
| `tools/file_tools.py` | `read_file`、`write_file`、`patch`、`search_files` |
| `tools/homeassistant_tool.py` | `ha_list_entities`、`ha_get_state`、`ha_list_services`、`ha_call_service` |
| `tools/image_generation_tool.py` | `image_generate` |
| `tools/kanban_tools.py` | `kanban_show`、`kanban_complete`、`kanban_block`、`kanban_heartbeat`、`kanban_comment`、`kanban_create`、`kanban_link` |
| `tools/memory_tool.py` | `memory` |
| `tools/mixture_of_agents_tool.py` | `mixture_of_agents` |
| `tools/process_registry.py` | `process` |
| `tools/rl_training_tool.py` | `rl_list_environments`、`rl_select_environment`、`rl_get_current_config`、`rl_edit_config`、`rl_start_training`、`rl_check_status`、`rl_stop_training`、`rl_get_results`、`rl_list_runs`、`rl_test_inference` |
| `tools/send_message_tool.py` | `send_message` |
| `tools/session_search_tool.py` | `session_search` |
| `tools/skills_tool.py` | `skills_list`、`skill_view` |
| `tools/skill_manager_tool.py` | `skill_manage` |
| `tools/terminal_tool.py` | `terminal` |
| `tools/todo_tool.py` | `todo` |
| `tools/tts_tool.py` | `text_to_speech` |
| `tools/vision_tools.py` | `vision_analyze`、`video_analyze` |
| `tools/web_tools.py` | `web_search`、`web_extract` |
| `tools/yuanbao_tools.py` | `yb_query_group_info`、`yb_query_group_members`、`yb_send_dm`、`yb_search_sticker`、`yb_send_sticker` |

### 3.3 工具辅助、安全和运行环境文件

| 文件/目录 | 作用 |
| --- | --- |
| `tools/approval.py` | 危险操作审批流，供 CLI、gateway、ACP 等表面复用。 |
| `tools/tirith_security.py`、`tools/path_security.py`、`tools/url_safety.py`、`tools/website_policy.py` | 终端、路径、URL、站点访问等安全策略辅助。 |
| `agent/file_safety.py` | 文件读取安全策略，`file_tools.py` 会引用。 |
| `tools/file_operations.py`、`tools/file_state.py`、`tools/fuzzy_match.py`、`tools/patch_parser.py`、`tools/binary_extensions.py` | 文件读写、补丁、搜索、状态和二进制判断。 |
| `tools/checkpoint_manager.py` | 文件写入或破坏性终端命令前的快照机制。 |
| `tools/tool_result_storage.py`、`tools/budget_config.py`、`tools/tool_output_limits.py` | 大工具结果持久化、每工具/每轮输出预算。 |
| `tools/schema_sanitizer.py` | 清理 JSON Schema，兼容 llama.cpp、Anthropic、Codex 等严格后端。 |
| `agent/tool_guardrails.py` | 识别重复失败、无进展循环，给出 warn/block/halt 决策。 |
| `tools/interrupt.py` | 工具执行中断标记，终端和并发 worker 会轮询。 |
| `tools/environments/*.py` | 终端后端抽象和实现：local、docker、ssh、modal、managed_modal、singularity、daytona、vercel_sandbox 等。 |
| `tools/process_registry.py` | 后台进程注册和查询，同时注册 `process` 工具。 |
| `tools/tool_backend_helpers.py`、`tools/env_passthrough.py`、`tools/managed_tool_gateway.py` | 远程/托管工具后端、环境变量透传、Nous managed tools 辅助。 |

### 3.4 动态工具和扩展文件

| 文件/目录 | 作用 |
| --- | --- |
| `tools/mcp_tool.py` | MCP client。连接 stdio/HTTP/SSE MCP server，发现远端 tools/resources/prompts，并动态注册为 Hermes 工具。 |
| `tools/mcp_oauth.py`、`tools/mcp_oauth_manager.py` | MCP OAuth 2.1/PKCE token 管理和认证提供者。 |
| `hermes_cli/mcp_config.py` | `mcp_servers` 配置的 CLI 管理入口。 |
| `mcp_serve.py` | Hermes 自身作为 MCP server 对外暴露能力的入口。 |
| `hermes_cli/plugins.py` | 通用插件系统。插件可注册工具、hook、slash command、platform adapter、image backend、context engine、skill。 |
| `plugins/**/plugin.yaml`、`plugins/**/__init__.py` | repo-shipped 插件。通用插件通过 `register(ctx)` 注册工具或 hook。 |
| `plugins/memory/**`、`agent/memory_manager.py`、`agent/memory_provider.py` | 外部 memory provider 插件体系。工具 schema 可由 memory manager 注入 Agent。 |
| `agent/context_engine.py`、`plugins/context_engine/**` | context engine 插件体系。context engine 可提供额外工具 schema。 |
| `agent/image_gen_provider.py`、`agent/image_gen_registry.py`、`plugins/image_gen/**` | `image_generate` 的后端 provider 插件体系。 |

### 3.5 平台、配置和调用入口文件

| 文件/目录 | 作用 |
| --- | --- |
| `hermes_cli/tools_config.py` | `hermes tools` / setup tools 的统一配置逻辑；解析 `platform_toolsets`；处理 MCP 默认启用、插件 toolset、平台限制、全局 disabled toolsets。 |
| `hermes_cli/config.py` | 默认配置和配置加载，包含 `toolsets`、`platform_toolsets`、`agent.disabled_toolsets`、`delegation`、`tool_loop_guardrails` 等。 |
| `cli.py` | 交互式 CLI 初始化 Agent、维护 enabled toolsets、监听 MCP 配置变化并刷新工具。 |
| `hermes_cli/main.py`、`hermes_cli/oneshot.py` | CLI 命令入口和一次性运行模式，负责解析命令行 toolsets 并传给 Agent。 |
| `hermes_cli/banner.py`、`hermes_cli/doctor.py`、`hermes_cli/dump.py` | 工具/工具集展示、诊断和配置导出。 |
| `gateway/run.py`、`gateway/session.py`、`gateway/platforms/*.py` | 消息平台入口。按平台解析 toolsets，传入 Agent，并支持 gateway 内的 MCP 刷新和平台专属工具。 |
| `gateway/platform_registry.py` | 插件平台 adapter 注册表，也会影响 `hermes-<platform>` toolset 自动生成。 |
| `tui_gateway/entry.py`、`tui_gateway/server.py` | TUI 后端入口，初始化 MCP、解析 CLI 平台工具集、向前端报告工具和 MCP 状态。 |
| `acp_adapter/entry.py`、`acp_adapter/server.py`、`acp_adapter/session.py`、`acp_adapter/tools.py` | ACP 编辑器集成入口，支持会话级 MCP server 和 `hermes-acp` 工具集。 |
| `cron/jobs.py`、`cron/scheduler.py` | 定时任务可声明 per-job `enabled_toolsets`；scheduler 在创建 Agent 前初始化 MCP。 |
| `batch_runner.py`、`rl_cli.py`、`environments/*.py` | 批处理、RL/eval 环境通过 `enabled_toolsets` 限定工具，并复用 `handle_function_call` 执行。 |

## 4. 注册与发现机制

### 4.1 内建工具发现

`tools/registry.py` 的 `discover_builtin_tools()` 会扫描 `tools/*.py`：

- 跳过 `__init__.py`、`registry.py`、`mcp_tool.py`。
- 使用 AST 判断模块顶层是否有 `registry.register(...)`。
- 对符合条件的模块执行 `importlib.import_module(...)`。
- 模块 import 时，其顶层 `registry.register(...)` 被执行，工具进入全局 `registry`。

这个设计避免了在 `model_tools.py` 中维护工具模块导入清单。新增内建工具时，只要文件里存在顶层 `registry.register(...)`，就会被发现；但是否暴露给模型仍取决于 `toolsets.py`。

### 4.2 注册项结构

`registry.register(...)` 注册的信息会被包装成 `ToolEntry`，核心字段包括：

- `name`：模型调用的 function name。
- `toolset`：工具所属工具集。
- `schema`：OpenAI function schema。
- `handler`：实际执行函数，约定返回 JSON 字符串。
- `check_fn`：可用性检查函数；失败时 schema 不会暴露给模型。
- `requires_env`：UI/doctor 可展示的依赖环境变量。
- `is_async`：是否异步 handler。
- `emoji`：显示层使用。
- `max_result_size_chars`：大输出持久化阈值。

注册表会拒绝非 MCP 工具之间的同名覆盖，防止插件或动态工具 shadow 内建工具。MCP 工具之间允许覆盖，因为 server refresh 或不同 MCP server 名称冲突属于运行期情况。

### 4.3 可用性缓存

`registry.get_definitions(...)` 只返回 `check_fn()` 通过的工具。`check_fn` 结果有约 30 秒 TTL 缓存，避免每次 Agent 初始化或每轮请求都重复探测 Docker、Playwright、API key、远程服务等状态。

### 4.4 动态工具来源

除内建工具外，工具还可以从以下渠道进入系统：

- 通用插件：`hermes_cli/plugins.py` 中 `PluginContext.register_tool(...)` 最终调用 `tools.registry.registry.register(...)`。
- MCP：`tools/mcp_tool.py` 把远端工具转为 `mcp_<server>_<tool>`，注册到 toolset `mcp-<server>`，并给 server 原名注册 alias。
- Memory provider：`run_agent.py` 初始化 memory manager 后，把 provider 的 `get_all_tool_schemas()` 直接追加到 `self.tools`，执行时走 `self._memory_manager.handle_tool_call(...)`。
- Context engine：`run_agent.py` 把 `context_compressor.get_tool_schemas()` 追加到 `self.tools`，执行时走 `context_compressor.handle_tool_call(...)`。

因此，`tools/registry.py` 是主要注册中心，但不是所有运行期工具 schema 的唯一来源；memory provider 和 context engine 是显式注入 Agent 的旁路。

## 5. Toolset 暴露策略

### 5.1 静态 toolsets

`toolsets.py` 中 `_HERMES_CORE_TOOLS` 是平台默认能力的核心列表，包含 web、terminal、file、vision、image、skills、browser、tts、todo、memory、session_search、clarify、code execution、delegation、cronjob、messaging、homeassistant、kanban 等工具。

`TOOLSETS` 定义三类工具集：

- 基础工具集：`web`、`search`、`vision`、`video`、`image_gen`、`terminal`、`file`、`skills`、`browser`、`memory`、`todo`、`clarify`、`code_execution`、`delegation` 等。
- 场景工具集：例如 `debugging` 包含 `web` + `file` + terminal/process；`safe` 包含 web/vision/image_gen，不含 terminal。
- 平台工具集：`hermes-cli`、`hermes-cron`、`hermes-telegram`、`hermes-discord`、`hermes-api-server`、`hermes-acp` 等。

`resolve_toolset(name)` 会递归展开 `includes`，去重并排序。`all` 或 `*` 是特殊别名，表示所有 toolsets。

### 5.2 动态 toolsets

`toolsets.py` 会查询 registry：

- 如果 registry 里存在静态 `TOOLSETS` 中没有的 toolset，它被视为插件/MCP 动态 toolset。
- MCP 会额外注册 alias，例如 config 中 server 名为 `github`，实际 toolset 是 `mcp-github`，但用户可以在配置里写 `github`。
- 对插件平台，如果平台名是 `foo` 且已注册到 `gateway.platform_registry`，`hermes-foo` 可以自动生成：核心工具 + 插件在 `foo` toolset 下注册的工具。

### 5.3 平台工具配置

`hermes_cli/tools_config.py::_get_platform_tools(...)` 是平台工具配置的关键函数：

- 从 `config.yaml` 的 `platform_toolsets.<platform>` 读取用户选择。
- 没有显式选择时，使用平台默认 toolset，例如 CLI 用 `hermes-cli`。
- 如果显式列出基础 toolset，则直接按基础 toolset 判断，避免组合 toolset 重新启用被用户关闭的工具。
- 默认关闭高成本或高风险工具集，例如 `moa`、`rl`、部分平台专属工具。
- 自动包含已启用的 MCP servers，除非配置里出现 `no_mcp`。
- 最后应用 `agent.disabled_toolsets`，作为全局禁用覆盖。

这一层解决的是“某个平台应该给 Agent 传哪些 toolset 名称”。真正的工具 schema 过滤仍在 `model_tools.get_tool_definitions(...)` 完成。

## 6. Schema 生成流程

Agent 初始化时，`run_agent.py` 调用：

```text
get_tool_definitions(enabled_toolsets=..., disabled_toolsets=..., quiet_mode=...)
```

`model_tools.py` 的流程如下：

1. 如果传入 `enabled_toolsets`，逐个调用 `validate_toolset()` 和 `resolve_toolset()` 展开为工具名集合。
2. 如果未传入，则默认遍历 `get_all_toolsets()`，把所有可解析工具加入候选集合。
3. 再应用 `disabled_toolsets` 做减法。
4. 调用 `registry.get_definitions(...)`，按 `check_fn` 过滤不可用工具并返回 OpenAI 格式 schema。
5. 基于最终可用工具动态重写部分 schema：
   - `execute_code`：只在 schema 中列出当前会话实际可用的 sandbox tools。
   - `discord` / `discord_admin`：根据 bot intents 和 config allowlist 重建 schema。
   - `browser_navigate`：如果 web 工具不可用，移除描述中对 `web_search` / `web_extract` 的引用。
6. 调用 `tools.schema_sanitizer.sanitize_tool_schemas(...)`，清理严格后端不接受的 JSON Schema 形态。
7. 更新 `_last_resolved_tool_names`，供 `execute_code` 等旧路径判断会话内可用工具。

`quiet_mode=True` 时，`get_tool_definitions` 使用 `(enabled_toolsets, disabled_toolsets, registry._generation, config 文件指纹)` 做缓存。registry 每次注册/注销工具或注册 alias 都会递增 `_generation`，从而让缓存自动失效。

## 7. 工具调用执行流程

### 7.1 模型请求

`run_agent.py` 的 Agent 会把 `self.tools` 传给不同 provider transport：

- Chat Completions 路径中，`agent/transports/chat_completions.py` 基本保持 OpenAI tool schema 原样。
- Responses API 路径中，`agent/transports/codex.py` 把 OpenAI function schema 转成 Responses API tool 格式。
- Anthropic、Bedrock 等 adapter 也会在各自路径中做格式转换和兼容处理。

### 7.2 tool call 校验

模型返回 tool calls 后，`run_agent.py` 会先做防御性处理：

- 尝试修复 hallucinated tool name。
- 检查工具名是否在 `self.valid_tool_names` 中。
- 检查 arguments 是否是合法 JSON；空字符串会被当作 `{}`。
- 对截断的 tool arguments 直接停止，避免执行半截参数。
- 多个重复 `delegate_task` 会被限制，重复 tool call 会去重。

如果工具名不存在或 JSON 参数不合法，Agent 不会直接崩溃，而是写入 tool error，让模型下一轮自我修正；连续失败超过阈值后才返回 partial/error。

### 7.3 顺序和并发执行

`run_agent.py::_execute_tool_calls(...)` 会判断是否可并发：

- `clarify` 等交互类工具永不并发。
- `web_search`、`web_extract`、`read_file`、`search_files`、`session_search`、`skill_view`、`skills_list`、`vision_analyze` 等只读工具可并发。
- `read_file`、`write_file`、`patch` 是路径作用域工具：只有路径不重叠时才可并发。
- 其他工具默认顺序执行。

并发执行使用 `ThreadPoolExecutor`，最多 8 个 worker。Agent 会把审批回调、sudo 回调、中断状态、activity callback 传播到 worker 线程，避免终端工具在 worker 中死等 stdin。

### 7.4 特殊 agent-level tools

`model_tools.py` 中 `_AGENT_LOOP_TOOLS = {"todo", "memory", "session_search", "delegate_task"}`。这些工具虽然在 registry 里有 schema，但需要 Agent 实例状态，因此由 `run_agent.py::_invoke_tool(...)` 特殊处理：

- `todo` 使用当前 Agent 的 `TodoStore`。
- `session_search` 使用当前 session DB 和当前 session id。
- `memory` 使用 Agent 的 `MemoryStore`，并桥接外部 memory provider 的写入通知。
- `delegate_task` 需要 parent agent，用于构造子 Agent。

此外：

- `clarify` 也由 Agent 传入 `clarify_callback`，因为它要回到用户交互层。
- memory provider 工具由 `MemoryManager` 处理。
- context engine 工具由 `context_compressor.handle_tool_call(...)` 处理。

普通工具最终进入 `model_tools.handle_function_call(...)`。

### 7.5 `handle_function_call` 的职责

`model_tools.handle_function_call(...)` 是普通工具调用的统一门面：

1. `coerce_tool_args(...)` 根据 schema 把 `"42"`、`"true"`、JSON 字符串数组等修正为目标类型。
2. 触发插件 `pre_tool_call` hook；如果 hook 返回 block 指令，则直接返回 JSON error。
3. 通知 `file_tools` 重置连续 read/search 追踪。
4. 对 `execute_code` 特殊传入当前会话可用工具列表。
5. 调用 `registry.dispatch(...)` 执行 handler。
6. 触发 `post_tool_call` hook。
7. 触发 `transform_tool_result` hook，允许插件替换工具结果。

`registry.dispatch(...)` 会：

- 找到 `ToolEntry`。
- 若 `is_async=True`，通过 `model_tools._run_async(...)` 把 async handler 接到同步调用栈。
- 捕获异常并返回标准 JSON error。

### 7.6 结果写回

工具结果返回后，`run_agent.py` 还会进行：

- `agent/tool_guardrails.py` 循环检测：重复失败、同工具失败、幂等工具无进展。
- `tools/tool_result_storage.py` 大输出持久化：超过 per-tool 阈值时把完整输出写到 sandbox 临时目录，context 中只保留预览和文件路径。
- 每轮总预算控制：一轮多个 tool result 总量过大时，优先持久化最大的结果。
- subdirectory context hints：从工具参数推断子目录上下文文件。
- 组装 `{"role": "tool", "name": ..., "tool_call_id": ..., "content": ...}` 加入 messages。

## 8. 主要工具类别解析

### 8.1 Web 工具

`tools/web_tools.py` 注册 `web_search` 和 `web_extract`，属于 `web` toolset；`search` toolset 只暴露 `web_search`。这两个工具的依赖环境由 `_web_requires_env()` 动态返回，schema 是否暴露取决于可用性检查。

### 8.2 浏览器工具

`tools/browser_tool.py` 是主浏览器自动化实现，配合：

- `tools/browser_supervisor.py`
- `tools/browser_camofox.py`
- `tools/browser_camofox_state.py`
- `tools/browser_cdp_tool.py`
- `tools/browser_dialog_tool.py`

`browser` toolset 默认包含浏览器动作工具和 `web_search`，让模型可先找 URL 再自动化网页。

### 8.3 终端和进程工具

`tools/terminal_tool.py` 注册 `terminal`，支持 local、docker、ssh、modal、managed_modal、singularity、daytona、vercel_sandbox 等后端。`tools/process_registry.py` 注册 `process`，管理后台进程状态。

终端工具是高风险工具，因此周边有审批、安全策略、checkpoint、中断、后台通知、输出截断等配套机制。

### 8.4 文件工具

`tools/file_tools.py` 注册 `read_file`、`write_file`、`patch`、`search_files`。它会使用：

- `agent/file_safety.py` 阻止敏感读取。
- `tools/file_operations.py` 执行实际文件操作。
- `tools/file_state.py` 管理文件读写状态。
- `tools/patch_parser.py`、`tools/fuzzy_match.py` 支持补丁和模糊匹配。

写入和补丁工具可触发 checkpoint。`read_file`、`search_files` 可并发，写入/补丁只有路径不重叠才并发。

### 8.5 Programmatic Tool Calling

`tools/code_execution_tool.py` 注册 `execute_code`。它允许模型写 Python 脚本，并在脚本里通过生成的 `hermes_tools.py` 调用一组白名单工具。

白名单由 `SANDBOX_ALLOWED_TOOLS` 定义：`web_search`、`web_extract`、`read_file`、`write_file`、`search_files`、`patch`、`terminal`。实际可用集合是白名单与当前会话已启用工具的交集。

该工具的价值是减少多轮 LLM 往返：复杂工具链可以在一个脚本中循环调用工具，中间结果不进入主上下文，只把最终 stdout 返回给模型。

### 8.6 Delegation 子代理

`tools/delegate_tool.py` 注册 `delegate_task`，负责构造子 `AIAgent`：

- 子 Agent 有独立对话、独立 `task_id`、独立终端会话。
- 默认继承父 Agent 的 toolsets，但会剥离 `delegate_task`、`clarify`、`memory`、`send_message`、`execute_code`。
- 如果指定 child toolsets，会与父工具集求交，防止子代理获得父代理没有的能力。
- 支持 `leaf` 和 `orchestrator` 角色，后者可在配置允许时保留 delegation 能力。

`run_agent.py` 不直接让 registry dispatch `delegate_task`，而是通过 `_dispatch_delegate_task(...)` 传入 parent agent。

### 8.7 Memory、Todo、Session Search

`todo`、`memory`、`session_search` 都有 registry schema，但运行时必须使用当前 Agent 状态：

- `tools/todo_tool.py` 提供 `TodoStore`。
- `tools/memory_tool.py` 提供内建 `MemoryStore`。
- `tools/session_search_tool.py` 查询 `hermes_state.py` 的 session DB。
- 外部 memory provider 由 `agent/memory_manager.py` 和 `plugins/memory/**` 接入。

这类工具体现了 Hermes 工具系统的一个重要特点：schema 暴露可以在 registry 中统一，但执行可以在 Agent 层按状态特殊路由。

### 8.8 Skills 工具

`tools/skills_tool.py` 注册 `skills_list`、`skill_view`，`tools/skill_manager_tool.py` 注册 `skill_manage`。相关辅助包括：

- `tools/skills_guard.py`
- `tools/skills_hub.py`
- `tools/skills_sync.py`
- `tools/skill_provenance.py`
- `tools/skill_usage.py`
- `agent/skill_utils.py`
- `agent/skill_commands.py`
- `agent/skill_preprocessing.py`

Skills 工具系统既是模型能力扩展入口，也是长期知识/流程沉淀机制。

### 8.9 MCP 工具

`tools/mcp_tool.py` 是动态工具系统里最复杂的一块：

- 从 `config.yaml` 的 `mcp_servers` 读取配置。
- 支持 stdio、Streamable HTTP、SSE transport。
- 支持 OAuth、sampling、resources、prompts。
- 连接 server 后调用 MCP `list_tools`，把远端工具 schema 转为 Hermes schema。
- 工具名规范为 `mcp_<server>_<tool>`。
- toolset 名规范为 `mcp-<server>`，并把 server 原名作为 alias 注册。
- 支持 `tools.include` / `tools.exclude` 精细过滤。
- MCP server 工具变化时可注销旧工具并重新注册。

MCP discovery 曾经是 `model_tools.py` 的模块级副作用，现在改为各入口显式初始化，避免 gateway 在 asyncio event loop 内 lazy import 时被慢 MCP server 阻塞。

### 8.10 插件工具

`hermes_cli/plugins.py` 的 `PluginContext.register_tool(...)` 会把插件工具注册到全局 registry。插件可以来自：

- repo 内置 `plugins/`
- 用户目录 `~/.hermes/plugins/`
- 项目目录 `./.hermes/plugins/`，需 `HERMES_ENABLE_PROJECT_PLUGINS`
- Python entry point group `hermes_agent.plugins`

插件还能注册 hooks。工具调用路径中的关键 hooks 包括：

- `pre_tool_call`：可阻断工具调用。
- `post_tool_call`：观察工具结果和耗时。
- `transform_tool_result`：可替换结果字符串。
- `pre_approval_request` / `post_approval_response`：观察审批流程。

## 9. 内建注册工具清单

以下为通过源码顶层 `registry.register(...)` 统计到的内建注册工具：

| toolset | 工具名 | 文件 |
| --- | --- | --- |
| `browser-cdp` | `browser_cdp` | `tools/browser_cdp_tool.py` |
| `browser-cdp` | `browser_dialog` | `tools/browser_dialog_tool.py` |
| `browser` | `browser_navigate` | `tools/browser_tool.py` |
| `browser` | `browser_snapshot` | `tools/browser_tool.py` |
| `browser` | `browser_click` | `tools/browser_tool.py` |
| `browser` | `browser_type` | `tools/browser_tool.py` |
| `browser` | `browser_scroll` | `tools/browser_tool.py` |
| `browser` | `browser_back` | `tools/browser_tool.py` |
| `browser` | `browser_press` | `tools/browser_tool.py` |
| `browser` | `browser_get_images` | `tools/browser_tool.py` |
| `browser` | `browser_vision` | `tools/browser_tool.py` |
| `browser` | `browser_console` | `tools/browser_tool.py` |
| `clarify` | `clarify` | `tools/clarify_tool.py` |
| `code_execution` | `execute_code` | `tools/code_execution_tool.py` |
| `cronjob` | `cronjob` | `tools/cronjob_tools.py` |
| `delegation` | `delegate_task` | `tools/delegate_tool.py` |
| `discord` | `discord` | `tools/discord_tool.py` |
| `discord_admin` | `discord_admin` | `tools/discord_tool.py` |
| `feishu_doc` | `feishu_doc_read` | `tools/feishu_doc_tool.py` |
| `feishu_drive` | `feishu_drive_list_comments` | `tools/feishu_drive_tool.py` |
| `feishu_drive` | `feishu_drive_list_comment_replies` | `tools/feishu_drive_tool.py` |
| `feishu_drive` | `feishu_drive_reply_comment` | `tools/feishu_drive_tool.py` |
| `feishu_drive` | `feishu_drive_add_comment` | `tools/feishu_drive_tool.py` |
| `file` | `read_file` | `tools/file_tools.py` |
| `file` | `write_file` | `tools/file_tools.py` |
| `file` | `patch` | `tools/file_tools.py` |
| `file` | `search_files` | `tools/file_tools.py` |
| `homeassistant` | `ha_list_entities` | `tools/homeassistant_tool.py` |
| `homeassistant` | `ha_get_state` | `tools/homeassistant_tool.py` |
| `homeassistant` | `ha_list_services` | `tools/homeassistant_tool.py` |
| `homeassistant` | `ha_call_service` | `tools/homeassistant_tool.py` |
| `image_gen` | `image_generate` | `tools/image_generation_tool.py` |
| `kanban` | `kanban_show` | `tools/kanban_tools.py` |
| `kanban` | `kanban_complete` | `tools/kanban_tools.py` |
| `kanban` | `kanban_block` | `tools/kanban_tools.py` |
| `kanban` | `kanban_heartbeat` | `tools/kanban_tools.py` |
| `kanban` | `kanban_comment` | `tools/kanban_tools.py` |
| `kanban` | `kanban_create` | `tools/kanban_tools.py` |
| `kanban` | `kanban_link` | `tools/kanban_tools.py` |
| `memory` | `memory` | `tools/memory_tool.py` |
| `moa` | `mixture_of_agents` | `tools/mixture_of_agents_tool.py` |
| `terminal` | `process` | `tools/process_registry.py` |
| `rl` | `rl_list_environments` | `tools/rl_training_tool.py` |
| `rl` | `rl_select_environment` | `tools/rl_training_tool.py` |
| `rl` | `rl_get_current_config` | `tools/rl_training_tool.py` |
| `rl` | `rl_edit_config` | `tools/rl_training_tool.py` |
| `rl` | `rl_start_training` | `tools/rl_training_tool.py` |
| `rl` | `rl_check_status` | `tools/rl_training_tool.py` |
| `rl` | `rl_stop_training` | `tools/rl_training_tool.py` |
| `rl` | `rl_get_results` | `tools/rl_training_tool.py` |
| `rl` | `rl_list_runs` | `tools/rl_training_tool.py` |
| `rl` | `rl_test_inference` | `tools/rl_training_tool.py` |
| `messaging` | `send_message` | `tools/send_message_tool.py` |
| `session_search` | `session_search` | `tools/session_search_tool.py` |
| `skills` | `skill_manage` | `tools/skill_manager_tool.py` |
| `skills` | `skills_list` | `tools/skills_tool.py` |
| `skills` | `skill_view` | `tools/skills_tool.py` |
| `terminal` | `terminal` | `tools/terminal_tool.py` |
| `todo` | `todo` | `tools/todo_tool.py` |
| `tts` | `text_to_speech` | `tools/tts_tool.py` |
| `vision` | `vision_analyze` | `tools/vision_tools.py` |
| `video` | `video_analyze` | `tools/vision_tools.py` |
| `web` | `web_search` | `tools/web_tools.py` |
| `web` | `web_extract` | `tools/web_tools.py` |
| `hermes-yuanbao` | `yb_query_group_info` | `tools/yuanbao_tools.py` |
| `hermes-yuanbao` | `yb_query_group_members` | `tools/yuanbao_tools.py` |
| `hermes-yuanbao` | `yb_send_dm` | `tools/yuanbao_tools.py` |
| `hermes-yuanbao` | `yb_search_sticker` | `tools/yuanbao_tools.py` |
| `hermes-yuanbao` | `yb_send_sticker` | `tools/yuanbao_tools.py` |

## 10. 新增工具的正确路径

### 10.1 首选：插件工具

对自定义或本地工具，推荐插件路径：

1. 创建 `~/.hermes/plugins/<name>/plugin.yaml` 和 `__init__.py`。
2. 在 `register(ctx)` 里调用 `ctx.register_tool(...)`。
3. 通过 `hermes plugins enable <name>` 或配置启用插件。
4. 如果需要平台可配置，插件 toolset 会出现在 `hermes tools` 的插件工具集列表中。

优点是无需改 Hermes core，适合本地工具、团队私有工具、实验工具。

### 10.2 内建 core 工具

只有当工具要随 Hermes 基础系统发布时，才走内建路径：

1. 在 `tools/<your_tool>.py` 创建 handler、schema、`check_fn`，并在模块顶层调用 `registry.register(...)`。
2. 在 `toolsets.py` 把工具名加入某个基础 toolset 或 `_HERMES_CORE_TOOLS` / 平台 toolset。

第二步很关键：自动发现只负责把工具注册到 registry；模型是否看见它，取决于 toolsets 解析结果。

### 10.3 MCP 工具

如果能力已经由 MCP server 提供，推荐配置 `mcp_servers`：

- Hermes 会连接 server、读取 tool list、生成 `mcp_<server>_<tool>` 工具名。
- 可通过 `tools.include` / `tools.exclude` 控制单个 MCP server 暴露哪些工具。
- 可通过 `platform_toolsets` 中的 server 名或 `no_mcp` 控制平台是否默认带上 MCP 工具。

## 11. 风险点和工程注意事项

1. 工具名唯一性很重要。registry 拒绝非 MCP shadow，但 memory/context engine 的 Agent 注入路径也要去重，否则部分 provider 会因重复 function name 返回 400。
2. 工具 schema 不是静态常量。`execute_code`、`discord`、MCP 工具、strict backend sanitizer 都会在暴露前动态改写 schema。
3. `enabled_toolsets=None` 语义是“所有工具集”，不是“无工具”。子代理逻辑因此需要从父 Agent 的 `valid_tool_names` 反推 toolsets。
4. MCP 初始化不能作为 `model_tools.py` import 副作用运行。gateway/TUI/cron/ACP 各自显式初始化 MCP，避免慢 server 阻塞事件循环。
5. agent-level tools 不能绕过 `run_agent.py` 状态路由。直接调用 `model_tools.handle_function_call("memory", ...)` 会得到 “must be handled by the agent loop” 类错误。
6. 高风险工具需要多层保护：toolset 开关、`check_fn`、审批、checkpoint、guardrail、输出预算缺一不可。
7. 并发工具执行只对明确安全的工具开放。新增可并发工具时，需要确认是否有共享状态、外部副作用、路径冲突。
8. 新增工具 schema 要尽量严格但兼容。复杂 JSON Schema 可能被本地/严格后端拒绝，必要时要考虑 `schema_sanitizer.py` 的规则。

## 12. 一句话架构图

```text
tools/*.py registry.register()
        ↓
tools/registry.py 保存 schema + handler + availability
        ↓
toolsets.py / hermes_cli.tools_config.py 决定本平台启用哪些 toolsets
        ↓
model_tools.get_tool_definitions() 解析 toolsets、过滤 check_fn、动态修 schema
        ↓
run_agent.AIAgent 把 tools 传给模型
        ↓
模型返回 tool_calls
        ↓
run_agent 校验/修复/并发策略/agent-level 路由
        ↓
model_tools.handle_function_call() / registry.dispatch() / 特殊 manager
        ↓
工具 JSON 结果 + guardrail + 结果持久化
        ↓
role=tool 消息写回上下文，进入下一轮模型调用
```

## 13. 建议阅读顺序

如果要继续维护或重构工具系统，建议按这个顺序读：

1. `tools/registry.py`
2. `model_tools.py`
3. `toolsets.py`
4. `run_agent.py` 中 `__init__`、`_invoke_tool`、`_execute_tool_calls`、主 loop 的 tool call 分支
5. `hermes_cli/tools_config.py`
6. `tools/terminal_tool.py`、`tools/file_tools.py`、`tools/code_execution_tool.py`、`tools/delegate_tool.py`
7. `tools/mcp_tool.py`
8. `hermes_cli/plugins.py`
9. `agent/tool_guardrails.py`、`tools/tool_result_storage.py`、`tools/schema_sanitizer.py`

