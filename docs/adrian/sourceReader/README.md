# Crush 源码阅读文档库

> 状态：待复核生成稿
> 生成日期：2026-08-16
> 基准提交：`16dce459cafecee92eae0ed7c47a0c641c8bbb9f`
> 工作区：dirty（仅本目录文档）
> 源码范围：`main.go`、`internal/`、根目录配置与构建资产
> 生成方式：源码、测试、配置、Schema 与部署资产静态分析

## 快速摘要

### 架构总览（模块与依赖）

Crush 是 Charm 用 Go 编写的终端 AI 编程助手。默认在一个进程里装配
`app.App`，由 TUI 或 `crush run` 通过 `workspace.Workspace` 驱动
`agent.Coordinator` / `sessionAgent`；可选 `CRUSH_CLIENT_SERVER=1` 时，
前端只持有 `ClientWorkspace`，真实 App 由独立 Server 上的
`backend.Backend` 托管。SQLite、Broker、Permission、Hook、Shell、LSP
和 MCP 提供状态与外部能力。

本目录按四层推进，不可跳级：先看骨架，再跟一条完整成功路径，再拆主链路
分支，最后才按模块补齐全部逻辑。

### 核心调用序列（逐步逻辑）

1. `main.go:main` 调用 `internal/cmd/root.go:Execute`，Fang/Cobra 解析命令。
2. 交互模式走 `rootCmd.RunE`，非交互模式走 `runCmd.RunE`；二者都先
   `setupWorkspace` 选择本地 `AppWorkspace` 或远程 `ClientWorkspace`。
3. 提示经 `Workspace.AgentRun` 或 `App.RunNonInteractive` 进入
   `coordinator.run`，再进入 `sessionAgent.Run`。
4. `fantasy.Agent.Stream` 产出文本/工具调用；工具经 `hookedTool`、
   Permission 与具体实现，结果写回消息并继续模型循环。
5. `message.Service.FlushAll` 后发布 `notify.RunComplete`；TUI 订阅
   Broker，本地 `crush run` 在 `Coordinator.Run` 返回时退出。

### 易错点与边界条件

- 必须按层阅读：第一、二层不展开每个函数分支。
- 交互 Agent 不等待慢 MCP；非交互 `crush run` 会 `mcp.WaitForInit`。
- 本地 `crush run` 以 `Coordinator.Run` 返回为退出条件；Client/Server 的
  `crush run` 以匹配 `RunID` 的 `RunComplete` 为退出条件。
- 本地 `AppWorkspace.AgentRun` 是**同步** `Coordinator.Run`；TUI 的异步来自
  `tea.Cmd`。远程 `SendMessage` 才是 fire-and-forget。
- `crushrc` 是首选配置，`crush.json` 仍兼容但已弃用。
- 本轮未运行完整 Go 测试；不声明线上 Provider/OAuth/MCP/LSP 已验证。

## 1. 文档库定位

输出目录由用户指定为 `docs/adrian/sourceReader/`。阅读顺序固定为：

入口 → 相关测试 → 代表性功能（一次用户提示形成完整 Agent turn）→ 扩展覆盖。

代表性功能选择理由：它穿过 CLI/TUI 入口、配置、Session、模型循环、工具、
权限、SQLite、事件和 stdout/TUI，是 Crush 的核心价值路径。

旧版按模块平铺、英文文件名的文档已替换为本结构（基准从 `5712d483` 更新为
当前 HEAD）。

## 2. 目录树与阅读顺序

```text
docs/adrian/sourceReader/
├── README.md                              # 文档地图；每篇标明所属层
├── 01-简单框架-系统骨架.md                 # 第一层
├── 02-简单例子-全路径走读.md               # 第二层
├── 03-详细逐步说明-主链路拆解.md           # 第三层
├── 04-运行入口与生命周期.md                # 第四层
├── 05-配置体系与Provider认证.md
├── 06-Agent运行时.md
├── 07-工具权限与Hook.md
├── 08-Skills提示词与上下文.md
├── 09-数据模型持久化与事件.md
├── 10-Workspace与ClientServer.md
├── 11-Shell-LSP-MCP.md
├── 12-并发可靠性安全与可观测性.md
├── 13-TUI架构与渲染.md
├── 14-构建部署发布与运维.md
├── 15-测试体系与质量保障.md
├── 16-从零重新实现指南.md
└── 17-源码索引与覆盖矩阵.md
```

必须按编号顺读。未读完 01 和 02 之前，不要跳进 04 以后的模块细节。

## 3. 内容大纲、所属层与状态

| 文档 | 层 | 大纲 | 状态 |
|---|---|---|---|
| [01-简单框架-系统骨架.md](01-简单框架-系统骨架.md) | 一 | 进程形态、模块边界、主路径骨架图 | 已写 |
| [02-简单例子-全路径走读.md](02-简单例子-全路径走读.md) | 二 | 本地 `crush run` 从入口到 stdout 的成功路径 | 已写 |
| [03-详细逐步说明-主链路拆解.md](03-详细逐步说明-主链路拆解.md) | 三 | dispatch、Stream、工具、落库、取消/排队、两种退出契约 | 已写 |
| [04-运行入口与生命周期.md](04-运行入口与生命周期.md) | 四 | `main`、Cobra 子命令、App 装配、Shutdown | 已写 |
| [05-配置体系与Provider认证.md](05-配置体系与Provider认证.md) | 四 | crushrc/json、Provider、模型、OAuth、热更新 | 已写 |
| [06-Agent运行时.md](06-Agent运行时.md) | 四 | Coordinator、SessionAgent、队列、取消、摘要、RunComplete | 已写 |
| [07-工具权限与Hook.md](07-工具权限与Hook.md) | 四 | 工具注册、Permission、hookedTool、全部内置工具 | 已写 |
| [08-Skills提示词与上下文.md](08-Skills提示词与上下文.md) | 四 | Prompt 模板、上下文文件、Skill 发现与隔离 | 已写 |
| [09-数据模型持久化与事件.md](09-数据模型持久化与事件.md) | 四 | SQLite、sqlc、Session/Message、debounce、Broker | 已写 |
| [10-Workspace与ClientServer.md](10-Workspace与ClientServer.md) | 四 | Workspace 接口、Backend、HTTP/SSE、重连 | 已写 |
| [11-Shell-LSP-MCP.md](11-Shell-LSP-MCP.md) | 四 | 嵌入式 Shell、LSP 管理、MCP 生命周期 | 已写 |
| [12-并发可靠性安全与可观测性.md](12-并发可靠性安全与可观测性.md) | 四 | Context、锁、投递等级、日志/指标、安全边界 | 已写 |
| [13-TUI架构与渲染.md](13-TUI架构与渲染.md) | 四 | 唯一 Bubble Tea Model、Chat、Dialog、样式、性能 | 已写 |
| [14-构建部署发布与运维.md](14-构建部署发布与运维.md) | 四 | Taskfile、sqlc、Goreleaser、跨平台 | 已写 |
| [15-测试体系与质量保障.md](15-测试体系与质量保障.md) | 四 | 测试布局、替身、关键断言、覆盖缺口 | 已写 |
| [16-从零重新实现指南.md](16-从零重新实现指南.md) | 四 | 分阶段闭环与客观验收 | 已写 |
| [17-源码索引与覆盖矩阵.md](17-源码索引与覆盖矩阵.md) | 四 | 包/文件索引、生产清单、待确认项 | 已写 |

## 4. 源码覆盖矩阵

覆盖口径是“生产目录或行为资产至少映射到一篇文档”，不是逐行解释生成代码。
约 382 个生产 Go 文件、212 个测试文件。

| 源码/资产范围 | 主文档 | 层 | 状态 |
|---|---|---|---|
| `main.go`、`internal/cmd/` | [04](04-运行入口与生命周期.md) | 四 | 已覆盖 |
| `internal/app/` | [04](04-运行入口与生命周期.md) | 四 | 已覆盖 |
| `internal/config/`、`shellconfig/`、`discover/`、`oauth/` | [05](05-配置体系与Provider认证.md) | 四 | 已覆盖 |
| `internal/agent/`（不含 tools） | [06](06-Agent运行时.md) | 四 | 已覆盖 |
| `agent/tools/`、`permission/`、`hooks/` | [07](07-工具权限与Hook.md) | 四 | 已覆盖 |
| `skills/`、`commands/`、Agent templates | [08](08-Skills提示词与上下文.md) | 四 | 已覆盖 |
| `session/`、`message/`、`db/`、`history/`、`filetracker/` | [09](09-数据模型持久化与事件.md) | 四 | 已覆盖 |
| `workspace/`、`backend/`、`server/`、`client/`、`proto/` | [10](10-Workspace与ClientServer.md) | 四 | 已覆盖 |
| `shell/`、`lsp/`、`agent/tools/mcp/` | [11](11-Shell-LSP-MCP.md) | 四 | 已覆盖 |
| `pubsub/`、`event/`、`lock/`、`csync/` | [12](12-并发可靠性安全与可观测性.md) | 四 | 已覆盖 |
| `internal/ui/` | [13](13-TUI架构与渲染.md) | 四 | 已覆盖 |
| Taskfile、sqlc、Goreleaser、CI、migrations | [14](14-构建部署发布与运维.md)、[09](09-数据模型持久化与事件.md) | 四 | 已覆盖 |
| `*_test.go`、catwalk golden | [15](15-测试体系与质量保障.md) | 四 | 已覆盖（静态） |
| 全仓库生产 Go 与非 Go 行为资产 | [17](17-源码索引与覆盖矩阵.md) | 四 | 已盘点 |

支撑小包（`fsext`、`diff`、`clipboard`、`env`、`home`、`format`、`dns`、
`update`、`version`、`projects`、`swagger`）见 [17](17-源码索引与覆盖矩阵.md)。

## 5. 代表性功能证据链

完整走读见 [02](02-简单例子-全路径走读.md)；分支与失败见 [03](03-详细逐步说明-主链路拆解.md)。

| 调用方文件与符号 | 关系 | 被调用方文件与符号 | 触发与输入 | 返回与后续处理 | 错误、状态与副作用 |
|---|---|---|---|---|---|
| `main.go:main` | 调用 | `cmd.Execute` | 进程启动；可选 `CRUSH_PROFILE` | Fang 解析 Cobra | 命令失败 `os.Exit(1)` |
| `cmd/run.go:runCmd.RunE` | 调用 | `setupLocalWorkspace` | 默认无 `CRUSH_CLIENT_SERVER` | 得到 `AppWorkspace` | 配置/DB 失败直接返回 |
| `app.App.RunNonInteractive` | 调用 | `agent.Coordinator.Run` | Session、prompt | 后台 goroutine 跑完 turn | 自动批准该 Session 权限 |
| `coordinator.run` | 调用 | `sessionAgent.Run` | 模型参数、工具、OnComplete | 合并终态后 `PublishMustDeliver` | 非交互会先 `mcp.WaitForInit` |
| `sessionAgent.Run` | 编排 | `fantasy.Agent.Stream` | 历史消息与 call | 消费流式 part，写 `message.Service` | 排队/取消/摘要；defer `FlushAll` |
| `message.service.Update` | 持久化/发布 | sqlc Query 与 Broker | 流式 Message | 文本 debounce；结构/终态同步 flush | 订阅者收到 `UpdatedEvent` |

## 6. 事实与验证边界

- 事实来自基准提交的实现、测试、配置、迁移或生成 Schema；设计解释标注为推断。
- 本轮做静态盘点与符号核对；未运行完整 `go test ./...`，因此不声明测试通过。
- 外部 Provider、OAuth、MCP、真实 LSP 和生产容量/SLO 未在线验证，均为待确认。
- 与旧文档冲突时以当前源码为准。已确认的一处：本地 `AgentRun` 同步，不是
  自行起 goroutine。

## 7. 文档维护规则

新增工具时同步核对 Agent 注册、Permission、Hook、工具描述和 TUI renderer；
新增配置项时同步核对 `Config`、`crushrc` builtin、JSON 兼容、默认值、Schema
和 UI；新增远程能力时同步核对 `proto`、server、client、Workspace、SSE 和重连
缓存。修改后先从入口正向追踪，再从核心函数反查调用方。
