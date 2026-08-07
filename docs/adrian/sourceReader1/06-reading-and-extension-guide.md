# 06. 阅读路线、修改定位与易错点

## 1. 推荐的实际断点路线

若要在调试器中观察一次交互，按顺序设断点：

1. `cmd.rootCmd.RunE`；
2. `setupLocalWorkspace` 或 `setupClientServerWorkspace`；
3. `app.New`；
4. `UI.Update` 的提交分支；
5. `AppWorkspace.SendMessage` 或 `ClientWorkspace.SendMessage`；
6. `Coordinator.run`；
7. `sessionAgent.Run`；
8. 工具 wrapper/具体 Execute；
9. `message.Service.Save`；
10. UI 收到 Message Updated 事件的分支；
11. RunComplete 发布与消费。

## 2. 按需求找修改点

| 需求 | 首先阅读 | 通常还要修改 |
| --- | --- | --- |
| 新 CLI 命令 | `internal/cmd` | root 注册、proto/backend（若远程） |
| 新配置项 | `config.Config` / `ConfigStore` | shellconfig、schema、默认值、UI |
| 新 Provider | `config/provider`、Coordinator build model | OAuth、discover、model UI |
| 新内置工具 | `agent/tools` | Prompt 描述、权限、UI renderer、测试 |
| 新 MCP 能力 | `agent/tools/mcp` | config、OAuth、proto/UI 状态 |
| 新 LSP 操作 | `lsp.Client` | Agent tool、WorkspaceEdit、chat renderer |
| 新数据库字段 | migrations / SQL | sqlc、Service、proto、UI |
| 新 Dialog | `ui/dialog` | `UI.Update` 编排、Styles、keys |
| 新 chat renderer | `ui/chat` 工厂 | cache、compact/expand、raw render |
| 新远程 API | proto + backend | server route、client、ClientWorkspace |
| 新 Skill | `internal/skills/builtin/<name>/SKILL.md` | discovery 测试；无需额外 embed wiring |
| 新 shell builtin | `shell.RegisterBuiltin` 或 handler | Context 取消、pipe IO、测试 |

## 3. 常见误区

### 3.1 把 App 与 Backend 当成同一层

App 是一个 Workspace 内部装配；Backend 是 Server 对多个 Workspace 的生命周期与业务门面。把 client refcount、路径去重写进 App 会污染本地模式。

### 3.2 在 HTTP 请求 Context 上跑 Agent

`SendMessage` 是 fire-and-forget。Run 必须绑定 Workspace Context，否则请求返回或 SSE 重连会取消仍应继续的模型 turn。

### 3.3 只发布普通错误通知

普通 Broker 允许丢事件。任何等待确定结束的调用都要依赖带 RunID 的 must-deliver RunComplete。

### 3.4 直接改 Config 指针或配置文件

应通过 ConfigStore 的 scope-aware 原子更新，否则多进程锁、OAuth token、staleness snapshot 与内存配置可能分叉。

### 3.5 只改本地 Workspace

TUI 面向接口。新增能力如果只写 `AppWorkspace`，Client/Server 模式会缺失；需检查 proto、Backend、Server、Client、ClientWorkspace 的完整链。

### 3.6 在 UI 子组件建立第二套 Update

项目规定顶层 UI 是唯一 Bubble Tea Model。子组件使用显式方法，避免消息路由、焦点和 Cmd 生命周期分裂。

### 3.7 用 `len` 计算终端文本宽度

ANSI、CJK、emoji 都会让字节长度不等于列宽。必须使用 `github.com/charmbracelet/x/ansi`。

### 3.8 手改生成代码

DB 查询改 SQL 源，API 文档改路由注释/生成流程。手改 `*.sql.go` 或 Swagger 生成文件会在下次生成时丢失。

## 4. 并发不变量

- 同 Session Run 交接由 per-session mutex 串行化；
- AcceptedRun 必须恰好 Close，避免 busy 永久为真；
- Cancel high-water 只覆盖取消前已接受的请求；
- Workspace closing 后不能增加新的 runWG；
- 关闭顺序必须先取消/等待 Agent，再释放 DB；
- Backend 与 clientsMu 的锁顺序不可反转；
- PubSub 订阅应绑定 Context，防止泄漏；
- DB/Config 文件级写入需事务或锁与原子替换。

## 5. 安全边界

需要特别审查的输入包括：模型提供的文件路径、Shell 命令、MCP URL/headers、Hook 修改后的 tool input、LSP WorkspaceEdit、OAuth redirect、远程 Workspace path。安全措施包括路径规范化、工作区边界、permission、allow list、Hook policy、超时、输出上限、错误脱敏和 channel opt-in 一致性。

“模型提出”不等于“用户授权”。Hook 在权限前执行是有意设计，但 Hook 明确批准应通过带 tool call ID 的 Context 精确传递，不能变成对整个 Session 的隐式全允许。

## 6. 测试策略

项目测试习惯：`require` 断言、`t.Parallel()`、`t.TempDir()`、`t.SetEnv()`。Provider 测试启用 mock providers 并在结束后 Reset。UI 使用 Catwalk/golden 时只有确认视觉变化正确后才更新 golden。

按改动选择最小测试，再执行较大范围：

```bash
go test ./internal/agent/...
go test ./internal/config/...
go test ./internal/ui/...
go test ./...
```

Go 代码必须格式化，优先 `gofumpt -w .`，其次 goimports/gofmt。日志文本首字母必须大写，独立行注释首字母大写并以句号结束。

## 7. 新手练习顺序

1. 给一个纯 helper 增加测试，熟悉 Go 测试；
2. 追踪一个 Session Created 事件从 DB 到 UI；
3. 给已有工具增加一个只影响展示的字段；
4. 给 Generic tool renderer 增加小改进；
5. 增加一个无副作用 shell builtin；
6. 最后再尝试 Provider、数据库 migration 或 Client/Server 协议变更。

## 8. 判断是否真正完成改动

不要只看编译通过。对任何跨层功能，写出一条“用户动作 → 接口 → 领域服务 → 持久化/外部系统 → 事件 → UI”的闭环，并确认取消、错误、远程模式、重启恢复和关闭各自会发生什么。能完整回答这五个问题，才算理解并完成了改动。
