# 02. 启动、CLI 与应用装配

> 状态：待复核生成稿｜生成日期：2026-08-14
> 基准提交：`5712d4839a6a10e9940804d511bb322dbe73a511`｜工作区：clean（开始分析时）
> 源码范围：`main.go`、`internal/cmd/`、`internal/app/`、`internal/workspace/`
> 生成方式：源码、测试、配置与部署资产静态分析

## 快速摘要

### 架构总览（模块与依赖）
`main` 是最小入口，`internal/cmd` 管理 CLI 与运行形态，`app.New` 是单工作区依赖装配中心，Workspace 适配本地和远程实现。

### 核心调用序列（逐步逻辑）
1. `main` 调 `cmd.Execute`。2. Cobra 解析 cwd、data-dir、模式与子命令。3. setup 创建 Workspace，随后运行 TUI 或非交互 Agent。

### 易错点与边界条件
早期日志初始化存在特殊处理；远程初始化受请求 context 与后台生命周期边界约束；清理顺序必须与构造依赖相反。

> 本章回答三个问题：`crush` 进程怎样启动？一个命令怎样得到完整工作区？各项服务怎样被装配成可运行应用？目标不是只会调用，而是能够按本文重新搭出启动骨架。

## 1. 先建立整体心智模型

Crush 的入口很薄，真正的启动链是：

```text
操作系统启动进程
  -> main.main
  -> cmd.Execute
  -> Cobra/Fang 解析命令、参数和信号
  -> 某个 Command.RunE
  -> setupWorkspace
       -> 本地模式：config -> db -> skills -> app.New -> AppWorkspace
       -> C/S 模式：连接/启动 server -> ClientWorkspace
  -> 交互 TUI 或非交互 agent run
  -> cleanup：停止服务、释放 DB、关闭连接
```

这套设计刻意把职责分开：`main` 只负责进程入口，`cmd` 负责用户接口和生命周期，`app` 负责领域服务装配，`workspace` 屏蔽本地与远程实现差异。

## 2. `main.go`：最小进程入口

关键文件：`main.go`。

### 2.1 包导入的副作用

- `_ "github.com/joho/godotenv/autoload"` 在程序启动时读取 `.env`。
- `_ "internal/dns"` 安装项目自己的 DNS 行为。
- `net/http/pprof` 以空白导入注册性能分析路由。

### 2.2 `main()` 的两件事

1. 若存在 `CRUSH_PROFILE`，后台在 `localhost:6060` 启动 pprof HTTP 服务。监听失败只记日志，不阻止主程序。
2. 调用 `cmd.Execute()`，把控制权交给 CLI 层。

复刻时应保持入口无业务状态；否则测试单个命令会被迫启动整个程序。

## 3. 根命令的建立

关键文件：`internal/cmd/root.go`。

包级 `init()` 完成命令树注册。全局参数包括：

| 参数 | 作用 |
|---|---|
| `--cwd/-c` | 指定工作目录 |
| `--data-dir/-D` | 覆盖 Crush 数据目录 |
| `--debug/-d` | 调试日志 |
| `--host/-H` | Client/Server 模式连接地址 |
| `--yolo/-y` | 跳过权限询问，危险 |
| `--session/-s` | 继续指定会话 |
| `--continue/-C` | 继续最近会话 |
| `--channels` | 隐藏参数，选择启用的 MCP channel |

`session` 与 `continue` 互斥。子命令有 `run`、`dirs`、`projects`、`update-providers`、`logs`、`logout`、`schema`、`login`、`stats`、`session`。

### 3.1 `Execute()`

执行顺序：

1. 暂时把 `slog` 指向丢弃 handler。原因是正式日志路径取决于尚未加载的数据目录；配置加载期间直接写 stderr 会污染 TUI。
2. 若 stdout 是终端，为版本输出增加彩色 Heartbit 图案。
3. `fang.Execute(context.Background(), rootCmd, ...)` 执行 Cobra 命令，并注册中断信号。
4. 任意顶层错误令进程以状态码 1 退出。

这是一个值得保留的边界：领域代码返回错误，只有 CLI 最外层决定进程退出码。

## 4. 工作目录解析

`ResolveCwd` 读取 `--cwd`；未给出时使用当前目录。它会规范路径并切换/传递给后续配置发现。所有项目级配置、技能、数据库默认位置都以此为锚点，因此“工作目录”不是普通显示字段，而是整个 Workspace 的身份输入。

## 5. 两种运行架构

`useClientServer()` 解析环境变量 `CRUSH_CLIENT_SERVER`。`setupWorkspace()` 据此分流。

### 5.1 本地模式 `setupLocalWorkspace`

严格顺序如下：

1. 读取 `debug`、`yolo`、`channels`、`data-dir` 等 flag。
2. 解析 cwd。
3. `config.Init(cwd, dataDir, debug)`：加载、合并、解析并保存配置快照。
4. 将只在本次进程生效的 CLI 值写入 `store.Overrides()`，而不是污染磁盘配置。
5. 以 `0700` 创建数据目录；首次创建 `.gitignore`，内容为 `*`。
6. `projects.Register` 登记项目；失败仅警告。
7. `db.Connect` 打开 `crush.db` 并执行迁移。
8. 此时数据目录已知，才用 `<dataDir>/logs/crush.log` 初始化日志。
9. 按配置发现 skills，构造每工作区 `skills.Manager`；本地模式使用 global mirror 兼容 TUI 读取包级状态。
10. `app.New(ctx, conn, store, skillsMgr)` 装配应用。
11. 配置允许时初始化遥测。
12. 用 `workspace.NewAppWorkspace` 包装应用，返回统一接口。
13. cleanup 负责 App 关闭与数据库引用释放。

任何一步失败都必须释放此前已获得的资源。尤其 `app.New` 失败时立即关闭/释放 DB，避免 WAL 与文件句柄遗留。

### 5.2 Client/Server 模式

该模式由 CLI 客户端通过 host 连接长期运行的服务端；服务端为每个工作目录创建 Workspace。客户端拿到 `proto.Workspace` 与 `client.Client`，再包装成 `ClientWorkspace`。

它带来三个额外问题：

- 工作区初始化是远程操作，需要等待 agent/MCP ready。
- 所有流式状态通过 SSE/协议事件传播，不能共享 Go 对象。
- server 可为数据目录启用进程锁，防止两个 server 同时写同一个 SQLite。

`internal/cmd/server.go` 处理服务进程探测、启动和连接；平台差异位于 `server_windows.go` 与 `server_other.go`。复刻时先完成本地模式，再把 Workspace 接口抽象成 RPC，难度会低很多。

## 6. `app.New`：依赖注入中心

关键文件：`internal/app/app.go`。

### 6.1 `App` 的主要字段

| 字段 | 责任 |
|---|---|
| `Sessions` | 会话 CRUD、用量与 TODO |
| `Messages` | 消息与流式写入 |
| `History` | 修改前文件版本历史 |
| `Permissions` | 工具授权请求 |
| `Questions` | agent 向用户提问 |
| `FileTracker` | 记录本会话读过的文件 |
| `AgentCoordinator` | 管理 coder/task agent |
| `LSPManager` | LSP 生命周期与诊断 |
| `Skills` | 工作区技能集合 |
| `events` | 面向 TUI/API 的聚合事件流 |
| `agentNotifications` | agent 专用通知流 |
| `runCompletions` | 每个顶层 run 的可靠终止信号 |
| `globalCtx` | 整个 App 的生命周期 context |
| `cleanupFuncs` | 逆向执行的清理动作 |

### 6.2 构造顺序

1. `db.New(conn)` 得到 sqlc `Queries`。
2. 构造 session、message、history、permission、question、filetracker 服务。
3. 构造 LSP Manager、事件 Broker、等待组。
4. `setupEvents()` 把各服务的事件桥接进 App 总事件流。
5. 尝试初始化剪贴板；失败不致命。
6. 后台检查版本更新。
7. 先 `mcp.ArmInit()` 再异步 `mcp.Initialize()`。Arm 必须同步发生，否则调用者可能在 goroutine 尚未登记初始化状态时错误地认为 MCP 已完成。
8. 初始化 herdr，并桥接权限、消息、run completion。
9. 注册 `db.Release(dataDir)` 和 `mcp.Close` 清理函数。
10. 若配置尚不完整，返回只有基础服务的 App，允许 TUI 引导配置。
11. 若已配置，初始化 coder agent。
12. 为 LSP 注册状态/诊断回调，再异步追踪已配置 LSP。

### 6.3 为什么 `runCompletions` 单独存在

流式消息的 Finish part 并不总能表示整个顶层 run 已可靠结束：消息写入可能仍在 debounce 缓冲，错误也可能发生在消息创建之前。因此 coordinator 在消息全部刷盘后发布一次 `RunComplete`，C/S 下的 `crush run` 以 `runID + sessionID` 精确匹配它后退出。

## 7. 交互模式

根命令无子命令时执行：

1. `setupWorkspaceWithProgressBar` 初始化 Workspace。
2. 解析 `--session` 的短 ID/完整 ID，并拒绝子会话。
3. 创建 `common.DefaultCommon(ws)`，再创建顶层 TUI model。
4. `tea.NewProgram` 注入环境、context 与输入 filter。
5. 后台 `ws.Subscribe(program)` 将 Workspace 事件送入 Bubble Tea。
6. `program.Run()` 阻塞至用户退出；崩溃被记录并转成友好错误。

进度条仅在 stderr 是终端且终端实现已知支持时启用，以 ANSI 设置/复位包围初始化。

## 8. 非交互命令 `crush run`

关键文件：`internal/cmd/run.go`。

### 8.1 输入与参数

- prompt 可来自位置参数，也可从 stdin 前置拼接。
- `--model`、`--small-model` 可临时覆盖模型。
- `--quiet` 隐藏 spinner，`--verbose` 显示日志。
- 无 prompt 直接报错。
- SIGINT/SIGTERM 取消 context。

### 8.2 本地运行

初始化本地 Workspace，检查 `Config.IsConfigured()`，必要时解析会话短 ID，最后调用 `App.RunNonInteractive`。

`RunNonInteractive` 的关键顺序是：

1. 重新建立不含交互专用工具的 coder agent。
2. 应用临时 large/small model 覆盖。
3. 必要时启动 spinner。
4. 等待 MCP 初始化完成并更新 agent tools/models。
5. 指定会话、最近会话或新建会话。
6. 对该会话自动批准权限；非交互模式无法弹窗。
7. goroutine 中运行 coordinator；主 goroutine消费结果和 context。
8. 停止 spinner、输出最终文本、确保终端状态恢复。

### 8.3 远程运行

客户端先订阅事件，再生成唯一 `runID` 调用 `SendMessage`。`runStream` 维护每条消息已输出到 stdout 的字符位置，最后以匹配 runID 的 `RunComplete` 为权威结束点，并做一次最终文本对账，避免 SSE 中间事件丢失造成截断。

## 9. 其他 CLI 命令导航

| 文件 | 命令及用途 |
|---|---|
| `login.go` | Hyper/Copilot 设备授权 |
| `logout.go` | 删除全局 OAuth/API key 字段；可交互选择平台 |
| `models.go` | 枚举已合并的 Provider/Model，并选择模型 |
| `session.go` | 会话列表、操作入口 |
| `projects.go` | 已登记项目列表与清理 |
| `dirs.go` | 输出配置、数据等目录，便于脚本使用 |
| `logs.go` | 查看/跟随日志 |
| `schema.go` | 输出配置 JSON Schema |
| `update_providers.go` | 更新 Catwalk Provider 元数据缓存 |
| `stats.go` | 聚合多个项目数据库并生成统计展示 |

`internal/cmd/stats/AGENTS.md` 要求其 HTML/CSS/JS 修改后使用 Prettier；OAuth callback 子目录也有相同规则，并要求 HTML 标签中的 Go template action 保持单行。

## 10. 错误与并发边界

- 配置、DB、App 构造错误是启动致命错误，立即返回。
- 项目登记、剪贴板、更新检查等辅助能力只记警告。
- 所有长任务使用 context；顶层 CLI 负责信号取消。
- 初始化 goroutine 前要先建立可等待状态，例如 MCP 的 Arm/Initialize。
- App 事件是进程内 fan-out；C/S 模式必须序列化为带类型标签的协议 Payload。
- cleanup 必须幂等，且通常按依赖创建的逆序执行。

## 11. 从零复刻建议

1. 先写 `main -> Execute -> root command`，只实现 `--cwd`。
2. 定义 `Workspace` 最小接口：Config、Sessions、Messages、Subscribe、Close。
3. 实现本地 `setupWorkspace`：配置、SQLite、App 三步。
4. App 先只装配 Session 和 Message，验证新建会话与消息。
5. 加 PubSub，把 service 事件汇总给一个简易终端循环。
6. 加权限、问题、历史、文件读取跟踪。
7. 加 agent、MCP、LSP，并明确每个后台任务的 context 与 cleanup。
8. 最后实现 ClientWorkspace，把相同接口映射成 HTTP/SSE。
9. 增加 `runID` 和权威完成事件，测试同一 session 排队多个 prompt。
10. 测试每个构造阶段失败时是否泄漏 DB、socket、goroutine。

## 12. 推荐阅读顺序

```text
main.go
internal/cmd/root.go
internal/cmd/run.go
internal/app/app.go
internal/app/provider.go
internal/cmd/server.go
internal/workspace/*
internal/client/* + internal/server/* + internal/proto/*
其余 cmd 子命令
```

阅读测试时重点看 `internal/app/app_test.go`、`provider_test.go`、`resolve_session_test.go`、`internal/cmd/run_stream_test.go` 与 `clientserverrace/race_test.go`：它们揭示了源码注释之外的时序契约。
