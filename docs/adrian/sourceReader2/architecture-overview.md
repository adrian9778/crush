# 一、项目整体架构

## 1.1 项目定位

Crush 是 Charm（终端 UI 框架 bubbletea 的开发商）构建的**终端 AI 编程助手**。它的本质是让 LLM（语言模型）获得在终端中的"手和眼"：

- **读**: 读取文件、搜索代码库
- **写**: 编辑文件、创建项目结构
- **执行**: bash 命令运行、管道处理
- **理解**: LSP 代码分析、跨仓库搜索

Crush 不是单纯的 CLI wrapper —— 它是一个**完整的 agent runtime**，包括 agent loop（对话循环）、工具调用链、权限系统、hooks 策略引擎、持久层等。

## 1.2 进程模型：两种运行模式

### 模式 A: Standalone（单进程）

```
                        Crush 单一进程
┌─────────────────────────────────────────────────┐
│                                                   │
│  CLI (cobra)                                    │
│       └── interactive TUI (bubbletea v2)        │
│          ├── AgentCoordinator                    │
│          ├── HTTP Server (internal, optional)    │
│          └── All subsystems                      │
│                                                   │
└─────────────────────────────────────────────────┘

启动: go run .
```

用户直接运行 `crush`，整个应用在一个进程中。这是默认的使用方式。

### 模式 B: Client/Server（分布式）

```
                    +------------------+
                    |    Server 进程     |
                    | (长期驻留)          |
                    |                   |
                    |  HTTP REST API   |  ◄─ Unix socket
                    |  /v1/workspaces  |  ◌ TCP port
                    |  SSE Event Stream|      or Named Pipe
                    +--------┬----------+
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
   +-------------+  +--------------+  +------------------+
   | VS Code     |  | Neovim       |  | curl / custom    |
   | 插件 (client)|  | 插件 (client) |  | HTTP clients     |
   +-------------+  +--------------+  +------------------+

启动 server: go run . server --host unix:///tmp/crush.sock
```

Server 进程始终运行，管理 workspace（每个目录 = one workspace）。Client 通过 HTTP API + SSE events 与之通信。

## 1.3 顶层模块关系图

```
                    main.go (cobra CLI)
                         │
      ┌─────────────┬───┼──────────────┐
      ▼             ▼     ▼            │
  cmd/run.go   server.go    login.go   │
      │             │             │    │
      ▼             ▼             │    │
AppWorkspace  Server       backend.Backend  ◄── C/S core
      │             │             │    │
      ▼             │             │    │
backend.New()  ◄───┘             │    │
      │                         All consume
      ▼                         ──────► App services:
  Workspace                  ┌───────────────────────┐
      │                        │ session.Service       │
      ▼                        │ message.Service       │
app.New() ─────────────────▶   │ permission.Service    │
      │                        │ agent.Coordinator     │
      ▼                        │ lsp.Manager           │
core App struct                │ skills.Manager        │
  ├── config.ConfigStore       │ pubsub.Broker         │
  ├── session / message        │ filetracker.Service   │
  ├── Coordinator              └───────────────────────┘
  ├── LSP Manager
  ├── MCP client
  ├── Permissions
  ├── pubsub.Broker[tea.Msg] (events)
  ├── Skills Discovery
  ├── FileTracker
```

## 1.4 关键包依赖拓扑（核心路径）

```
main.go ──▶ cmd/ (Cobra commands)
    │
    ├──► internal/app/app.go (App wiring)
    │       ├── agent.Coordinator   ──▶ sessionAgent (agent.go)
    │       │                          ├── fantasy (LLM provider abstraction)
    │       │                          ├── tools/*.go (25+ tools)
    │       │                          ├── loop_detection.go
    │       │                          └── prompt/ (system templates)
    │       │
    │       ├── lsp.Manager         ──▶ Client (new connections via powernap)
    │       │
    │       ├── MCP client          ──▶ init lifecycle + tools registration
    │       │
    │       ├── session.Service     ──▶ db/sessions.sql.go (sqlc generated)
    │       ├── message.Service     ──▶ db/messages.sql.go (sqlc generated)
    │       ├── permission.Service  ──▶ policy + runtime approval
    │       ├── skills.Manager      ──▶ discovery from config paths
    │       └── filetracker.Service ──▶ db/files.sql.go
    │
    ├──► internal/server/ (HTTP REST API)
    │       ├── controllerV1 (HTTP handlers → backend methods)
    │       └── SSE EventStreams for event push
    │
    └──► internal/backend/backend.go (transport-agnostic ops)
            ├── Backend + Workspace (lifecycle management)
            ├── Client connection tracking
            ├── Grace periods + auto-shutdown
            └── Path deduplication

内部基础设施:
    ├── pubsub/broker.go  → async event dispatch
    ├── csync/           → concurrent-safe collections
    ├── hooks/           → hook execution engine (PreToolUse)
    ├── config/load.go   → multi-source configuration merge pipeline
    └── schema.json      → JSON Schema for crush.json validation
```

## 1.5 配置加载管线

```
                          用户的工作目录 / ~/.crush/
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
     crushrc (Bash script)   crush.json      config overrides
         (shell DSL)       (JSON format)     (CLI flags/env vars)
              │               │               │
              ▼               ▼               ▼
    shellconfig.LoadShellConfig()  config.Parse JSON   Merge logic:
      ┌──────────────────────┐                    1. Base (crushrc)
      │ shell.Run(source)    │                    2. + crush.json deep-merge
      │                      │                    3. + config overrides
      │ ConfigBuilder mutates│                    4. + CLI/env last-wins
      │ via builtins:        │
      │   provider()          │
      │   model()             │
      │   mcp(), lsp(),       │
      │   permissions(),      │
      │   hook(), options()   │
      └──────────┬────────────┘
                 ▼
     builder.JSON() → JSON bytes (统一中间格式)
                 │
                 ▼
     config.MergeConfigStore(
       shellconfigJSON,
       rawJSONFile,
         overrides
     )
                 │
                 ▼
     config.ConfigStore  (最终配置，不可变读取 + atomic写入)
```

## 1.6 数据流：用户输入到 Agent 执行

```
┌──────────┐    Enter key    ┌─────────────┐     pubsub    ┌──────────────────┐
│ Textarea │              ──▶ │ UI.Update() │◄──────────────│ SSE Events Broker│
└──────────┘                 │             │                │                  │
                              │ sendMessage │                └──────────────────┘
                              │     │       │                      ▲
                              ▼     │       │                      │
                     ┌─────────┐    │       │    agent notifications
                     │ App     │──▶ │  pubsub│◀────── agent.Notifications()
                     │.SendEvent│                └──────────────────┘
                     └────┬────┘
                          ▼
              ═══════ AgentCoordinator.Run ═══════
                  │         │           │
                  ▼         ▼           ▼
            Session  Fantasy     Tools
              ID     LLM API    Execution
             session   │        (bash/
                      ▼       edit/view)
                agent.      │
               Run(·)      │
                       ┌──┴─────────┐
                       ▼  Result ◀──┘
                  ═══════ Persist to SQLite ═══════


              UI rendering updates via pubsub.Broker[tea.Msg]:
                  ├── UpdateAvailableMsg (版本更新通知)
                  ├── runComplete (一轮 agent turn 完成的信号)
                  └── tool output / text delta events
```

## 1.7 EventStream 事件体系

SSE (Server-Sent Events) 是 Client/Server 模式下的实时通信通道：

| Event Type | Payload Description | Producer |
|-----------|--------------------|----------|
| session_created | New session created | server/controller |
| session_title_updated | Session renamed by title gen | agent.titleGenerator |
| message_updated | Streaming text_delta (partial AI response) | pubsub broker |
| message_complete | Full message persisted | session/message service |
| workspace_event | Workspace lifecycle events | backend.Workspace |
| lsp_state_update | LSP server state changes | lsp manager callback |

## 1.8 Client-Server 通信协议栈

```
┌───────────────────────┐
│        HTTP REST       │  ◦ CRUD API (CRUD-style)
│     +    SSE streams   │  ◦ Realtime events push
│                         │
├───────────────────────┤
│ transport              │  ◦ Unix domain socket (macOS/Linux)
│                       │  ◦ TCP/IP (localhost or remote server)
│                       │  ◦ Windows named pipe /var/run/
├───────────────────────|
│ HTTP framework          │   standard Go net/http + http-swagger/OpenAPI
│                         │
├───────────────────────|
│ Server layer            │   internal/server/controllerV1
│                         │   ◦ ~50 REST endpoints (see full listing in later section)
├───────────────────────┤
│ Business logic layer    │   internal/backend.Backend + Workspace
│                       │   ◦ workspace mgmt, session ops, agent dispatch
└───────────────────────┘
```

## 1.9 Go Module 与核心依赖

```go
// go.mod: key dependencies (non-exhaustive)
module github.com/charmbracelet/crush

require (
    charm.land/bubbletea/v2      // TUI framework (Bubble Tea v2 + UV screen buffer)
    charm.land/catwalk/pkg       // Model testing (snapshot/golden tests)
    charm.land/fantasy           // LLM provider abstraction (multi-vendor protocol support)
    charm.land/lipgloss/v2        // Terminal styling
    charm.land/glamour/v2         // Markdown rendering in terminal

    github.com/chroma-lang/chroma // Syntax highlighting (chroma lexer)
    github.com/sourcegraph/jsonrpc2 // LSP protocol layer
    github.com/swaggo/http-swagger // Swagger/OpenAPI integration
    golang.org/x/sync             // errgroup, semaphores
    modernc.org/sqlite            // SQLite C bindings (pure Go implementation)

    sqlc                          // SQL-to-Go code generator
)
```

## 1.10 代码组织原则

| 原则 | 实现方式 |
|------|----------|
| 单一职责 | 每个包一个主题（session/message/lsp/hooks/permission） |
| Service pattern | Config/Session/Message/FileTracker/Permissions all use `Service` interface |
| Event-driven pub/sub | `pubsub.Broker[M]` 解耦模块通信 |
| Transport-agnostic core | `backend.Backend` 完全脱离 HTTP/WS，只暴露业务方法 |
| sqlc codegen | SQL queries in `sql/*.sql`, Go structs in `*.sql.go` |
| Atomic ops | `atomicwrite.*` for race-free file writes (crucial config updates) |
| Thread safety | Custom `csync.*` collections (Map, Slice, Value) — no sync.Map overhead |
