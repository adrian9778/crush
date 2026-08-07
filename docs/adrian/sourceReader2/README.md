# Crush 项目源码文档

本目录对 [Crush](https://github.com/charmbracelet/crush) 项目源码进行深度解析，按主题分拆为多篇文章。

## 文章总览

| # | 文档 | 内容概要 | 阅读建议 |
|---|------|---------|---------|
| 1 | [architecture-overview.md](architecture-overview.md) | **整体架构**：两种运行模式、模块关系图、数据流图、配置管线、EventStream、Client/Server 通信协议栈 | ⭐ 必读第一篇，建立全局认知 |
| 2 | [agent-core.md](agent-core.md) | **Agent 引擎**：双模型策略、Coordinator vs SessionAgent、fantasy LLM 集成、Loop Detection、Auto-Summarize、Prompt Queuing、Accept Queue | Agent 是 Crush 大脑 |
| 3 | [config-system.md](config-system.md) | **配置系统**：多来源 deep merge (crushrc bash DSL + crush.json + CLI overrides)、crushrc builtins (provider/model/mcp/lsp/permissions/hook/options) | 理解如何定制行为 |
| 4 | [tools-and-hooks.md](tools-and-hooks.md) | **工具链**：25+ 内置工具的完整分类（文件/bash/LSP/web/MCP）；Hooks 系统（Decision/Aggregate/Halt Exit Codes/Runner） | Agent 的手和眼 |
| 5 | [backend-workspace.md](backend-workspace.md) | **Backend & Workspace**：C/S 业务逻辑、Workspace 生命周期 (Create→Operate→Grace→Teardown)、path dedup、Client lifecycle + Grace Periods | C/S 模式的精髓 |
| 6 | [tui-architecture.md](tui-architecture.md) | **TUI**：Hybrid rendering (Ultraviolet + Bubble Tea v2)、唯一 UI Model、Message Routing Switch Pattern、Chat Interface、Dialog Stack、Styling Three-Layer | 用户看到的一切 |
| 7 | [lsp-mcp-integration.md](lsp-mcp-integration.md) | **LSP & MCP**：LSP Manager (powernap lazy-load)、MCP Client Lifecycle (Arm→Initialize→Wait→Run)、三种连接模式 (stdio/SSE/HTTP)、Resources + Channels | 代码理解与外部服务 |
| 8 | [data-persistence.md](data-persistence.md) | **数据持久化**：SQLite+sqlc 体系、Session CRUD、Message Service、FileTracker、Stats / cost tracking | 所有运行状态 |

## 文档阅读路径建议

```
新手上手:
  architecture-overview → agent-core → config-system → tools-and-hooks
  → tui-architecture → backend-workspace → lsp-mcp-integration → data-persistence

聚焦某个方向:
  • Agent/Hooks: agent-core + tools-and-hooks
  • C/S Server: backend-workspace + architecture-overview
  • UI/TUI: tui-architecture
  • LSP/MCP: lsp-mcp-integration
  • Config/Extensions: config-system
```
