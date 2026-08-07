# 07. 全部源码包索引

本索引覆盖 `internal/` 下的生产代码目录。测试文件用于确认行为与竞态，未逐文件重复列出；生成文件已标注。建议用本表定位后，再回到前几章理解调用关系。

## A. 入口、应用与远程架构

| 包 | 职责与关键文件 |
| --- | --- |
| `internal/cmd` | Cobra/Fang CLI。`root.go` 启动 TUI 与 Workspace；`run.go` 非交互运行；`server.go` 服务进程；其余文件负责 login、models、session、stats、schema、projects、logs 等。`cmd/stats` 有独立展示约束。 |
| `internal/app` | 单 Workspace composition root。`app.go` 创建服务、Agent、MCP/LSP、事件扇入与关闭；`events.go` 管 Broker 桥接；`lsp_events.go` 保存全局 LSP 展示状态；`provider.go` 解析模型名。 |
| `internal/workspace` | UI 依赖的统一接口。`app_workspace.go` 直连 App；`client_workspace.go` 调远端并维护 SSE 状态缓存；`workspace.go` 定义能力面。 |
| `internal/backend` | 传输无关的服务端业务。管理多 Workspace、路径去重、客户端 claims、Agent 异步派发、Session/Message/Permission/LSP/MCP/Config 委托和关闭竞态。 |
| `internal/server` | Unix socket/Windows named pipe 上的 HTTP/SSE 服务器、路由、协议转换、日志、recover 与事件 stream。 |
| `internal/client` | Server 客户端、dial 平台差异、HTTP API、SSE 解码、配置/错误转换。 |
| `internal/proto` | 跨进程 DTO 与领域对象转换：agent、message、session、permission、MCP、skills、history、tools、version 和 requests。 |
| `internal/swagger` | Swagger 产物；`docs.go/json/yaml` 主要由生成流程维护。 |

## B. Agent 与扩展能力

| 包 | 职责与关键文件 |
| --- | --- |
| `internal/agent` | AI 核心。`coordinator.go` 构建 Provider/Model/Prompt/Tools；`agent.go` 执行 Session turn、队列、取消、流式消息、摘要、标题；`hooked_tool.go` 前置 Hook；`agent_tool.go` 子 Agent；`runid/run_marker` 可靠完成关联；`loop_detection.go` 工具循环保护。 |
| `internal/agent/prompt` | Go template Prompt 构建、上下文文件发现、Git/平台/工作目录和 Skills 数据注入。 |
| `internal/agent/notify` | Agent notification 与 RunComplete 数据类型。 |
| `internal/agent/hyper` | Hyper Provider 余额/credits 相关状态与请求。 |
| `internal/agent/agenttest` | 测试用 Coordinator/Agent stub，不进入产品运行路径。 |
| `internal/agent/tools` | 内置工具集合。文件、搜索、Shell、后台 job、网络、Sourcegraph、Question、Todos、LSP、MCP resource、下载、Crush 状态/日志，以及参数 schema、Markdown 描述和安全辅助。 |
| `internal/agent/tools/mcp` | MCP 生命周期、transport、OAuth、工具注册表、资源、channels、初始化 barrier 和状态。 |
| `internal/skills` | SKILL.md 解析、发现、诊断、去重、过滤、Prompt XML、Catalog、安全读取、Tracker 与 Workspace Manager。 |
| `internal/skills/builtin` | 编译进二进制的 `crush-config`、`crush-hooks`、`jq` 技能正文。 |
| `internal/hooks` | PreToolUse Hook 输入协议、Runner、超时/并行/去重和 allow/deny/input rewrite 聚合。 |
| `internal/permission` | 工具权限请求、一次/持久 grant、deny、yolo/allow list、Hook approval 与事件。 |
| `internal/question` | Agent 结构化提问批次、等待答案、取消与通知。 |

## C. 配置、Provider 与认证

| 包 | 职责与关键文件 |
| --- | --- |
| `internal/config` | `Config` 数据模型、加载/合并/默认值/校验、Provider 元数据缓存、模型解析、ConfigStore 原子写、scope、OAuth token 刷新、staleness/reload、Docker MCP 和测试辅助。 |
| `internal/shellconfig` | `crushrc` 语言。通过嵌入 Shell 注册 provider/model/mcp/lsp/permissions/hook/options builtin，并写入 ConfigBuilder。 |
| `internal/discover` | 自动探测 Ollama、LM Studio、llama.cpp、LiteLLM、OMLX 等本地模型端点，统一/enrich 结果。 |
| `internal/oauth` | 通用 OAuth token/flow 支撑。 |
| `internal/oauth/callback` | 本地 OAuth callback server 与严格回调处理；有独立 AGENTS 约束。 |
| `internal/oauth/copilot` | GitHub Copilot device/auth token 获取、刷新与存储。 |
| `internal/oauth/hyper` | Hyper OAuth 流程。 |
| `internal/oauth/mcp` | MCP OAuth discovery、PKCE、授权、callback、token refresh。 |
| `internal/env` | 可替换环境变量读取抽象，提升可测性。 |
| `internal/home` | 用户 home 目录解析。 |
| `internal/projects` | 最近/已注册项目目录持久化。 |

## D. 数据与领域服务

| 包 | 职责与关键文件 |
| --- | --- |
| `internal/db` | SQLite 连接、migration、连接复用；`sql/*.sql` 为查询源，`*.sql.go` 为 sqlc 生成代码。 |
| `internal/session` | Session CRUD、父子/内部 Agent Session、标题与 usage/cost、事件。 |
| `internal/message` | Message、Content parts、Attachment、流式保存与事件。 |
| `internal/history` | 已读文件/上下文历史的数据库服务。 |
| `internal/filetracker` | 按 Session 记录被工具修改的文件。 |
| `internal/pubsub` | 泛型 Broker、订阅生命周期、普通/可靠发布和 drop 指标。 |
| `internal/commands` | 用户命令/斜杠命令定义与解析。 |

## E. Shell、文件与语言服务

| 包 | 职责与关键文件 |
| --- | --- |
| `internal/shell` | 基于 mvdan shell 的有状态/无状态执行、handler middleware、builtin registry、jq/coreutils、命令阻断、流式捕获、PTY/平台 exec、后台 job 与消息持久化。 |
| `internal/lsp` | Language Server Manager/Client、按文件自动启动、JSON-RPC handler、诊断和符号/引用/重命名/层次请求。 |
| `internal/lsp/util` | 应用 TextEdit/WorkspaceEdit，处理 UTF 编码位置和文件操作。 |
| `internal/fsext` | 文件查找、路径展开、ignore、目录树、paste、owner 与平台 drive 辅助。 |
| `internal/filepathext` | 小型路径辅助与规范化。 |
| `internal/diff` | 文本 diff 生成。 |
| `internal/diffdetect` | 判断/解析输入是否是 diff。 |
| `internal/clipboard` | 剪贴板初始化与 supported/not-supported 平台实现。 |

## F. TUI

| 包 | 职责与关键文件 |
| --- | --- |
| `internal/ui/model` | 唯一顶层 Bubble Tea UI、页面/焦点状态、布局、消息路由、Chat、header/sidebar/status/pills、Session、LSP/MCP/Skills 状态。 |
| `internal/ui/chat` | Assistant/User/Tool 消息 item、工具 renderer 工厂、streaming Markdown、diff、selection/cache/expand/compact。 |
| `internal/ui/list` | 懒渲染滚动列表、viewport、focus、highlight、filterable 能力。 |
| `internal/ui/dialog` | Overlay stack 与模型、Session、命令、权限、认证、文件、通知、问题表单等 dialogs。 |
| `internal/ui/common` | Common 依赖容器、Markdown、Chroma、diff、高亮、ANSI16、scrollbar、按钮与共享 rendering helper。 |
| `internal/ui/completions` | 编辑器补全 popup、候选 item、过滤和键位。 |
| `internal/ui/attachments` | 输入附件集合和编辑行为。 |
| `internal/ui/styles` | Styles 结构、token 驱动 quickStyle、Theme 和 gradient。 |
| `internal/ui/diffview` | unified/split diff 布局与样式。 |
| `internal/ui/notification` | bell、OSC、native、noop 通知及平台实现。 |
| `internal/ui/anim` | Spinner/帧动画。 |
| `internal/ui/image` | 终端图片显示。 |
| `internal/ui/logo` | Crush 字符 Logo。 |
| `internal/ui/logo/example` | Logo 渲染的独立示例程序，不参与主 TUI 运行。 |
| `internal/ui/xchroma` | Chroma lexer 缓存/匹配。 |
| `internal/ui/util` | UI 共享消息与小工具。 |

## G. 并发、平台与支撑

| 包 | 职责与关键文件 |
| --- | --- |
| `internal/csync` | 并发安全的泛型 Map、Slice、Value 和工具。 |
| `internal/lock` | Unix/Windows 文件锁。 |
| `internal/log` | slog 文件设置与 HTTP 日志。 |
| `internal/event` | 遥测事件、匿名标识与 logger。 |
| `internal/herdr` | herdr pane 客户端、状态/权限/消息桥接。 |
| `internal/update` | 检查最新版本。 |
| `internal/version` | 构建版本变量。 |
| `internal/dns` | Android/特定平台 DNS 初始化。 |
| `internal/ansiext` | ANSI 字符串辅助。 |
| `internal/stringext` | 通用字符串辅助。 |
| `internal/format` | 人类可读格式化辅助。 |

## H. 阅读测试的优先级

测试不是附属物。遇到下面主题，应优先阅读同名测试，因为它们记录了最难从主流程看出的不变量：

- Agent：accepted/queued/cancel/run complete/loop detection；
- Backend：workspace 创建去重、SSE detach/reconnect、race、shutdown；
- Config：合并顺序、token refresh、staleness、shellconfig flags；
- Shell：dispatch、background、isolation、builtin pipe/context；
- LSP：自动选择 server、编码 edit；
- UI：golden、窄宽度、streaming Markdown、list viewport；
- Permission/Question/PubSub：重复解决、取消和 must-deliver。

## I. 非 Go 文件

| 路径 | 含义 |
| --- | --- |
| `README.md` | 产品概览和用户入口 |
| `docs/config/` | 配置协议与未来设计 |
| `docs/hooks/`、`HOOKS.md` | Hook 用户协议、示例与未来设计 |
| `schema.json` | 配置 Schema 产物/分发文件 |
| `go.mod`/`go.sum` | Go module 与依赖锁定 |
| `Taskfile.yml`（若存在） | build/test/lint/fmt/dev 任务 |
| `.github/workflows` | CI、发布与自动化 |
| `flake.nix` | Nix 开发/构建环境 |
| `scripts/` | 日志大小写检查、labeler 等维护脚本 |

至此，全部生产源码区域都可从本索引定位；若目录新增，应同时补充本表和对应架构章节。
