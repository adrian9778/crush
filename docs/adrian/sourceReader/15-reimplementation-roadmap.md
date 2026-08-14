# 15. 从零重新实现 Crush 的路线图

> 状态：待复核生成稿｜生成日期：2026-08-14
> 基准提交：`5712d4839a6a10e9940804d511bb322dbe73a511`｜工作区：clean（开始分析时）
> 源码范围：全仓库模块、构建、测试、配置与部署资产
> 生成方式：源码依赖顺序与客观行为契约静态分析

## 快速摘要

### 架构总览（模块与依赖）
重实现顺序从领域模型和最小 Agent 闭环开始，再逐层加入工具安全、持久化、配置、App/TUI、Shell/LSP/MCP，最后加入远程架构和可靠性。

### 核心调用序列（逐步逻辑）
1. 定义不变量。2. 完成内存服务与假 Provider。3. 加入工具和权限。4. 加入 SQLite/配置。5. 接入 UI 与基础设施。6. 扩展 Client/Server 并做故障验收。

### 易错点与边界条件
不要先复刻 UI 外观；每阶段必须有正常、错误、取消、并发、重启和关闭验收；可替换技术实现不能改变外部行为契约。

本章不是现有构建命令，而是依据依赖关系安排“另写一套等价系统”的实施顺序。每一阶段都给出最小闭环和验收标准。

## 阶段 0：先写约束，不写功能

确定：

- Go 版本和跨平台范围；
- Workspace、Session、Run 的身份定义；
- 本地与远程是否都要支持；
- Provider abstraction；
- 权限和配置的安全模型；
- 数据持久化兼容目标；
- 事件可丢/不可丢分类。

输出领域词汇表、包依赖图和禁止反向依赖规则。否则后续很容易让 UI、HTTP 和 Agent 互相 import。

## 阶段 1：领域数据与内存 Service

实现：

- Session、Message、ContentPart、Attachment；
- SessionService、MessageService 的内存版本；
- 泛型 Broker；
- Created/Updated/Deleted 事件；
- ID 和时间注入以便测试。

验收：创建 Session，增量追加 assistant text/tool call/tool result/finish，另一个订阅者按顺序观察状态。

暂不接 SQLite、模型或 TUI。

## 阶段 2：最小 Provider 与 Agent 循环

定义自己的 `LanguageModel` 接口或先采用 Fantasy：输入 system/history/tools，输出 stream steps。

实现 SessionAgent 最小流程：

1. 校验 Session/Prompt；
2. 保存 user message；
3. 读取历史；
4. 调 Provider；
5. 保存 assistant text；
6. 保存 finish/usage；
7. 发布 RunComplete。

先用 deterministic fake Provider 测试，避免 API 网络影响架构。

## 阶段 3：工具协议

实现 `Tool`：Name、Description、InputSchema、Execute。先做只读 View、Glob/Grep，再做 Write/Edit，最后 Bash。

要求每个工具：

- 参数解析错误明确；
- Context 取消；
- 输出大小限制；
- 工作区路径规范化；
- 结构化 metadata；
- tool call/result 持久化。

验收：fake model 发 tool call，Agent 执行并把 result 回送下一 model step，最终生成文本。

## 阶段 4：Permission 与 Hook

先做 Permission pending request + grant/deny，再加 allow list、session auto approve、persistent key。随后实现 Hook 单命令、输出协议、deny/allow，再扩展并行、聚合、rewrite、halt、timeout。

固定顺序：Hook → final input → Permission → Execute。用两个并发 UI 模拟同时解决 Permission，证明只有一个 winner。

## 阶段 5：并发 Agent

增加：

- per-session active cancel；
- 同 Session queue；
- 不同 Session 并行；
- accepted reservation 和 sequence high-water；
- clear queue、cancel all；
- exactly-once-intent RunComplete；
- Provider 认证 retry coalescing。

用 channel 人工卡住 accepted→active、active→finish 各阶段，逐个测试取消窗口。

## 阶段 6：SQLite

先设计 migrations，再写 SQL/query，最后 Service adapter。建议表：sessions、messages（parts JSON）、files/history、read_files、stats。

要求：

- migration 可从每个历史版本升级；
- 跨表操作事务化；
- 保存成功后才发布事件；
- data-dir 连接引用和关闭清晰；
- 不直接修改 sqlc 生成代码。

用同一套 Service contract tests 同时验证内存和 SQLite 实现。

## 阶段 7：配置系统

先实现纯 `Config` + defaults + validation，再实现加载多来源和 deep merge，然后实现 ConfigStore 的 scope mutation/atomic write/staleness。

`crushrc` 可后加：先定义 ConfigBuilder 操作，再将 Shell builtin 映射到 Builder。不要让 builtin 直接散落修改全局 Config。

验收：全局+项目+JSON+crushrc 冲突有确定优先级；外部修改可检测；并发写不产生半文件。

## 阶段 8：Provider catalog、Model 与 Prompt

实现 ProviderConfig → Provider client 工厂，large/small selection、同名歧义、调用参数 merge。再实现 Prompt template、上下文文件、Git/平台信息。

Skills 分成：解析/验证 → 路径发现 → last-wins 去重 → disable filter → Catalog → Prompt XML → 按需 Read。Builtin 用 embedded FS 和虚拟 URI。

## 阶段 9：App 与本地 Workspace

用 `app.New` 集中装配所有 Service、Agent、Broker、LSP/MCP stub 和 cleanup。定义 Workspace 接口，先完成 AppWorkspace。

验收：一个简单 CLI 可创建 Workspace/Session、运行 fake Agent、观察事件、关闭后无 goroutine泄漏。

## 阶段 10：TUI 骨架

按顺序实现：

1. 唯一顶层 Bubble Tea UI；
2. landing/chat 两状态；
3. textarea 提交；
4. Session/Message 事件；
5. 简单 List/Chat；
6. streaming assistant；
7. tool renderer；
8. dialogs；
9. completions/attachments；
10. styles/theme/sidebar/status。

坚持 IO 在 Cmd，状态在 Update，View 纯渲染。先覆盖 80x24 和极窄终端，再做视觉丰富度。

## 阶段 11：Shell

实现无状态 Run、进程组取消和 capture，再实现 stateful Shell、handler chain、builtin registry、script dispatch、background jobs。

先确保 cancellation 和进程树清理，再把 Bash 暴露给模型；安全性优先于功能数量。

## 阶段 12：LSP

从一个固定 Go LSP 开始：initialize、didOpen、didChange、diagnostics。再抽 Manager/candidate matching，添加默认 catalog、root markers、restart、symbols/references/rename/call hierarchy 和 WorkspaceEdit 编码。

UTF-16/CJK/emoji 测试必须在 Rename 上线前完成。

## 阶段 13：MCP

先单个 stdio server 和 ListTools/CallTool，再做 registry/filter/state events。随后 resources/prompts、多个 server、HTTP/SSE、OAuth、generation、lazy renew、hot reconcile、channels。

每加一种生命周期操作，都测试“连接未完成时 disable/remove/reconfigure”。

## 阶段 14：Client/Server

先为 Workspace 的同步方法设计 Proto DTO 和 Backend，无 SSE。然后：

1. HTTP handlers/client；
2. SSE domain events；
3. ClientWorkspace cache；
4. path dedupe；
5. client claims；
6. create/detach/idle grace；
7. reconnect + full resync；
8. workspace recreation；
9. RetireClient；
10. RunID/reliable completion。

Server handler 不拥有业务状态；Backend 不依赖 HTTP；UI 不依赖 Proto。

## 阶段 15：可靠性和可观测性

加入：

- debug 文件日志与 HTTP 脱敏；
- Broker drop counters；
- update check；
- telemetry opt-out；
- panic recovery；
- goroutine/process leak tests；
- config/DB locks；
- shutdown deadlines；
- race tests。

最后才做 herdr、native notifications、terminal image 等外围体验。

## 每阶段通用验收模板

### 正常路径

给定输入能产生确定输出，状态成功持久化并发布正确事件。

### 错误路径

参数错误、外部服务错误、持久化错误均保留上下文，且不会报告假成功。

### 取消路径

Context 取消能停止当前 IO/进程，不损坏已提交状态，不留下 busy 标记。

### 并发路径

两个相同 key 和两个不同 key 同时操作的结果符合串行化边界。

### 重启恢复

进程退出重开后，持久数据可恢复，临时/active 状态不会伪装成仍运行。

### 关闭路径

拒绝新工作、取消生产者、等待 worker、关闭外部资源、最后关闭存储与事件。

## 建议的包实现顺序

```text
domain types
 -> pubsub/csync
 -> in-memory services
 -> provider abstraction
 -> agent loop
 -> tools/permission/hooks
 -> db services
 -> config/shellconfig
 -> prompt/skills
 -> app/workspace local
 -> TUI
 -> shell/LSP/MCP
 -> proto/backend/server/client
 -> recovery/reliability/observability
```

## 最终完成定义

当你能为以下动作画出完整时序并解释每个失败点，才算完成等价重实现：

> 两个 Client 打开同一路径，其中一个提交提示，模型读取和编辑文件、触发 Hook 与 Permission、LSP 返回诊断、消息同步到两个 TUI；随后 SSE 闪断并恢复，用户取消下一条排队提示，最后两个 Client 退出且 Server 无孤儿进程、无 goroutine、无未关闭 DB。

这条场景把 Crush 最重要的架构承诺串在一起。
