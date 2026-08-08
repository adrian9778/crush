# 07. 数据模型、SQLite、服务与事件

> 本章从数据库表一直讲到 Go 领域对象、流式写入和 UI 事件。按本文复刻时，应把“持久化事实”和“进程内暂态”明确分开。

## 1. 数据关系总览

```text
Session（一次对话）
  ├─ Message[]（用户、助手、工具消息；parts 为多态 JSON）
  ├─ File[]（本次会话关联的文件历史版本）
  ├─ ReadFile[]（本会话读过哪些路径、最后读取时间）
  └─ child Session[]（task agent、标题生成等内部会话）

Service CRUD
  -> sqlc Queries
  -> SQLite
  -> PubSub Event
  -> App 聚合事件
  -> TUI 或 Server SSE
```

SQLite 是权威持久层；PubSub 只负责变化通知，不是可靠日志。订阅者丢事件后应重新查询当前状态。

## 2. 数据库连接与迁移

关键文件：`internal/db/connect.go`、`connect_modernc.go`、`connect_ncruces.go`、`datadirlock.go`。

### 2.1 `Connect`

输入是 data directory，实际文件为 `<dataDir>/crush.db`。过程：

1. 拒绝空 dataDir。
2. 将数据库路径绝对化，作为进程内连接池 key。
3. 在 `poolMu` 下查 `map[path]*connEntry`；存在则引用计数加一并复用。
4. 确保数据目录以 `0700` 存在。
5. server 可用 `WithDataDirLock(true)` 获取目录锁；环境开关可跳过。锁随最后一个引用释放。
6. 用当前构建选择的 SQLite driver 打开连接。
7. `SetMaxOpenConns(1)`，将所有访问串行到一个底层连接。注释明确指出，多连接并发 write/checkpoint 曾导致 WAL/header 不同步与 `SQLITE_NOTADB`。
8. `PingContext` 验证。
9. `sync.Once` 初始化 Goose SQLite dialect。
10. 从 embed FS 执行全部 migrations。
11. 放入连接池，引用数为 1。

`Release(dataDir)` 减引用，归零才关闭 DB 和目录锁。测试用 `ResetPool` 强制清空。

`ConnectReadOnly` 用于跨项目统计：只读打开、不迁移、同样限制单连接。

### 2.2 SQLite pragmas

项目设置：foreign keys ON、WAL、page size 4096、temp store memory、约 8MB cache、synchronous NORMAL、secure delete ON、busy timeout 30 秒。意义：

- WAL 改善读写并行，但仍不意味着任意多 writer 安全。
- foreign keys 才会真正执行级联删除。
- busy timeout 避免短暂写锁立即失败。
- secure delete 降低敏感对话残留风险。

### 2.3 数据目录锁

进程内连接池只防同一进程重复打开；server 模式还用文件锁阻止另一个 Crush server 同时管理同一数据目录。本地模式暂未默认启用，以保持多本地实例兼容。复刻时不要把这两个锁混为一谈。

## 3. Schema 与迁移历史

迁移由时间戳排序，嵌入二进制。初始迁移建三张表，后续只做增量：

- 2025-05：session 增加 `summary_message_id`。
- 2025-06：created_at 索引；message 增加 `provider`。
- 2025-08：message 增加 `is_summary_message`；session 增加 `todos` JSON。
- 2026-01：新增 `read_files`。

### 3.1 `sessions`

字段：`id` 主键、可空 `parent_session_id`、title、message_count、prompt/completion tokens、cost、summary_message_id、todos、created_at、updated_at。

约束：计数/token/cost 不得为负。message_count 不是服务层手工维护，而由 messages 的 insert/delete trigger 增减。列表通常按 updated/created 时间排序；最近会话只选顶层会话，避免恢复到 task child。

### 3.2 `messages`

字段：id、session_id 外键、role、`parts` JSON 字符串、model、provider、is_summary_message、created/updated/finished 时间。Session 删除时级联删除。

最重要的设计是 `parts`：表结构无需为每种流式内容建子表，但 Go 层必须用显式 type discriminator 安全反序列化。

### 3.3 `files`

字段：id、session_id、path、content、version、时间；唯一约束 `(path, session_id, version)`。它保存 agent 修改前/过程中的内容快照，支持撤销和差异，不是磁盘文件索引。

### 3.4 `read_files`

以 session_id + path 唯一标识读取记录，重复读取采用 upsert 更新 `read_at`。它用于判断工具写文件前是否已看过最新内容，并可给 agent 列出本会话上下文文件。

### 3.5 时间单位陷阱

初始注释称时间为毫秒，但 SQLite trigger 使用 `strftime('%s','now')`（秒）。Go 层与查询应以实际生成代码/测试为准，不要只依赖注释。复刻时应统一单位并为迁移/触发器写断言测试。

## 4. sqlc 层

原始 SQL 在 `internal/db/sql/*.sql`，生成代码在 `internal/db/*sql.go`、`models.go`、`querier.go`。修改查询应编辑 `.sql` 后重新生成，禁止手改生成文件。

`DBTX` 抽象 `*sql.DB` 与 `*sql.Tx` 的共同方法；`Queries.WithTx(tx)` 让相同生成查询参与事务。`db.Querier` 接口便于 message 测试注入慢写/错误实现。

主要查询：

- sessions：Create/Get/GetLast/List/Update/Rename/Delete/UpdateTitleAndUsage。
- messages：Create/Update/Get/List、只列 user、按 session 删除。
- files：按 ID、路径、session 查询，最新版本与新文件查询。
- read_files：upsert、Get、List。
- stats：按日、模型、小时、星期、工具聚合 token/cost/响应时间。

## 5. Session 领域服务

关键文件：`internal/session/session.go`。

### 5.1 `Session`

字段几乎映射数据库，另有 `EstimatedUsage` 纯内存标志；`Todos []Todo` 从 JSON 解析。Todo 状态为 pending、in_progress、completed，`HasIncompleteTodos` 检查是否仍有未完成项。

`HashID` 用 XXH3 将 UUID 转成短显示/定位 hash，但不能替代数据库主键或安全 token。

### 5.2 Service 接口

提供 Create、CreateTaskSession、CreateTitleSession、Get/GetLast/List、Save、Rename/Delete、UpdateTitleAndUsage，以及 agent tool session ID 编解码。服务嵌入 `Broker[Session]`，CRUD 后发布 Created/Updated/Deleted。

### 5.3 三种会话

- 顶层用户会话：随机 UUID，无 parent。
- task child：parent 指向用户会话，ID 由 message/tool-call 组合形成，方便从工具调用反查。
- title child：为生成标题建立，避免内部消息污染主会话。

`resolveSession` 与 CLI 明确拒绝继续 child 或 agent-tool session。

### 5.4 删除事务

删除会话不仅删一行：它要清理 child、消息和文件，并保持事件可观察。服务持有 `*sql.DB` 是为了在需要时建立事务。外键 cascade 是最后保障，但若调用者需要为每个资源发 DeletedEvent，就必须先读取并通过各 service 删除或显式发布。

### 5.5 EstimatedUsage

这是进程内 map 状态，用 mutex 保护。更新真实 token/cost 后清除估算标志；从 DB 转领域对象后再叠加当前进程的估算状态。它不适合作为跨进程权威事实。

## 6. Message：最复杂的数据对象

关键文件：`internal/message/content.go`、`message.go`、`attachment.go`。

### 6.1 Role 与 `Message`

角色包括 user、assistant、tool 等 Fantasy 协议所需类型。Message 保存 ID、SessionID、Role、Parts、Model、Provider、summary 标志与各时间。

非 assistant 消息创建时自动追加 `Finish{Reason: stop}`，因为它们不会经历模型流式结束。

### 6.2 ContentPart 多态结构

`ContentPart` 是带私有 `isPart()` 的封闭式接口。实现包括：

| 类型 | 用途 |
|---|---|
| `TextContent` | 用户/助手文本 |
| `ReasoningContent` | 思考文本、签名与时长信息 |
| `ImageURLContent` | URL 图片与 detail |
| `BinaryContent` | MIME + 二进制附件 |
| `ToolCall` | 调用 ID、名称、输入、完成状态 |
| `ToolResult` | 对应 tool call 的内容、错误标志 |
| `Finish` | stop/error/cancel 等结束原因、消息、详情 |
| `ShellCommand` | TUI 展示 shell 命令状态 |

`marshalParts` 不直接 marshal interface，而是为每项写 type + raw data wrapper；`unmarshalParts` 按 type switch 还原。新增 part 类型必须同时更新两端，否则旧消息无法读取。

### 6.3 Message 辅助方法

- `Content`/`ReasoningContent` 汇总对应 part。
- `ToolCalls`/`ToolResults` 类型筛选。
- `AppendContent`、`AppendReasoningContent` 尽量合并相邻 part，减少数组膨胀。
- `AppendToolCallInput` 按 ID 追加流式 JSON 参数。
- `FinishThinking`、`FinishToolCall` 结束结构阶段。
- `AddFinish` 添加权威结束 part。
- `Clone` 深拷贝 Parts，跨 goroutine 发布前必须使用。
- `ResetStreamedContent` 为重试清掉流式助手内容而保留适当上下文。
- `ToAIMessage` 转 Fantasy message；损坏媒体替换为占位文本，不能令整个历史不可用。

### 6.4 Attachment

Attachment 保存名称、MIME、内容。`PromptWithTextAttachments` 把文本附件嵌入 prompt；图片/二进制由 ContentPart 送模型。Base64/媒体解析失败有测试保证降级而非 panic。

## 7. 流式 Message 的 debounce 写入

模型可能每个 token 更新一次，若每次写 SQLite 并广播会严重拖慢 TUI。`message.service` 为每个 message ID 保存 `pendingState`，默认窗口 33ms。

### 7.1 `pendingState`

- `latest`：最新内存快照。
- `dirty`：尚未写盘。
- `flushing`：已有 goroutine 写该 ID，防写入重排。
- `timer`：debounce timer。
- `lastFlushed` / `hasFlushed`：识别结构性或终止变化的基线。

所有字段由 service mutex 保护。SQL 写入时释放 mutex，避免全局阻塞；`flushing` 保证同 ID 序列化。

### 7.2 Update 决策

1. 深克隆传入 Message，避免调用者继续修改 Parts。
2. debounce <= 0 时同步 flush。
3. 写入 pending.latest 并置 dirty。
4. `shouldFlushNow(previous, next)` 判断：出现 Finish、tool call 新增/完成、reasoning 结束等结构性/终止变化必须同步写。
5. 普通文本 delta 只启动一次 timer；timer 用 `context.Background()`，避免调用者 stream context 刚取消就把最终缓冲永远留在内存。

### 7.3 `flushOne`

同步 caller 遇到正在 flush 会等待并重试；timer caller 则退出。它在锁下获取 snapshot、设置 flushing，锁外执行 SQL，再锁内更新 lastFlushed。若写期间又收到 Update，dirty 会再次变 true，循环继续，保证同步 Flush 返回时磁盘至少包含调用时最新状态。

写成功后发布克隆的 UpdatedEvent。终止/结构事件使用 `PublishMustDeliver`；中间 token 使用普通 lossy Publish。

### 7.4 强一致读取边界

`Update` 对普通 delta 是最终一致。需要立刻读取最新消息的路径——会话切换、shutdown、run complete、测试——必须先 `Flush(id)` 或 `FlushAll()`。这是复刻时最容易漏掉的契约。

删除消息会停止 timer 并移除 pending，防止稍后 timer 把已删除行“写回来”。

## 8. File History

关键文件：`internal/history/file.go`。

`Create` 创建 version 0；`CreateVersion` 先查同路径最新版本再 +1。插入放在事务中，若 `(path, session, version)` 唯一冲突则最多重试三次并递增版本。这用于处理并发 sub-agent 同时记录历史。

注意当前“查最新版本”在事务之外，真正的冲突由唯一约束与重试兜底。更严格的复刻可在单事务中执行 `MAX(version)+1`，但仍需处理 SQLite writer contention。

创建/删除后发布 File 事件。`ListLatestSessionFiles` 用 SQL 为每个 path 取该 session 最新版本。

## 9. FileTracker

`RecordRead` 是 best-effort：SQL 错误只记录日志，不让一次 view 操作整体失败。路径先相对当前进程 cwd 归一化，列表时再拼回绝对路径。

这隐含约束：Workspace 工作期间 cwd 应稳定；否则记录和恢复会使用不同基准。更独立的实现可把 workspace root 注入 service，而不是每次 `os.Getwd()`。

`LastReadTime` 查询失败返回 zero time，调用者把它理解为“从未读过”。`ListReadFiles` 则返回错误，因为完整列表通常用于明确操作。

## 10. Permission Service

关键文件：`internal/permission/permission.go`。

### 10.1 请求模型

`CreatePermissionRequest` 是工具传入参数；服务补 UUID 和规范化目录后生成 `PermissionRequest`。请求包含 session、toolCall、tool、action、path、description、params。另有 `PermissionNotification` 告诉 UI 某 tool call 已允许或拒绝。

### 10.2 判断顺序

1. 全局 skip/yolo：立即允许。
2. `allowedTools` 命中 `tool:action` 或 `tool`：允许。
3. PreToolUse hook 已在 context 标记同 toolCallID：允许，并仍发 granted notification 留审计痕迹。
4. 发布“正在请求”通知。
5. session 是非交互 auto-approve：允许。
6. 将文件路径规范成目录；`.` 替换为 workingDir。
7. 检查 session persistent permission key `(session, tool, action, path)`。
8. 放入 pending request map，发布 Request，阻塞等待 UI response 或 context cancel。

`requestMu` 令请求串行，符合 TUI 一次只展示一个 permission dialog 的能力。response channel 容量 1。

### 10.3 解决竞态

Grant/Deny 通过 pending map 的原子 Take 决胜，只有第一个成功。`GrantPersistent` 仅在它赢得请求竞态后写 session permission；否则一个输给 Deny 的迟到 Grant 会错误地允许未来调用。

skip 用 atomic，active request 与 auto-approve/session permission 各有独立锁或并发 map，避免一把大锁覆盖阻塞等待。

## 11. Question Service

关键文件：`internal/question/question.go`。

支持 yes/no、single choice、multi choice、free text。Request 可包含最多 5 个问题，每题/描述/选项均有长度与数量限制；错误文案刻意写给 LLM，指出应使用 `choices` 而非 `options` 等修复方式。

`Ask` 自动补 batch/question UUID，多问题补确认标题和描述，校验后建立 answer/cancel channel，发布 Request 并阻塞。`Answer` 或 `Cancel` 在 mutex 下取走 pending；解决后发布 Notification，让未回答的其他客户端关闭表单。

只有一个 pending，因工具调用本身同步等待。这与 Permission 相同：若未来支持多个并行询问，需要引入 request-ID 到 channel 的 map，而不能复用单 channel。

## 12. PubSub Broker

关键文件：`internal/pubsub/broker.go`、`events.go`。

### 12.1 数据结构

泛型 `Broker[T]` 持有订阅 channel set、RWMutex、done channel、订阅数、buffer 配置与两个 atomic drop counter。每个订阅者默认 buffer 4096；context 取消时自动移除并关闭 channel。

### 12.2 两种投递

- `Publish`：逐订阅者非阻塞发送，满则丢弃、警告、dropCount++。适合 token delta。
- `PublishMustDeliver`：先快路径发送，满则每订阅者最多阻塞 50ms；超时仍会丢，但记录单独计数。适合 finish、tool result、error、cancel、RunComplete。

“MustDeliver”不是无限可靠队列，只是 bounded best effort。其设计保证慢 UI 不会永久卡住 agent。

`Shutdown` 关闭 done，再在锁下关闭所有订阅 channel。Publish 持 RLock，避免与关闭 channel 并发产生 send-on-closed panic。

### 12.3 事件协议

资源事件类型只有 created/updated/deleted。跨 RPC 时 `Payload{Type, RawMessage}` 加判别标签，包括 session、message、file、permission、question、LSP、MCP、skills、config changed、run complete 等。

## 13. App 事件桥接

`app.setupEvents()` 分别订阅 Session、Message、History、Permission、Question、LSP/MCP/Agent 等 broker，把领域事件包装成 `tea.Msg` 或 proto payload 后发布到 App 总 broker。每条 bridge goroutine 都受 `eventsCtx` 管理并登记 WaitGroup；关闭时先取消 context、等桥接退出，再 Shutdown broker，避免向已关闭 channel 发布。

关键原则：领域 service 不导入 TUI；App 是适配层。

## 14. 错误、事务和并发清单

- SQLite 连接限制为 1，不代表可省掉 Go mutex；Message 的内存 pending 仍需锁。
- 事务失败必须 rollback；commit 失败也不能发布 CreatedEvent。
- 事件只在持久化成功后发布，否则 UI 会显示数据库不存在的对象。
- 发布 slice/map 前深克隆，避免订阅者与 agent 并发读写。
- 普通流事件可丢；终止事件用 bounded must-deliver，并提供重新查询/最终对账。
- 删除必须清掉 pending timer。
- RunComplete 前必须 FlushAll，保证调用方收到完成事件时能够读到最终消息。
- 子会话要与用户会话隔离，List/GetLast 和 continue 必须过滤。
- best-effort 服务（read tracking）可吞错误；权威数据（message/session）不可吞。

## 15. 从零复刻步骤

1. 写四张表和外键/唯一约束，先不写 Service。
2. 写迁移 runner；启动临时 DB 后检查每列、索引、trigger。
3. 用 sqlc 生成 Queries，保持 SQL 是唯一查询源。
4. 实现泛型 Broker，并测试取消、Shutdown、满 buffer、must-deliver timeout。
5. 实现 Session CRUD + 事件；再实现 child session ID 与过滤。
6. 定义 ContentPart 判别序列化，逐类型 round-trip 测试。
7. 实现同步 Message CRUD，先确保最终状态正确。
8. 再加入 per-ID debounce、terminal detection、Flush/FlushAll；使用慢 Querier 做竞态测试。
9. 实现 File History 唯一冲突重试与事务。
10. 实现 FileTracker，并将 workspace root 显式注入会更稳健。
11. 实现 Permission 的 pending map/单次决胜，测试 Grant 与 Deny 竞争。
12. 实现 Question 单 pending 状态机与严格 validation。
13. 在 App 层桥接所有事件；最后才接 TUI/SSE。
14. 做压力测试：同 message 高频 delta、并行文件版本、慢订阅者、关闭时 in-flight SQL。

## 16. 推荐阅读顺序

```text
internal/db/migrations/*.sql
internal/db/sql/*.sql
internal/db/connect.go + driver 文件 + datadirlock.go
internal/db/models.go + querier.go（生成代码只需理解形状）
internal/pubsub/events.go + broker.go
internal/session/session.go
internal/message/content.go + message.go + attachment.go
internal/history/file.go
internal/filetracker/service.go
internal/permission/permission.go
internal/question/question.go
internal/app/app.go 中 setupEvents/Close
```

测试重点：`connect_test.go`、`session_test.go`、`message_test.go`、`history/file` 相关测试、`filetracker/service_test.go`、`permission_test.go`。特别是 Message 测试覆盖 debounce 顺序、同步终止写、in-flight Flush 与 PubSub 饱和，是重新实现时不可省略的验收清单。
