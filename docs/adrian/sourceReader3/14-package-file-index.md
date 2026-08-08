# 14. 全包与关键文件索引

本章用于“知道需求后快速定位源码”。它覆盖所有生产代码目录，并将同包文件按职责分组。测试文件未逐个罗列，但专题章节已指出关键测试主题。

## 1. 进程入口与 CLI

### `main.go`

可选启动 pprof，随后调用 `cmd.Execute`。空白导入加载 dotenv 与平台 DNS 初始化。

### `internal/cmd`

- `root.go`：根命令、全局 flags、TUI 启动、本地/远程 Workspace 选择与初始化。
- `run.go`：非交互模式、stdin/附件、模型临时覆盖、流式/JSON 输出和 RunComplete 等待。
- `server.go`、`server_other/windows.go`：Server 进程、socket/pipe 启动与控制。
- `login.go`、`logout.go`：Provider key/OAuth 登录。
- `models.go`、`update_providers.go`：模型列表与 Catwalk 更新。
- `session.go`：Session 列表、查看、重命名、删除等 CLI。
- `stats.go`：统计 Web/终端入口；`internal/cmd/stats/AGENTS.md` 约束 HTML/CSS/JS 格式。
- `dirs.go`、`projects.go`、`logs.go`：数据目录、最近项目、日志。
- `schema.go`：配置 JSON Schema。
- `root_other.go`、`root_windows.go`：平台差异。

## 2. 单工作区装配

### `internal/app`

- `app.go`：App 字段、New、Coordinator 初始化、非交互运行、事件/关闭生命周期。
- `lsp_events.go`：进程内 LSP 展示状态和 Broker。
- `provider.go`：`provider/model` 字符串解析与歧义校验。
- `testing.go`：不启动 DB/LSP/MCP 的最小测试 App。

## 3. Workspace 与远程架构

### `internal/workspace`

- `workspace.go`：前端统一 Workspace 接口、Connection/LSP/MCP 前端类型。
- `app_workspace.go`：本地 App 适配。
- `client_workspace.go`：HTTP 委托、状态缓存、SSE 消费、重连、Workspace recreate、Proto 转换。

### `internal/backend`

- `backend.go`：Backend/Workspace/clientState、创建去重、claims、grace、teardown、idle shutdown。
- `agent.go`：fire-and-forget Run、AcceptedRun、RunComplete fallback、Agent/Shell 操作。
- `events.go`：App event 到 Proto event 的转换/订阅。
- `session.go`、`filetracker.go`、`permission.go`、`question.go`、`config.go`：各领域门面。
- `util.go`：路径、Workspace DTO、client ID 等辅助。
- `testing.go`：测试构造与可调超时。

### `internal/server`

- `server.go`：Server、ServeMux 路由、listener 生命周期。
- `proto.go`：大量 HTTP handler、输入输出和错误映射。
- `events.go`：SSE 编码与 Workspace attach/detach。
- `config.go`：配置相关 handler。
- `socket.go`、`socket_classify.go`、`net_other/windows.go`：socket/pipe 与 stale socket。
- `logging.go`、`recover.go`：请求日志、panic recovery。

### `internal/client`

- `client.go`：Client、transport dial、基础 HTTP 方法、health/version/server control。
- `proto.go`：Workspace/Session/Agent/LSP/MCP API 调用。
- `config.go`：配置、OAuth、Skills、Docker MCP 调用。
- `errors.go`：HTTP/Proto 错误转换。
- `dial_other.go`、`dial_windows.go`：平台 transport。

### `internal/proto`

- `proto.go`：Workspace、AgentInfo、RunComplete、Permission/Question、LSP 基础 DTO。
- `message.go`：Message 多态 parts、附件与 JSON 编解码。
- `agent.go`、`session.go`、`permission.go`、`history.go`：领域 wire 类型。
- `mcp.go`、`skills.go`：MCP/Skills 状态和事件。
- `requests.go`：Config/LSP/MCP 请求体。
- `tools.go`：工具参数/metadata DTO，部分 alias 内部工具类型。
- `server.go`、`version.go`：控制与版本协商。

### `internal/swagger`

`docs.go` 是生成的 OpenAPI 文档，不应直接手改。

## 4. Agent 核心

### `internal/agent`

- `coordinator.go`：Coordinator 接口/实现、Provider 创建、Model 更新、工具装配、OAuth/AWS 重试。
- `agent.go`：SessionAgent、Run/stream、消息持久化、队列、取消、摘要和标题。
- `agent_tool.go`：子 Agent 工具与子 Session。
- `agentic_fetch_tool.go`：探索型子 Agent/抓取。
- `hooked_tool.go`：Hook 装饰器与 Permission 交接。
- `runid.go`、`run_marker.go`：远程 Run 关联与完成 marker。
- `loop_detection.go`：重复工具交互签名和循环保护。
- `usage_fallback.go`：Provider usage 缺失时估计/兼容。
- `aws_sso_refresh.go`：Bedrock SSO 认证刷新。
- `prompts.go`：嵌入模板和 coder/task prompt 构造。
- `event.go`、`errors.go`：事件/错误辅助。

子包：

- `agent/prompt/prompt.go`：模板数据、上下文文件、Git/平台/路径。
- `agent/notify/notify.go`：Notification、RunComplete。
- `agent/hyper/provider.go`：Hyper credits。
- `agent/agenttest/coordinator.go`：测试替身。
- `agent/templates/*`：coder、task、initialize、summary、title、agentic fetch 等提示模板。

## 5. 内置工具

### `internal/agent/tools`

基础与注册：

- `tools.go`：工具名、context keys、构造和公共类型。
- `safe.go`：路径/命令安全与权限参数。
- `rg.go`、`search.go`：搜索底层辅助。

文件工具：

- `view.go`、`write.go`、`edit.go`、`edit_whitespace.go`、`multiedit.go`。
- `glob.go`、`grep.go`、`ls.go`。
- `download.go`。

Shell/任务：

- `bash.go`、`job_output.go`、`job_kill.go`。

网络：

- `fetch.go`、`fetch_helpers.go`、`fetch_types.go`。
- `web_fetch.go`、`web_search.go`、`sourcegraph.go`。

交互/状态：

- `question.go`、`todos.go`、`crush_info.go`、`crush_logs.go`。

LSP：

- `diagnostics.go`、`references.go`、`lsp_helpers.go`。
- `lsp_definition.go`、`lsp_symbols.go`、`lsp_rename.go`、`lsp_replace_symbol.go`。
- `lsp_call_hierarchy.go`、`lsp_restart.go`。

MCP exposure：

- `mcp-tools.go`、`list_mcp_resources.go`、`read_mcp_resource.go`。

每个主要工具旁的 `.md`/`.md.tpl` 是给模型看的描述，也属于行为的一部分。

### `internal/agent/tools/mcp`

- `init.go`：全局状态、初始化 barrier、transport/session、状态更新、renew。
- `lifecycle.go`：配置 reconcile、single-flight reinitialize。
- `tools.go`、`resources.go`、`prompts.go`：三类 registry 和调用。
- `channel.go`：channel notification gate/发布。
- `process_unix.go`、`process_other.go`：stdio 子进程组清理。

## 6. Prompt、Skills、命令

### `internal/skills`

- `skills.go`：frontmatter 解析、发现、诊断、去重、过滤、Prompt XML。
- `embed.go`：`//go:embed builtin/*` 和 `crush://skills/`。
- `manager.go`：Workspace 快照、事件、GlobalMirror 兼容。
- `catalog.go`：来源标签、安全读取、有效 Skill 查找。
- `tracker.go`：记录本次 Agent 已加载 Skill。
- `builtin/*/SKILL.md`：crush-config、crush-hooks、jq。

### `internal/commands`

`commands.go` 从项目/全局 Markdown、user-invocable Skills 和 MCP Prompts 生成斜杠命令，解析 `$ARGUMENTS` 等参数。

## 7. 配置与 Provider

### `internal/config`

- `config.go`：所有配置结构、默认 Agent tools、解析辅助。
- `load.go`：路径发现、JSON/shell 合并、默认值、Provider/Model 解析与验证。
- `store.go`：并发/原子写、scope mutation、OAuth refresh、staleness/reload。
- `resolve.go`：Shell 变量 resolver 与错误脱敏。
- `provider.go`、`catwalk.go`、`hyper.go`：Provider catalog/cache/update。
- `docker_mcp.go`：Docker MCP 配置持久化。
- `init.go`：项目初始化状态。
- `scope.go`：Global/Project scope。
- `atomicwrite*.go`：跨平台原子替换。
- `copilot.go`：Copilot token/config 辅助。
- `config_unix/windows.go`：平台目录/默认差异。

### `internal/shellconfig`

- `load.go`：执行 `crushrc` 并得到数据。
- `builder.go`：Context 中的 ConfigBuilder 和 merge 操作。
- `register.go`：把 builtin 注册进 Shell。
- `provider.go`、`model.go`、`mcp.go`、`lsp.go`、`hook.go`、`permissions.go`、`options.go`：命令实现。
- `flags.go`：统一 flag parsing/类型转换。

### `internal/discover`

- `discover.go`：OpenAI-compatible `/models` 发现。
- `enricher.go`：按 Provider type 注册 metadata enricher。
- `ollama.go`：`/api/show` 上下文窗口。
- `lmstudio.go`、`llamacpp.go`、`litellm.go`、`omlx.go`：各本地服务原生 metadata。

## 8. OAuth

- `internal/oauth/token.go`：通用 token 数据。
- `oauth/callback/page.go`：callback HTML 模板；修改需遵守同目录 AGENTS。
- `oauth/copilot/*`：device flow、HTTP、URL、磁盘 token。
- `oauth/hyper/device.go`：Hyper device flow。
- `oauth/mcp/handler.go`：MCP metadata/PKCE/callback/refresh。
- `oauth/mcp/savingtokensource.go`：刷新时同步保存 token。

## 9. 数据模型与存储

### `internal/db`

- `db.go`：连接复用、migrations、release。
- `connect.go`、`connect_modernc.go`、`connect_ncruces.go`：driver 选择。
- `datadirlock.go`：Server data directory 锁。
- `models.go`：DB 层小类型。
- `*.sql.go`、`querier.go`：sqlc 生成代码。
- `sql/*.sql`：手写 Session/Message/File/ReadFile/Stats 查询。
- `migrations/*`：Schema 演进事实来源。

### 领域服务

- `internal/session/session.go`：Session CRUD、父子会话、usage/cost、事件。
- `internal/message/message.go`：Message Service 和流式保存。
- `internal/message/content.go`：领域 ContentPart。
- `internal/message/attachment.go`：多模态附件。
- `internal/history/file.go`：会话文件历史。
- `internal/filetracker/service.go`：按 Session 记录已读文件。
- `internal/permission/permission.go`：权限 pending/resolve/persist/事件。
- `internal/question/question.go`：结构化问题 pending/answer/cancel。
- `internal/pubsub/broker.go`、`events.go`：泛型事件总线。

## 10. Shell 与 Hooks

### `internal/shell`

- `shell.go`：有状态 Shell、block funcs、Exec。
- `run.go`：无状态入口、handler chain、capture/PTY。
- `dispatch.go`：路径/shebang/env/binary 分派。
- `exec_unix.go`、`exec_windows.go`：外部进程与取消。
- `builtins_registry.go`、`jq.go`、`coreutils*.go`：进程内命令。
- `background.go`：后台任务注册表。
- `stream.go`：progress writer。
- `expand.go`：配置字符串展开。
- `persist_message.go`：Shell 输出保存为消息。

### `internal/hooks`

- `input.go`：stdin/env 和两种输出协议解析。
- `runner.go`：matcher、并行、去重、timeout/abandon。
- `hooks.go`：Decision、metadata、聚合和 input shallow merge。

## 11. LSP

- `internal/lsp/manager.go`：默认 catalog、配置合并、按文件懒启动、停止。
- `internal/lsp/client.go`：Powernap Client、文件同步、诊断、符号/引用/层次请求、restart。
- `internal/lsp/handlers.go`：server request/notification handlers。
- `internal/lsp/util/edit.go`：TextEdit/WorkspaceEdit 与字符编码。

## 12. TUI

### 顶层和布局

- `ui/model/ui.go`：唯一 Bubble Tea Model、Update/View/Draw、状态/焦点。
- `model/chat.go`：Chat 包装 List。
- `model/session.go`、`landing.go`、`onboarding.go`：页面流程。
- `model/header.go`、`sidebar.go`、`status.go`、`pills.go`：区域绘制。
- `model/lsp.go`、`mcp.go`、`mcp_auth.go`、`skills.go`：领域状态。
- `model/keys.go`、`filter.go`、`history.go`、`workspace_cache.go`：输入与缓存。

### Chat

- `ui/chat/messages.go`：MessageItem 接口和公共嵌入行为。
- `assistant.go`、`user.go`：两类消息。
- `tools.go`：Tool item factory/接口/状态。
- `bash.go`、`file.go`、`search.go`、`fetch.go`、`agent.go`、`mcp.go` 等：专用 renderer。
- `streaming_markdown.go`：增量 Markdown。
- `unified_diff.go`、`tool_result_content.go`：diff/结果处理。

### List、Dialog 和组件

- `ui/list/*`：懒渲染 viewport、item、focus、highlight、filterable。
- `ui/dialog/dialog.go`、`actions.go`、`common.go`：Overlay/接口/共享布局。
- `ui/dialog/models*`、`sessions*`、`permissions.go`、`question_*`、认证/文件/命令等：具体 Dialog。
- `ui/completions/*`：补全过滤与键位。
- `ui/attachments/attachments.go`：输入附件。

### 渲染基础

- `ui/styles/styles.go`：语义 Styles 形状。
- `quickstyle.go`：token 驱动基础样式。
- `themes.go`、`grad.go`：具体 Theme/渐变。
- `ui/common/*`：Markdown、Chroma、diff、ANSI、scrollbar、按钮。
- `ui/diffview/*`：统一/分栏 diff。
- `ui/notification/*`：bell/OSC/native/noop。
- `ui/image/image.go`：终端图片。
- `ui/anim/anim.go`、`ui/logo/*`、`ui/util/util.go`、`ui/xchroma/chroma.go`：动画、Logo、辅助和 lexer cache。
- `ui/logo/example/main.go`：独立 Logo 示例，不进入主程序。

## 13. 通用基础包

- `internal/csync`：并发 Map/Slice/Value/VersionedMap。
- `internal/fsext`：路径展开、向上查找、ignore-aware glob/list、粘贴路径、跨平台 owner/drive。
- `internal/filepathext`：SmartJoin/绝对路径/glob prefix。
- `internal/diff`、`diffdetect`：生成和识别 diff。
- `internal/ansiext`、`stringext`：ANSI 转义和字符串规范化/base64。
- `internal/clipboard`：平台剪贴板。
- `internal/home`：home/config 路径缩写/展开。
- `internal/lock`：跨进程 advisory file lock。
- `internal/log`：slog setup、HTTP body/header debug、panic recovery。
- `internal/event`：PostHog 遥测、匿名 ID、事件 helper。
- `internal/projects`：最近项目列表。
- `internal/format`：非交互 spinner。
- `internal/herdr`：herdr pane 状态上报与 App 事件 bridge。
- `internal/update`：GitHub release 更新检查。
- `internal/version`：构建版本/ID。
- `internal/dns`：Android/Termux DNS fallback。

## 14. 从测试反查行为

最值得优先搜索的测试关键词：

```text
accepted / queued / cancel / run_complete
workspace / multiclient / detach / recover / retire
staleness / atomic / refresh / merge
hook / timeout / abandon / rewrite
diagnostics / utf16 / utf32 / workspace_edit
streaming_markdown / narrow / resize / cache / golden
must_deliver / duplicate / race
```

当主代码注释说明“为什么”时读注释；当你需要知道精确边界输入/输出时读 table-driven tests；当你要修改 TUI 布局时读 golden。
