# 09. Shell、Hooks、LSP 与 MCP

这四个子系统把模型或配置连接到外部世界，也是取消、进程泄漏、权限和协议错误最集中的区域。

## 第一部分：嵌入式 Shell

## 1. 为什么不用 `exec.Command("sh", "-c", ...)`

`internal/shell` 基于 `mvdan.cc/sh/v3` 的 parser/interpreter，获得：

- POSIX shell 解析、变量、管道、重定向和函数；
- 可插入的 ExecHandler middleware；
- 进程内 builtin；
- 可测试的 stdin/stdout/stderr；
- stateful Shell 与 stateless Run 共享行为；
- 命令阻断、进程组取消和跨平台 dispatch。

这套 Shell 同时服务 Bash tool、用户显式 shell command、Hooks、`crushrc` 和配置值展开。

## 2. 两种入口

### `Shell`

有状态对象，字段保存目录、环境、变量/函数 handler 状态和 block funcs。每次 Exec 创建 Runner，执行后通过 `updateShellFromRunner` 把 cwd、环境、函数等状态带回下一次。适合 Agent Bash 工具连续执行 `cd`、`export`。

### `Run`

无状态函数，所有行为由 `RunOptions` 指定：Command、Cwd、Env、Stdin/Out/Err、KillTimeout、BlockFuncs 等。Hooks 和 Backend shell endpoint 使用它。

配套入口：

- `RunAndCapture`：缓冲 stdout/stderr，返回组合 Output/ExitCode；
- `RunAndCaptureStream`：每次写入触发 progress；
- `RunAndCapturePTY`：需要终端语义的执行；
- `RunAndPersist`：完成后回调保存命令消息。

## 3. Handler 链

`standardHandlers` 的顺序具有安全语义：

```text
script dispatch
 -> builtin handler
 -> block handler
 -> process-group OS exec
```

Middleware 由后向前包装，但逻辑效果是命令先被 script/builtin 识别，再进入 block 检查，最后运行外部程序。Builtin 在 block handler 前，因此配置 builtin 或 jq 可被进程内处理。

`BuiltinHandler` 必须从 `interp.HandlerCtx(ctx)` 取得 stdin/stdout/stderr，不能直接用 `os.Stdout`，否则管道与重定向失效。

## 4. Builtin registry

`RegisterBuiltin(name, handler)` 把实现加入注册表。`jq` 在 init 中注册；`shellconfig` 将 provider/model/mcp/lsp/hook/permission/option 注册到同一 Shell，但这些配置 builtin 通过 Context 中的 ConfigBuilder 判断是否处于配置加载，普通 Bash tool 中不会修改 Crush 配置。

长循环 builtin 必须轮询 `ctx.Err()` 并原样返回。这不只是风格：Hook timeout 依赖 Context 让 builtin 主动退出；不检查会造成 goroutine 被放弃但继续运行。

## 5. 外部进程取消

Unix `processGroupExecHandler` 将子进程放进独立进程组，取消时对整个 group 发信号，防止脚本生成的孙进程成为孤儿。Windows 使用对应进程策略。

错误被规范化为 `interp.ExitStatus`；Context 取消保留为 interrupt，`IsInterrupt` 与 `ExitCode` 让调用者区分超时、信号和普通非零退出。

执行环境会加入 Crush marker，并剥离 herdr 环境，避免 Agent 启动的子 Crush/Shell 误认为自己仍在同一受管 pane 中。

## 6. Script dispatch

当 Shell 遇到一个路径命令时，`scriptDispatchHandler` 检查文件：

- 是否存在、是否目录；
- shebang；
- 是否二进制；
- `/usr/bin/env` shebang 的参数；
- Windows drive/path 特殊形式。

Shell script 可通过当前 interpreter source 执行，从而保持内嵌状态；需要其他解释器或二进制时委托外部 exec。这里避免了“文件没有 executable bit 但有合理 shebang”与跨平台路径问题。

## 7. 后台任务

`BackgroundShellManager` 是全局并发安全注册表。Start 创建 ID、独立 Context/cancel、同步 stdout/stderr buffer 和 done channel，goroutine 执行 Shell。JobOutput 读取快照，JobKill 调 cancel，Cleanup 清除已完成任务，KillAll 用于 App 关闭。

`syncBuffer` 必须加锁，因为命令 goroutine 写、Agent 查询 goroutine读。done channel 关闭表示最终 err 和输出可稳定读取。

## 第二部分：Hooks

## 8. Hook 的位置

Hook 是工具的前置政策层。目前事件是 `PreToolUse`。Coordinator 构建工具后用 hookedTool 包装：

```text
模型 tool call
 -> Hook Runner
 -> 可能 rewrite input
 -> allow / deny / halt
 -> Permission
 -> 原工具 Execute
 -> Hook metadata/context 合入结果
```

Hook 在 Permission 前执行是刻意的：Hook 可能拒绝或修改最终动作；若明确 allow，可通过带 tool-call ID 的 Context 告知 Permission，避免重复询问。

## 9. Runner

`NewRunner` 预编译 matcher regex；坏 regex 被警告并跳过。Run 选择匹配 tool name 的 hooks，按 command 字符串去重，然后并行运行。

每个 Hook 收到：

- JSON stdin：event/session/cwd/tool_name/tool_input；
- 继承的环境；
- `CRUSH_EVENT`、`CRUSH_TOOL_NAME`、`CRUSH_SESSION_ID` 等专用变量；
- command/file_path 的快捷环境变量。

Hook 使用与 Bash tool 相同的 `shell.Run`，但不提供 BlockFuncs，因为 Hook 是用户信任的配置代码。

## 10. 超时与放弃

每个 Hook 有独立 timeout Context。Runner 用 goroutine 执行 Shell；超时后再等待一秒 abandonGrace。如果解释器仍不让出，Runner 返回 neutral 并放弃 goroutine，且绝不能再读取可能仍在被写入的 bytes.Buffer，否则产生数据竞争。

这解释了为什么 builtin 必须响应 Context，也解释了 Hook 测试要使用故意不退出的替身。

## 11. 输出协议与聚合

退出码：

- 0：解析 stdout JSON；
- 2：deny 当前工具，stderr 为原因；
- 49：halt 整个 turn；
- 其他非零：记录非阻断错误，工具继续。

支持 Crush 与 Claude Code 两种 JSON envelope。聚合规则：deny 胜 allow，allow 胜 none；halt sticky；reason/context 按配置顺序拼接；多个 `updated_input` 依次做顶层 shallow merge，后者覆盖冲突 key。

## 第三部分：LSP

## 12. Manager 的职责

`lsp.Manager` 加载 Powernap 默认 server catalog，再合并用户 LSPConfig。用户 disable 会移除默认项；用户可能用 command 当名字，`resolveServerName` 会映射到 catalog 正式名。

Manager 持有 clients、近期 unavailable 时间、ConfigStore、catalog、状态 callback，以及可替换的 now/lookPath 便于测试。

## 13. 按文件延迟启动

文件工具触发 `Manager.Start(path)`：

1. 转绝对路径并拒绝工作区外文件；
2. 并行检查所有 server；
3. 已 Ready/Starting/Disabled 则复用；
4. 用户显式配置只检查 filetype/root markers；
5. 自动 server 还检查 AutoLSP、通用/危险 command blacklist、近期 unavailable、PATH；
6. 创建 Client，以独立 Background Context + timeout 初始化；
7. 等待 ready，更新 State 并 callback。

启动 Context 不绑定单次工具请求，因为 LSP 要跨多个 Run 长期存活。找不到的自动 server 缓存 30 秒，避免每次文件读取都扫描 PATH。

## 14. Client

LSP Client 包装 Powernap，保存：配置、cwd、长期 Context、Resolver、diagnostics VersionedMap、诊断计数缓存、openFiles、atomic ServerState。

Initialize 前必须注册 server request handler，因为有些语言服务器在 initialize handshake 未返回前就发送 progress/create。处理的请求/通知包括 workspace/applyEdit、workspace/configuration、registerCapability、workDoneProgress、showMessage、publishDiagnostics。

## 15. 文件同步与诊断

首次使用文件时 OpenFile 发 didOpen 并保存版本/内容；修改时 NotifyChange 增加版本并发 didChange。RefreshOpenFiles 比较磁盘，保证外部修改被 LSP 看到。

Diagnostics 保存在 VersionedMap。计数缓存记录 map version，只有诊断变化才重新遍历，避免 UI 每帧 Copy 全量 map。`WaitForDiagnostics` 等待首次或稳定窗口，供工具在修改后收集可靠结果。

## 16. WorkspaceEdit 与字符编码

`lsp/util/edit.go` 支持 TextEdit 和 DocumentChange 中的文件创建、改名、删除。应用前排序 edits、检查重叠，按从后向前或正确偏移执行，防止前一项改变后一项坐标。

LSP position 可能是 UTF-8、UTF-16 或 UTF-32 code unit，而 Go string 索引是 byte。转换函数必须正确处理 CJK、emoji 和 surrogate pair。重实现时优先用包含多字节字符的单元测试锁定行为。

## 17. Restart 与 Shutdown

Restart 保存 open file 列表，取消旧长期 Context，优雅关闭/必要时 kill，清诊断缓存，重建底层 client、initialize、重新打开文件。Shutdown 是终态，不可复用。

Close 有 5 秒上限，因为 JSON-RPC 内部 send lock 可能不响应 Context；超时直接 Kill。Manager.StopAll 走优雅路径，KillAll 用于快速进程关闭。

## 第四部分：MCP

## 18. 全局状态与初始化 barrier

MCP 包维护并发安全的全局 sessions、states、tools、resources、prompts、pending auth、generation 和事件 Broker。App.New 先 `ArmInit()` 再 goroutine `Initialize()`；Coordinator 每次构建本轮工具表前 `WaitForInit()`。

先 Arm 再启动 goroutine非常关键，否则 WaitForInit 可能在 goroutine 尚未登记进行中时立即返回，导致本轮看不到慢启动 MCP 工具。

## 19. Transport

MCPConfig 支持：

- stdio：启动子进程并通过 stdin/stdout；
- HTTP/SSE：连接远程 endpoint；
- OAuth：需要时进入 NeedsAuth 状态，生成授权 URL；
- Docker MCP：通过配置辅助动态启停；
- channels：包装 connection，拦截特定 notification 并发布外部消息。

Unix stdio server 被放进独立进程组，取消时 kill 整组，避免 npx/node 或 signal-cli 子孙进程泄漏；WaitDelay 防止孙进程持有 pipe 导致 Wait 永久阻塞。

## 20. 状态机

典型状态：Disabled、Starting、Connected、Error、NeedsAuth。`ClientInfo` 还保存 Error、Session、tool/resource/prompt Counts、上次成功 Config 和 in-flight PendingConfig。

每次状态变化通过 `updateState` 更新 map 并发布 Event。进入 Error/Disabled/Remove 会 teardown session 和注册表内容。

## 21. 工具、资源和 Prompt 注册

连接并 initialize 后：

- ListTools 总是调用，即使 capabilities.tools 是空对象；
- enabled_tools 是 allow list，之后再应用 disabled_tools deny list；
- tools 按 MCP server 名保存在 registry，Coordinator 包装成 Fantasy tool；
- Resources 只在 capability 存在时 list，method-not-found 被视为不支持；
- Prompt 只保留/转换可消费的 user text messages；
- refresh 更新 Count 和 Connected state。

RunTool 将 JSON input 转 map，调用 CallTool，再把多个 TextContent 拼接；首个 image/audio 作为二进制媒体返回。`ensureRawBytes` 兼容 SDK 已解码和异常传来 base64 文本两种情况。

## 22. Lazy renew 与 generation

工具执行时若 session 失效，`getOrRenewClient` 可按 server 名 single-flight 重连。Generation token 防止旧的慢启动 goroutine在配置已变化或 server 已 teardown 后回写新状态。没有 generation，旧连接可能覆盖新配置的 session。

## 23. 配置热重载

`Reinitialize` 对当前 MCPConfig 与运行状态做纯函数 reconcile：

- 配置删除：Remove；
- disabled：Disable；
- Connected 且配置不变：保留；
- Starting 且 PendingConfig 等于当前：让其继续；
- 其他或配置变化：Start/Restart。

Reinitialize 全局 single-flight。运行中再收到配置写入，只设置 dirty；当前 pass 完成后再跑一轮，合并突发写入同时保证最终状态对应最新配置。

## 24. OAuth

MCP OAuth 涵盖 server metadata discovery、client registration/配置、PKCE、浏览器授权、本地 callback、token 保存与刷新。无 GUI/显式 suppress browser 时只发布 URL 给 UI。PendingAuth 与 AuthURL 通过 Workspace/Proto 暴露。

Token 属于配置中的机器状态，但 Config 比较刻意忽略内部 OAuthToken，避免刷新 token 导致无意义 restart。

## 25. 重实现顺序

1. Shell parser + 外部命令 + Context 取消；
2. stateful Shell 和 handler middleware；
3. builtin registry、capture、background；
4. Hook payload/单 Hook/聚合；
5. LSP Manager 的 filetype/root marker 选择；
6. LSP Client 生命周期、open/change、diagnostics；
7. WorkspaceEdit 编码；
8. MCP 单 stdio transport + state；
9. MCP tool registry；
10. resources/prompts、HTTP/OAuth、reconcile、generation 与 channels。
