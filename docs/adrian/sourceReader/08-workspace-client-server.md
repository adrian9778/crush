# 08. Workspace 与 Client/Server 架构

> 状态：待复核生成稿｜生成日期：2026-08-14
> 基准提交：`5712d4839a6a10e9940804d511bb322dbe73a511`｜工作区：clean（开始分析时）
> 源码范围：`internal/workspace/`、`backend/`、`server/`、`client/`、`proto/`
> 生成方式：源码、协议、生命周期与重连测试静态分析

## 快速摘要

### 架构总览（模块与依赖）
Workspace 是 TUI 防腐层；AppWorkspace 直接委派 App，ClientWorkspace 通过 Client/Server/Proto 到 Backend，Backend 管理多工作区生命周期。

### 核心调用序列（逐步逻辑）
1. Client 创建/claim Workspace。2. Backend 规范化路径并创建 App。3. HTTP 接受带 RunID 的消息。4. 后台 RunAccepted 执行。5. SSE 回送领域事件并更新客户端缓存。

### 易错点与边界条件
请求 context 不能错误取消后台初始化；多个 grace period 含义不同；重连必须假设事件可能永久丢失并以重新拉取状态恢复。

## 1. `workspace.Workspace`：前端的防腐层

`internal/workspace/workspace.go` 定义一个较大的接口，按 Sessions、Messages、Agent、Permissions、Questions、FileTracker、History、LSP、Config、Project、Skills、MCP、Events 分组。

接口大并不是为了抽象任意实现，而是精确表达 TUI/CLI 对一个“可运行工作区”的全部需求。两个实现：

- `AppWorkspace`：薄适配器，直接调用 `app.App` 内 Service；
- `ClientWorkspace`：远程代理，调用 `client.Client`，维护缓存、SSE 和重连状态。

重写时应先让本地实现通过该接口跑通，再添加远程实现。不要让 UI 针对两种模式写条件分支。

## 2. AppWorkspace

`AppWorkspace` 持有 `*app.App` 和 `*config.ConfigStore`。大部分方法是一行委托，但有几个重要例外：

### AgentRun

本地 `AgentRun` 启动 goroutine 调用 Coordinator，使 UI 提交动作快速返回；错误通过 App 事件发回，而不是阻塞 Update。

### Shell command

`AgentRunShellCommand` 使用 Shell 流式回调，必要时把命令/输出保存成用户消息，并触发标题生成。这是用户显式 Shell 与模型 Bash tool 的不同入口。

### LSP/MCP 类型转换

App 内状态转换成 Workspace 前端类型，避免 UI 依赖 App 包的内部全局状态结构。

### Subscribe

订阅 App 的统一 Broker，把每个 `tea.Msg` 送入 `tea.Program.Send`。它还负责等待订阅 goroutine，以便 Shutdown 不在消息仍发送时销毁资源。

## 3. ClientWorkspace 的缓存

远程接口不能每帧发 HTTP，所以 ClientWorkspace 缓存：

- Workspace/Config 快照；
- Agent info、busy session 和 queued prompts；
- LSP/MCP states；
- Skills states；
- pending MCP auth；
- Permission skip；
- 当前连接错误与状态。

读方法有两类：强一致查询直接发 HTTP，如 ListMessages；高频展示读缓存，如 AgentIsBusy、Config、MCPGetStates。SSE 事件更新缓存，断线恢复后主动 resync。

## 4. Proto：跨进程领域镜像

`internal/proto` 为 Workspace、Session、Message、Agent、Permission、Question、LSP、MCP、Skills、Config 请求定义 DTO。

特别复杂的是 Message：`ContentPart` 是 tagged union，包含 Text、Reasoning、ImageURL、Binary、ToolCall、ToolResult、Finish、ShellCommand。`MarshalParts`/`UnmarshalParts` 用显式 type tag 保存多态。ClientWorkspace 再将 Proto Part 转为内部 `message.ContentPart`。

为什么不直接 JSON 序列化内部领域对象？因为内部接口、error、Provider 类型和演进节奏不适合作为稳定 wire contract。Proto 层可以定义文本枚举、定制 MarshalJSON，并让 Client/Server 分别转换。

## 5. Server 层

`server.Server` 创建 `http.ServeMux`，路由大致分为：

- health/version/server control；
- Workspace create/get/delete/events/current-session；
- Session/Message/History/FileTracker；
- Agent run/init/update/cancel/queue/summarize/shell；
- Permission/Question；
- Config mutation/OAuth；
- LSP；
- Skills；
- MCP；
- Swagger。

Handler 的标准工作是：解析 path/query/body → 调 Backend → 把领域错误映射到 HTTP status/Proto Error → 编码 JSON。不要在 handler 中复制 Workspace 去重或 Agent 生命周期逻辑。

生产 transport 在 Unix 类系统使用 Unix socket，在 Windows 使用 named pipe。Client 的 dialer 把 HTTP 请求连接到该本地 transport，HTTP 仍只是应用协议。

## 6. Backend：真正的多工作区业务层

`Backend` 持有：

- `workspaces`：ID 到 Workspace；
- `pathIndex`：规范化绝对路径到 Workspace ID；
- `pending`：尚在慢初始化阶段的 create 数量；
- idle shutdown timer 与 closing latch；
- retired client IDs；
- ConfigStore、Context 和 shutdown callback；
- create/detach/idle 三种 grace 时长。

`backend.Workspace` 嵌入 `*app.App`，并增加 ID、Path、Config、Env、Skills、规范化路径、Workspace Context、Run WaitGroup、closing 状态和 client claims。

## 7. CreateWorkspace 逐步解析

1. 校验 path 和 client UUID；
2. Abs + EvalSymlinks 得到去重 key；
3. 在 Backend mutex 下检查 closing/retired；
4. 取消待执行的 idle shutdown；
5. 若 pathIndex 已存在，校验 channels 一致、记录 first-wins flag 差异、注册客户端 claim，直接复用；
6. 否则增加 `pending` 后释放锁，执行慢初始化；
7. `config.Init`，应用 YOLO/channels runtime override；
8. 创建数据目录、连接带 data-dir lock 的 DB；
9. 每 Workspace 发现 Skills，Server 模式不使用 GlobalMirror，防止多工作区相互覆盖；
10. `app.New`；
11. 创建 Workspace Context 和 Backend Workspace；
12. 重新拿锁，再次检查 client 是否退休；
13. 重新检查 pathIndex，处理并发创建同路径时的 loser；
14. winner 注册 workspaces/pathIndex/client claim；
15. 版本不一致时发布警告；
16. deferred 减少 pending，并在失败后重新判断是否应 idle shutdown。

这里的“双重检查”是典型并发初始化模式：慢 IO 不能持全局锁，但放锁后另一个 goroutine 可能先创建成功，因此提交阶段必须再次去重，并关闭 loser 已创建的 App/DB。

## 8. Client claim 状态机

一个 Workspace 可被多个 Client 共享。每个 `clientState` 保存：

- `streams`：当前 SSE 流数量；
- `holdTimer`：创建后等待首次 attach，或断线后等待 reconnect；
- `currentSessionID`：该 Client 当前查看的 Session；
- `released`：Client 已明确放弃 claim。

```text
CreateWorkspace
  -> creation hold
  -> SSE Attach：停止 hold，streams++
  -> SSE Detach：streams--
      -> clean release：立即移除 claim
      -> accidental drop：detach grace
          -> reconnect：恢复 stream
          -> grace 到期：移除 claim
  -> 所有 claims 消失：teardown workspace
  -> 所有 workspace/pending 消失：idle shutdown timer
```

`RetireClient` 是权威“这个 Client 永远退出”的信号，会释放它在所有 Workspace 的 claim，并把 UUID 永久放入 retired set，拒绝仍在网络途中迟到的 Create 请求。

## 9. 为什么需要三个 grace

- **createGrace**：Create HTTP 成功后，Client 还没来得及建立 SSE，防止 Workspace 被立即回收。
- **detachGrace**：网络闪断或笔记本暂停时给同一 Client 重连机会。
- **idleShutdownDelay**：最后一个 Workspace 消失后稍等，避免用户快速切换目录时 Server 退出与新建发生竞态。

这三者解决不同窗口，不能合并成一个超时。

## 10. Agent fire-and-forget 与 RunID

Server 的 `Backend.SendMessage` 不等待模型完成：

1. 找 Workspace/Coordinator；
2. 调 `agent.ValidateCall`；
3. `BeginAccepted` 预留一次尚未进入 SessionAgent 的运行；
4. 在 runMu 下拒绝 closing，增加 runWG；
5. goroutine 调 `RunAccepted`，绑定 Workspace Context；
6. 返回 HTTP accepted。

HTTP 请求 Context 不能作为 Run Context，否则响应结束或 Client 断线会取消模型。RunID 从请求写入 Context，并在权威 RunComplete 中原样返回。

如果错误发生在 SessionAgent 能发布 RunComplete 之前，Backend 检查 run-complete marker 并补发 must-deliver 错误完成，避免 `crush run` 永远等待。

## 11. SSE 事件

Server 订阅 App events，将不同 `tea.Msg` 转成带类型的 Proto 事件并编码为 SSE。Permission/Question 请求、Message 更新、Agent notification、RunComplete、LSP/MCP/Skills/Config 等都经过这里。

建立 SSE 时 Backend `AttachClient`，退出时 deferred `DetachClient`。所以事件流同时是数据通道和客户端资源 claim。

## 12. ClientWorkspace 重连

`runSubscription` 是长期循环：

1. 打开当前 Workspace 的 events stream；
2. 成功后发布 ConnectionRecovered；
3. `consumeEvents` 转译事件并更新缓存；
4. stream 关闭则标记 degraded；
5. 若 Server 返回 workspace gone，调用 `recoverWorkspace` 重新 Create；
6. Create 可能得到新 Workspace ID，原子更新本地快照；
7. `afterReconnect` 全量刷新 Workspace/Agent/LSP/MCP/Skills 等缓存；
8. 指数 backoff 后继续；
9. 连续失败达到阈值时发 Stuck 状态，但仍不停重试。

Shutdown 要和正在进行的 recovery 协调，等待 subscription/recovery 停止，再 Release Workspace 并 RetireClient。若 create response 丢失，RetireClient 仍能让 Server 清理未知 ID 的孤儿 claim。

## 13. ConfigChanged 的多客户端一致性

一个 Client 修改 ConfigStore 后，Server 发布 ConfigChanged。所有同 Workspace Client 收到后刷新配置快照，因此兄弟 TUI 能看到主题/模型等变化。这里是事件触发的最终一致性，而不是共享内存。

## 14. Channel opt-in

MCP channels 是外部能力授权。相同路径的多个 Client 共享同一 Workspace 时，后来的 Client 请求 channels 必须与已有 Workspace 完全一致，否则返回 `ErrChannelOptInMismatch`。这是防止第二个 Client 悄悄扩大第一个 Client 的工具边界。

## 15. 锁顺序与关闭

需要同时持 `Backend.mu` 和 `Workspace.clientsMu` 时，固定先 Backend 再 Workspace。Detach 路径短暂持 clientsMu 后释放，再进入可能取 Backend.mu 的 teardown，避免 AB/BA。

Workspace 关闭顺序：

1. runMu 下设置 closing；
2. cancel Workspace Context；
3. Coordinator.CancelAll；
4. `runWG.Wait()`；
5. `app.App.Shutdown()` 关闭 MCP/LSP/DB/Broker。

顺序反过来会让仍运行的 Agent 在 DB 关闭后写消息。

## 16. 重实现建议

1. 先定义 Workspace 接口和一个内存 AppWorkspace；
2. 定义最小 Proto DTO；
3. 实现无状态 Server handler + Backend 单 Workspace；
4. 实现 Client 与同步调用；
5. 添加 SSE 事件；
6. 添加 path 去重和多 Client claims；
7. 添加 create/detach/idle grace；
8. 添加 recovery/resync 和 RetireClient；
9. 最后处理版本差异、channels、run completion 等可靠性边界。
