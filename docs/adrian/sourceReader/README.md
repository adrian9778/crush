# Crush 最新源码重实现手册

> 状态：待复核生成稿
> 生成日期：2026-08-14
> 基准提交：`5712d4839a6a10e9940804d511bb322dbe73a511`
> 工作区：clean（开始分析时）
> 源码范围：`main.go`、`internal/`、根目录配置与构建资产
> 生成方式：源码、测试、配置、Schema、既有三套源码文档与部署资产静态分析

## 快速摘要

### 架构总览（模块与依赖）

Crush 是一个以 Workspace 为资源边界、以 Session 为对话边界的终端 AI
编程助手。`internal/cmd` 选择本地或 Client/Server 运行形态；本地形态由
`internal/app.App` 装配单工作区服务，远程形态由 `internal/backend.Backend`
托管多个 App；`internal/workspace.Workspace` 让 TUI 使用同一套接口；
`internal/agent.Coordinator` 选择模型与工具，`sessionAgent` 执行模型循环；
SQLite、Broker、Permission、Hook、Shell、LSP 和 MCP 提供状态与外部能力。

### 核心调用序列（逐步逻辑）

1. `main.go:main` 调用 `internal/cmd/root.go:Execute`，Cobra 解析命令与参数。
2. `rootCmd.RunE` 创建本地 `AppWorkspace` 或远程 `ClientWorkspace`，再启动
   `internal/ui/model.UI`。
3. 用户提示经 `workspace.Workspace.AgentRun` 到
   `agent.Coordinator.Run/RunAccepted`，刷新模型、组装 Provider 参数和工具。
4. `sessionAgent.Run` 串行化同 Session dispatch，创建消息并调用
   `fantasy.Agent.Stream`，把流式 part 写入 `message.Service`。
5. 工具调用依次经过 `hookedTool`、Permission 与具体工具，结果回到模型循环。
6. `message.Service.Flush` 保证最终消息落库；`RunComplete` 携带 RunID 经
   Broker 可靠发布，本地 TUI 或远程 SSE 消费最终状态。

### 易错点与边界条件

- 交互 Agent 不等待慢 MCP 初始化；非交互运行才等待工具集合稳定。
- accepted run 用序列高水位处理取消竞态；空闲 cancel 不能污染下一次提示。
- 流式文本会 debounce；终止 part、工具结构变化、推理结束和强一致读取必须
  同步 flush。
- Broker 普通事件允许慢订阅者丢包；关键终态和交互请求使用 must-deliver。
- TUI 只有一个 Bubble Tea Model；昂贵 IO 必须返回 `tea.Cmd`。
- `crushrc` 是首选配置，`crush.json` 仍兼容；sqlc/Swagger 生成文件不是手工
  修改入口。

## 1. 文档库定位与整合原则

本目录是 `docs/adrian/sourceReader1`、`sourceReader2`、`sourceReader3` 的统一
整理版。整合时以当前提交的生产代码和测试为第一事实源：

- Reader1 的短篇导航用于校准新手阅读路径和修改定位表；
- Reader2 的 Agent、配置、Backend、LSP/MCP 专题用于补充设计动机；
- Reader3 的调用链、并发语义、源码清单和重实现路线作为主体；
- 文档与源码冲突时采用当前源码，并在
  [17-整合差异与待确认项.md](17-整合差异与待确认项.md) 记录口径。

## 2. 目录树与阅读顺序

```text
docs/adrian/sourceReader/
├── README.md
├── 01-system-overview.md
├── 02-bootstrap-cli.md
├── 03-config-providers-auth.md
├── 04-agent-runtime.md
├── 05-tools-permissions-hooks.md
├── 06-skills-prompts-context.md
├── 07-data-model-persistence.md
├── 08-workspace-client-server.md
├── 09-shell-hooks-lsp-mcp.md
├── 10-concurrency-lifecycle-reliability.md
├── 11-tui-architecture.md
├── 12-tui-chat-rendering.md
├── 13-tui-dialog-styles-performance.md
├── 14-package-file-index.md
├── 15-reimplementation-roadmap.md
├── 16-source-manifest.md
└── 17-整合差异与待确认项.md
```

建议新手按编号顺读。聚焦 Agent 时读 04、05、07、10；聚焦配置与扩展时读
03、06、09；聚焦 TUI 时读 11、12、13；从零复刻时以 15 为主线，用 14、16
核对覆盖。

## 3. 内容大纲与状态

| 文档 | 主要内容 | 状态 |
|---|---|---|
| [01](01-system-overview.md) | 系统上下文、分层、资源边界、端到端数据流 | 已整合 |
| [02](02-bootstrap-cli.md) | `main`、Cobra、Workspace 初始化、App 装配、CLI | 已整合 |
| [03](03-config-providers-auth.md) | 配置发现/合并、Provider、模型、认证、热更新 | 已整合 |
| [04](04-agent-runtime.md) | Coordinator、SessionAgent、Run、队列、取消、摘要 | 已复核核心链路 |
| [05](05-tools-permissions-hooks.md) | 工具注册、安全边界、Permission 与 Hook 顺序 | 已整合 |
| [06](06-skills-prompts-context.md) | Prompt、上下文文件、Skill 发现与工作区隔离 | 已整合 |
| [07](07-data-model-persistence.md) | SQLite、Session、Message、debounce、事件 | 已复核关键测试 |
| [08](08-workspace-client-server.md) | Workspace 接口、Backend、HTTP/SSE、重连 | 已整合 |
| [09](09-shell-hooks-lsp-mcp.md) | 嵌入式 Shell、Hook、LSP、MCP 生命周期 | 已整合 |
| [10](10-concurrency-lifecycle-reliability.md) | Context、锁、投递等级、取消与关闭不变量 | 已整合 |
| [11](11-tui-architecture.md) | 顶层 UI 状态机、消息路由、布局与绘制 | 已整合 |
| [12](12-tui-chat-rendering.md) | List、Chat、MessageItem、工具与 Markdown 渲染 | 已整合 |
| [13](13-tui-dialog-styles-performance.md) | Dialog、样式 token、终端宽度与性能 | 已整合 |
| [14](14-package-file-index.md) | 包职责、关键文件与测试反查入口 | 已整合 |
| [15](15-reimplementation-roadmap.md) | 分阶段闭环与客观验收 | 已整合 |
| [16](16-source-manifest.md) | 生产源码与非 Go 行为源清单 | 已核对基准提交 |
| [17](17-整合差异与待确认项.md) | 三套旧文档取舍、冲突与未验证范围 | 已新增 |

## 4. 源码覆盖矩阵

覆盖口径是“生产目录或行为资产至少映射到专题和索引”，不是逐行解释生成代码。

| 源码/资产范围 | 主文档 | 补充文档 | 状态 |
|---|---|---|---|
| `main.go`、`internal/cmd/` | [02](02-bootstrap-cli.md) | [01](01-system-overview.md) | 已覆盖 |
| `internal/app/` | [02](02-bootstrap-cli.md) | [07](07-data-model-persistence.md)、[10](10-concurrency-lifecycle-reliability.md) | 已覆盖 |
| `internal/agent/`、`agent/prompt/` | [04](04-agent-runtime.md) | [06](06-skills-prompts-context.md) | 已覆盖 |
| `agent/tools/`、`permission/`、`hooks/` | [05](05-tools-permissions-hooks.md) | [09](09-shell-hooks-lsp-mcp.md) | 已覆盖 |
| `config/`、`shellconfig/`、`discover/`、`oauth/` | [03](03-config-providers-auth.md) | [09](09-shell-hooks-lsp-mcp.md) | 已覆盖 |
| `session/`、`message/`、`db/`、`history/`、`filetracker/` | [07](07-data-model-persistence.md) | [10](10-concurrency-lifecycle-reliability.md) | 已覆盖 |
| `workspace/`、`backend/`、`server/`、`client/`、`proto/` | [08](08-workspace-client-server.md) | [01](01-system-overview.md)、[10](10-concurrency-lifecycle-reliability.md) | 已覆盖 |
| `shell/`、`lsp/`、`agent/tools/mcp/` | [09](09-shell-hooks-lsp-mcp.md) | [05](05-tools-permissions-hooks.md) | 已覆盖 |
| `skills/`、`commands/`、Agent templates | [06](06-skills-prompts-context.md) | [03](03-config-providers-auth.md) | 已覆盖 |
| `internal/ui/` | [11](11-tui-architecture.md) | [12](12-tui-chat-rendering.md)、[13](13-tui-dialog-styles-performance.md) | 已覆盖 |
| 通用支撑包 | [14](14-package-file-index.md) | [16](16-source-manifest.md) | 索引级覆盖 |
| migrations、SQL、Schema、构建与发布资产 | [07](07-data-model-persistence.md) | [15](15-reimplementation-roadmap.md)、[16](16-source-manifest.md) | 已覆盖 |

## 5. 代表性功能证据链

选择“一次用户提示形成完整 Agent turn”作为代表性功能，因为它穿过入口、模型、
工具、权限、持久化、事件和 UI/远程协议。

| 调用方文件与符号 | 关系 | 被调用方文件与符号 | 触发与输入 | 返回与后续处理 | 错误、状态与副作用 |
|---|---|---|---|---|---|
| `internal/ui/model/*` | 调用接口 | `workspace.Workspace.AgentRun` | Session、prompt、attachments | 本地调用 App，远程发送请求 | UI 切换 busy；错误成为领域事件 |
| `workspace/app_workspace.go:AgentRun` | 委派 | `agent.Coordinator.Run` | 同上 | 返回 `fantasy.AgentResult` | 共享本地进程 context |
| `backend/backend.go:SendMessage` | 异步派发 | `agent.Coordinator.RunAccepted` | RunID、accepted handle | HTTP 接收后后台执行 | 失败时补发最终 completion |
| `agent/coordinator.go:coordinator.run` | 配置并调用 | `sessionAgent.Run` | 模型参数、工具、RunID | 合并最终 completion | OAuth 重试；must-deliver 终态 |
| `agent/agent.go:sessionAgent.Run` | 编排 | `fantasy.Agent.Stream` | 历史消息和 call | 消费流式 step/part | 排队、取消、摘要、落库 |
| `message/message.go:service.Update` | 持久化/发布 | sqlc Query 与 Broker | 流式 Message | 更新 DB 并通知订阅者 | 文本 debounce；结构/终态同步 flush |

## 6. 事实与验证边界

- 事实来自基准提交的实现、测试、配置、迁移或生成 Schema；设计解释标注为推断。
- 本轮做了静态盘点、符号核对、链接/格式/敏感信息检查；未运行完整 Go 测试，
  因此不声明测试通过。
- 外部 Provider、OAuth、MCP、真实 LSP 和生产容量/SLO 未在线验证，均为待确认。

## 7. 文档维护规则

新增工具时同步核对 Agent 注册、Permission、Hook、工具描述和 TUI renderer；
新增配置项时同步核对 `Config`、`crushrc` builtin、JSON 兼容、默认值、Schema
和 UI；新增远程能力时同步核对 `proto`、server、client、Workspace、SSE 和重连
缓存。修改后先从入口正向追踪，再从核心函数反查调用方。
