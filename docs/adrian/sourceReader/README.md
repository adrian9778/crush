# Crush 源码阅读手册

> 面向第一次接触 Go、终端 UI 或 AI Agent 工程的新读者。本文档以当前仓库源码为准，不是产品使用说明。

## 1. 这套文档解决什么问题

Crush 不是一个“调用一次模型 API 然后打印结果”的小程序。它同时包含 CLI、TUI、工作区生命周期、Agent 循环、工具系统、权限、配置语言、数据库、HTTP/SSE 服务、LSP、MCP、OAuth 和跨平台 Shell。直接从文件名逐个阅读，很容易看见局部却不知道它为什么存在。

建议按下面顺序阅读：

1. [01-architecture.md](01-architecture.md)：先建立系统地图，理解本地与 Client/Server 两种运行形态。
2. [02-bootstrap-and-config.md](02-bootstrap-and-config.md)：从 `main.go` 走到一个可用 Workspace。
3. [03-agent-and-tools.md](03-agent-and-tools.md)：理解一次提问如何变成模型流、工具调用和持久化消息。
4. [04-data-and-infrastructure.md](04-data-and-infrastructure.md)：理解 Session、Message、DB、事件、权限、Hooks、Shell、LSP、MCP。
5. [05-tui.md](05-tui.md)：理解 Bubble Tea 顶层状态机、渲染管线、聊天项和对话框。
6. [06-reading-and-extension-guide.md](06-reading-and-extension-guide.md)：按需求找修改点，并避开常见坑。
7. [07-package-index.md](07-package-index.md)：全部 `internal` 源码包的职责索引。

## 2. 一句话心智模型

Crush 是一个以 Workspace 为资源边界、以 Session 为对话边界、由 Coordinator 装配模型和工具、由 SessionAgent 执行对话循环、通过服务接口把同一套业务同时暴露给本地 TUI 和远程 Client/Server 的终端 AI 编程助手。

## 3. 阅读符号约定

- “服务”表示有明确接口和生命周期的业务对象，例如 `session.Service`。
- “Broker”表示进程内发布/订阅总线，不等同于外部消息队列。
- “Workspace”既是一个工作目录，也是一组绑定到该目录的配置、数据库、Agent、LSP、Skills 和事件资源。
- “消息”可能指持久化的 `message.Message`，也可能指 Bubble Tea 的 `tea.Msg`；正文会明确区分。
- `proto` 是跨进程数据契约，`message`/`session` 等是内部领域模型。

## 4. 源码规模与阅读策略

当前仓库约有 592 个 Go 文件、14 万余行 Go 代码。体量最大的手写实现包括 `internal/ui/model/ui.go`、`internal/agent/agent.go`、`internal/agent/coordinator.go`、`internal/config/load.go` 和 Client/Server Workspace 适配层。`internal/swagger/docs.go` 与 `internal/db/*.sql.go` 主要是生成代码，应理解生成来源和使用边界，无需像业务代码一样逐行背诵。

阅读时始终追踪四样东西：

1. 谁创建对象；
2. 谁持有对象；
3. 谁负责关闭对象；
4. 状态变化通过返回值、数据库还是事件传播。

## 5. 最重要的源码入口

| 入口 | 作用 |
| --- | --- |
| `main.go` | 进程入口，可选启动 pprof，随后进入 CLI |
| `internal/cmd/root.go` | 根命令、TUI 启动、本地/远程 Workspace 选择 |
| `internal/app/app.go` | 单 Workspace 内所有核心服务的装配中心 |
| `internal/agent/coordinator.go` | Provider、Model、Prompt、Tools 的协调与刷新 |
| `internal/agent/agent.go` | 单 Session 的模型循环、排队、取消、摘要、消息落库 |
| `internal/backend/backend.go` | 多 Workspace 服务端业务与生命周期 |
| `internal/workspace/*.go` | TUI 面向的统一 Workspace 接口及两种实现 |
| `internal/ui/model/ui.go` | 唯一顶层 Bubble Tea Model 和 UI 状态机 |

## 6. 文档维护规则

代码变更后优先更新调用链和不变量，不要只改文件列表。新增工具时同时检查 Agent 注册、权限、工具说明模板和 TUI renderer；新增配置项时同时检查 JSON、`crushrc` builtin、默认值、校验、Schema 与 UI；新增跨进程能力时同时检查 `proto`、server、client、workspace 适配层和事件传递。
