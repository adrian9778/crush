# 04. 数据层与基础设施

## 1. Session 服务

`internal/session/session.go` 定义 Session 领域对象和 Service。职责包括创建普通、标题、任务 Session，读取最近/列表，保存、重命名、删除，更新标题及 token/cost，以及编码/解析 Agent Tool Session ID。

服务用 DB Querier 持久化，并通过 Broker 发布 Created/Updated/Deleted 事件。删除 Session 时数据库外键/事务要保证消息和文件记录一致。父子 Session 字段用于隐藏内部任务会话并支持层级追踪。

## 2. Message 与 Content

`internal/message/message.go` 管理消息元数据：ID、SessionID、Role、创建/更新时间、模型与 Provider、完成状态等。`content.go` 定义一条消息中的多种 part，例如文本、推理、工具调用、工具结果、附件相关内容、错误、finish/usage。

消息在流式生成中会多次保存更新，因此 Message Service 既要保持 part 顺序，也要发更新事件。UI 根据 Content 具体类型构建不同 chat item；跨进程时由 `proto/message.go` 做 DTO 转换。

Attachment 独立建模文件名、MIME、URL/内容等，进入 Provider 前转换为模型支持的文本或多模态 part。

## 3. SQLite 与 sqlc

`internal/db` 的层次：

- `migrations/`：Schema 演进，是数据库结构事实来源；
- `sql/*.sql`：手写查询，是数据访问行为来源；
- `*.sql.go`、`querier.go`：sqlc 生成的类型安全 Go；
- `db.go`：连接、迁移、连接池复用和 release；
- `connect_modernc.go`/`connect_ncruces.go`：不同 SQLite 驱动/平台方案。

主要表围绕 sessions、messages、files/read_files 与 stats。复杂跨表操作由 Service 持有 `*sql.DB` 开事务，而简单 CRUD 直接走 Querier。修改数据模型时顺序通常是：新增 migration → 修改 SQL → 重新生成 sqlc → 改 Service/Proto/UI → 测试旧库升级。

数据库连接按数据目录复用并计数，`db.Release` 只在最后一个引用释放时关闭底层连接。

## 4. History 与 FileTracker

`history.Service` 记录 Agent 已读文件或与提示历史相关的持久信息，帮助系统构造上下文并避免无依据操作。`filetracker.Service` 记录某 Session 修改过的文件，向 UI/统计提供“本轮触碰了什么”的事实。

二者名字容易混淆：History 更偏已读取/上下文历史，FileTracker 更偏写操作结果。文件工具应在成功后更新 Tracker，而不是在尝试前记录。

## 5. Permission

`permission.Service` 将工具执行请求标准化为 Session、Tool、Action、Path/Command 等数据：

- yolo/skip 模式直接允许；
- 配置 allow list 可自动允许；
- Session 级 auto approve 可允许后续请求；
- 已持久允许的 `PermissionKey` 可复用；
- Hook 明确批准可通过 Context 跳过重复询问；
- 其余请求发布 must-deliver 事件并等待 UI grant/deny 或 Context 取消。

GrantPersistent、Grant、Deny 都必须只解决仍在等待的请求，避免重复点击或迟到事件改变已完成状态。通知 Broker 与请求 Broker 分开，前者用于状态反馈，后者用于实际交互。

## 6. Question

`question.Service` 与 Permission 类似，但承载模型主动提出的问题批次。每个问题可为确认、单选、多选、自由文本等类型。Service 保存等待中的请求，UI dialog 提交答案或取消，Agent 工具同步等待结果。Client/Server 模式必须把批次、回答和通知完整映射。

## 7. Pub/Sub

Broker 为每个订阅者创建 channel，通过订阅 Context 自动移除。`Publish` 不能让一个慢 UI 阻塞 Agent，因此可丢弃并计数；`PublishMustDeliver` 会在超时/Context 范围内尝试可靠交付，也有独立 drop 计数。

事件类型是 Created、Updated、Deleted 等通用语义；PayloadType 用于跨进程识别具体领域对象。新增事件时要同时考虑序列化类型标签和旧客户端兼容。

## 8. Shell

`internal/shell` 基于 `mvdan.cc/sh/v3`，而不是简单 `exec.Command("sh", "-c", ...)`。

两种入口：

- `Shell`：有状态，连续命令保留 cwd、变量、函数，供 Bash 工具；
- `Run`：无状态，按 Options 执行，供 Hooks、后端 shell command 等。

执行 handler 链顺序很重要：builtin interception → block functions → 默认 OS exec。builtin 从 handler Context 取 stdin/stdout/stderr，因此支持管道与重定向。注册表当前包括 `jq` 和 shellconfig 在特定 Context 下使用的配置 builtin。

`dispatch.go` 负责解析可能需要特殊处理的命令块；`exec_unix/windows` 做平台执行；`expand.go` 提供变量展开；`stream.go` 做流式 capture；`background.go` 管后台进程；`persist_message.go` 把用户手动 shell 输出作为消息保存。

超时依赖 Context 传播。任何 builtin 的无界循环必须主动检查 `ctx.Err()`，否则 Hook timeout 无法真正终止 goroutine。

## 9. Hooks

`internal/hooks` 分三层：

- `input.go`：构造 JSON stdin、环境变量并解析 stdout；
- `runner.go`：匹配、并发执行、超时、去重；
- `hooks.go`：决定类型与聚合规则。

Hook 是 Agent 工具前置政策，不是 Shell 的权限系统替代品。它可以修改输入，因此修改后应以最终输入进行执行和展示。Hook 运行使用与 Bash 工具共享的嵌入 Shell handler，行为应保持一致。

## 10. LSP

`lsp.Manager` 根据文件路径、扩展名、root markers 和用户配置选择 Server。配置来源包含用户 `LSPConfig` 与 Powernap 默认目录。按需 Start 可避免所有语言服务器在启动时一起运行；近期不可用缓存防止反复启动失败进程。

`lsp.Client` 包装 JSON-RPC 客户端，维护 ServerState、打开文件版本、诊断 map 和回调。初始化后注册 diagnostics、workspace configuration、capability、applyEdit 等 handler。文件工具修改文件时触发 open/change/refresh，使诊断与磁盘同步。

`lsp/util/edit.go` 将 LSP UTF-16/UTF-32 位置转换为字节偏移，校验重叠 edits，并应用 TextEdit、Create/Rename/Delete 等 WorkspaceEdit。这里最危险的是把字符索引当字节索引；包含 emoji/CJK 的测试不可省略。

## 11. MCP

MCP 是全局注册与每 Workspace 配置交叉的子系统。初始化读取已启用 server，按类型创建 transport，完成能力协商，注册 tools/resources/prompts，并发布状态。关闭时停止 session/transport。

HTTP MCP 可能需要 OAuth：发现授权服务器元数据、PKCE、浏览器回调、本地 callback server、token 存储与刷新。Channel 是显式 opt-in；同一路径 Workspace 被多个客户端复用时，channels 必须一致，否则 Backend 返回 `ErrChannelOptInMismatch`，避免后来的客户端悄悄扩大已有 Workspace 的外部能力。

## 12. Skills

Skills 来源包括编译进二进制的 `internal/skills/builtin/*`、全局目录、项目目录和配置路径。发现步骤读取 YAML frontmatter，验证 name/description/目录关系，产生成功或诊断 `SkillState`，按名字去重；后出现的用户 Skill 可覆盖 builtin。

Builtin 通过 `//go:embed builtin/*` 编译进程序，路径显示为 `crush://skills/...`，View 工具从 embedded FS 读取，不访问磁盘。`Manager` 保存本 Workspace 的 all/active/states 快照并发布变化；可选 GlobalMirror 兼容本地 TUI 的全局状态读取。`Catalog` 负责安全解析项目/全局/内置标签并阻止父目录逃逸。

## 13. Server、Client 与 Proto

`proto` 定义请求响应、Session、Message、Agent、Permission、Skills、MCP、History、Version 等传输对象与转换函数。

`server` 负责 socket、HTTP route、Swagger、错误恢复、请求日志与 SSE。平台文件在 Unix 使用 socket、Windows 使用 named pipe。Server 不直接实现业务规则，而是调用 Backend。

`client` 封装 dial、HTTP 调用、错误解码与 SSE/事件协议。`ClientWorkspace` 进一步把远端 API 翻译成 `workspace.Workspace` 接口，并缓存 Session、Message、Agent、LSP、MCP、Skills 状态供同步 UI 使用。

`backend` 是传输无关业务层：创建/去重/关闭 Workspace，转发 Session/Message/Agent/Permission/LSP/MCP 操作，管理 client claim 和可靠事件。这样未来接入不同协议无需复制 Workspace 生命周期规则。

## 14. 其他基础包

- `csync`：泛型并发 Map、Slice、Value 等小容器；
- `fsext`、`filepathext`：路径展开、查找、ignore、目录列举与文件安全辅助；
- `diff`、`diffdetect`：生成/识别 diff；
- `format`、`ansiext`、`stringext`：展示和字符串辅助；
- `clipboard`：按平台选择剪贴板能力；
- `log`、`event`：文件日志、HTTP 日志与遥测；
- `herdr`：在 herdr 管理的 pane 中桥接 Agent、权限和消息状态；
- `commands`：斜杠命令定义/发现；
- `question`：结构化问答；
- `dns`：特定平台 DNS 修正。
