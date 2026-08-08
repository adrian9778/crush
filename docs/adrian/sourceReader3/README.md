# Crush 源码重实现手册

> 目标读者：第一次阅读大型 Go 工程、终端 UI 或 AI Agent 的开发者。
>
> 目标：不仅知道“文件在哪里”，还要理解为什么这样分层、对象如何创建和销毁、状态如何流动，以及如果从零重写应采用什么顺序。

## 1. 如何使用这套文档

不要从 `internal/` 随机挑文件阅读。Crush 的局部实现高度依赖全局生命周期：例如一次工具调用同时经过 Agent、Hook、Permission、Message 持久化和 TUI renderer；只看工具文件会漏掉一半行为。

建议按以下顺序：

1. [01-system-overview.md](01-system-overview.md)：先建立完整心智模型。
2. [02-bootstrap-cli.md](02-bootstrap-cli.md)：从进程入口走到可用 Workspace。
3. [03-config-providers-auth.md](03-config-providers-auth.md)：理解配置、Provider、模型与认证。
4. [04-agent-runtime.md](04-agent-runtime.md)：理解一次 Agent Run 的执行循环。
5. [05-tools-permissions-hooks.md](05-tools-permissions-hooks.md)：理解模型如何安全地作用于真实世界。
6. [06-skills-prompts-context.md](06-skills-prompts-context.md)：理解系统提示、上下文和技能。
7. [07-data-model-persistence.md](07-data-model-persistence.md)：理解 Session、Message、SQLite 和事件。
8. [08-workspace-client-server.md](08-workspace-client-server.md)：理解本地与远程两种运行形态。
9. [09-shell-hooks-lsp-mcp.md](09-shell-hooks-lsp-mcp.md)：理解 Shell、LSP、MCP 等基础设施。
10. [10-concurrency-lifecycle-reliability.md](10-concurrency-lifecycle-reliability.md)：集中理解并发与可靠性。
11. [11-tui-architecture.md](11-tui-architecture.md)：理解唯一顶层 Bubble Tea Model。
12. [12-tui-chat-rendering.md](12-tui-chat-rendering.md)：理解聊天消息与工具渲染。
13. [13-tui-dialog-styles-performance.md](13-tui-dialog-styles-performance.md)：理解 Dialog、样式与性能。
14. [14-package-file-index.md](14-package-file-index.md)：按包和文件定位源码。
15. [15-reimplementation-roadmap.md](15-reimplementation-roadmap.md)：按依赖顺序从零复刻。
16. [16-source-manifest.md](16-source-manifest.md)：核对全部生产源码与非 Go 行为源。

## 2. 阅读符号约定

- **领域对象**：Session、Message、Permission 等业务数据。
- **Service**：对领域对象提供操作和事件订阅的接口。
- **Workspace**：一个工作目录及绑定其上的配置、DB、Agent、LSP、Skills、MCP 和事件生命周期。
- **Run**：用户提交一次提示后发生的一次顶层 Agent turn。
- **Part**：一条 Message 中的文本、推理、工具调用、工具结果或结束标记。
- **Broker**：进程内泛型发布/订阅器，不是外部消息队列。
- **DTO/Proto**：跨 Client/Server 边界传输的数据结构。
- **生成代码**：sqlc 和 Swagger 生成结果，应理解其来源，不应直接当作手写业务逻辑修改。

## 3. 源码规模与覆盖方法

本次盘点的生产代码约 381 个 Go 文件、9.5 万行，分布在 67 个源码目录。文档采用三层覆盖：

1. 端到端调用链覆盖真实用户动作；
2. 专题章节覆盖核心机制与不变量；
3. 包/文件索引覆盖所有生产代码目录。

测试代码不是简单附属物。Agent 取消、Backend 关闭、ClientWorkspace 重连、Permission 多客户端竞争、UTF 编码编辑、TUI 窄窗口等关键规则主要由测试表达。各专题会指出必须阅读的测试类别。

## 4. 新手阅读时始终问六个问题

面对任何类型或函数，都尝试回答：

1. 谁创建它？
2. 谁持有它？
3. 谁并发调用它？
4. 状态写入内存、数据库还是文件？
5. 状态变化通过返回值还是事件传播？
6. 谁负责取消与关闭？

只要其中一个问题答不上来，就说明还没有真正理解这个组件的生命周期。

## 5. 事实来源优先级

遇到文档与代码不一致时，以当前分支的生产代码和测试为准，其次是仓库内 AGENTS.md 与专题文档，再其次才是公开 README。特别是配置格式已经从 JSON 演进为首选 `crushrc`，Client/Server 架构也仍在持续演进。
