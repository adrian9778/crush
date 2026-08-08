# 05. 工具、权限与 Hooks：Agent 如何安全地操作外部世界

## 1. 工具的统一形态

所有工具最终实现 `fantasy.AgentTool`：`Info()` 暴露名称、描述和 JSON Schema，`Run(ctx, ToolCall)` 执行，`ProviderOptions/SetProviderOptions` 携带 Provider 缓存配置。多数工具通过 Fantasy 的泛型构造器创建，描述来自同名 `.md` 或 `.md.tpl` 嵌入文件。

工具 Context 传递四项运行信息：SessionID、当前 assistant MessageID、模型是否支持图片、模型名。SessionAgent 在 `PrepareStep` 创建 assistant 后写入；Agent/AgenticFetch/Permission/View 等工具依赖这些值。

## 2. `buildTools` 注册流水线

Coordinator 每次 `UpdateModels` 都重建工具：

1. 若 Agent allowlist 含 `agent` / `agentic_fetch`，先构造这两个特殊工具。
2. 创建 Hook Runner（仅当配置存在 PreToolUse）。
3. 构造全部普通内置工具。
4. 只在顶层交互 Agent 加 `question`。
5. 配置了 LSP 或 AutoLSP 未关闭时加入 LSP 工具。
6. 配置 MCP 时加入资源列表/读取工具。
7. 用 Agent `AllowedTools` 过滤所有内置工具。
8. 读取动态 MCP 工具注册表，并按 `AllowedMCP` 过滤：nil=无限制，空 map=全部禁止，server 对应空列表=允许该 server 全部工具，否则按原 MCP 工具名允许。
9. 按最终工具名排序，保证稳定。
10. 顶层 Agent 的每个工具外包一层 `hookedTool`；子 Agent 不包。

MCP 工具名称经过命名空间处理，包装器保存 MCP server/name；调用前走统一 Permission 请求，再把 JSON 输入交给 MCP Client。MCP 初始化、重连和配置 reconcile 会维护工具、prompt、resource 注册表；错误状态必须关闭旧 Session 并清空陈旧能力，重连后重新注册。`WaitForInit` 保证首轮 buildTools 不漏掉慢服务。

## 3. 内置工具分类详解

### 3.1 文件读取与搜索

| 工具 | 作用与实现要点 |
| --- | --- |
| `view` | 分段读取文本或图片；限制单行、返回内容和图片大小；支持 `crush://skills/...` 内嵌 Skill；成功读取 active Skill 时通知 Tracker；可结合 LSP/FileTracker |
| `glob` | 使用 ripgrep/文件遍历匹配路径，规范化输出并限制数量；防止符号链接逃出搜索根 |
| `grep` | 正则内容搜索，支持 include/ignore、行列位置、二进制/文本判断与结果上限；缓存 glob→regex |
| `ls` | 列目录；工作区边界之外需要权限 |
| `sourcegraph` | 调 Sourcegraph 搜索公共代码并格式化有限结果 |

### 3.2 文件修改

| 工具 | 作用与实现要点 |
| --- | --- |
| `write` | 创建/覆盖完整文件；请求写权限；写入后记录 History、FileTracker，并触发 LSP 变更 |
| `edit` | 精确替换 old_string；检测 0/多重匹配，支持 replace_all；保留 CRLF/文件元数据；有空白诊断和缩进自适应 |
| `multiedit` | 同一文件顺序应用多项 edit；后项基于前项结果；报告逐项成功/失败并允许部分成功 |
| `lsp_rename` | 由语言服务计算跨文件 WorkspaceEdit，再经权限、历史与文件跟踪执行 |
| `lsp_replace_symbol` | 定位完整 symbol 范围并替换，适合结构级修改 |

重新实现时，权限应在任何写入之前；History/FileTracker/LSP 通知应在写成功之后。MultiEdit 不能把每项都对原始文本应用。

### 3.3 Shell 与后台任务

`bash` 使用项目内嵌 shell。它做命令链检测、权限请求、工作目录控制、输出截断、attribution，以及达到阈值后自动转后台。高风险点是链式命令：必须对整个调用走权限，拒绝时不能执行任一段。

后台 Job 保存在共享注册表：

- `job_output`：列任务或增量读取 stdout/stderr、退出状态。
- `job_kill`：取消指定后台任务。

测试覆盖并发读取、多次增量、空输出、退出码、kill、stdout+stderr、自动后台化和 BlockFuncs。

### 3.4 网络工具

- `download`：下载 URL 到工作区文件，先请求权限。
- `fetch`：HTTP 获取内容，识别 HTML/JSON，清理页面噪声并转 Markdown；受权限控制。
- `agentic_fetch`：见上一章，它是带专用工具集的小模型研究代理。
- `web_search` / `web_fetch`：只给 agentic-fetch 子 Agent 的低层网络工具。

网络客户端未注入时构建有超时、连接池限制的默认 Client；调用方可注入 Client 便于测试。

### 3.5 LSP 只读工具

- `lsp_diagnostics`：诊断。
- `lsp_references`：引用。
- `lsp_definition`：定义。
- `lsp_symbols`：文档/工作区符号。
- `lsp_call_hierarchy`：调用层级。
- `lsp_restart`：重启语言服务。

公共 helper 负责把文件、行列转 LSP position、选择 Server、格式化响应。Symbol offset 的测试明确要求不能越过目标。

### 3.6 会话与诊断工具

- `todos`：读取/更新当前 Session 的任务列表。
- `question`：通过 question.Service 向人提问；只在 interactive 顶层 Agent 注册；兼容原生数组和字符串编码数组。
- `crush_info`：输出配置文件、新旧状态、模型、Provider、LSP、MCP、Skill、权限、禁用工具、选项、Attribution、Hook；不输出 Secret，排序稳定。
- `crush_logs`：读取结构化日志尾部，限制行数/单行大小，跳过坏行，脱敏敏感字段并稳定格式化。
- `list_mcp_resources` / `read_mcp_resource`：枚举或读取 MCP Resource，并按 Server/URI 请求权限。
- `agent`：通用 task 子 Agent 工具。

## 4. 权限服务

### 4.1 请求结构

工具提供 SessionID、ToolCallID、ToolName、Action、Description、Params、Path。Service 把 Path 规范到目录，创建带 UUID 的 `PermissionRequest`，通过 Broker 通知 UI，然后阻塞等待 Grant/Deny 或 Context 取消。

### 4.2 判定顺序

`Request` 的短路顺序非常重要：

1. 全局 skip/yolo → 允许。
2. `allowedTools` 包含 `tool:action` 或 `tool` → 允许。
3. Context 中有同 ToolCallID 的 Hook approval → 允许，并发布 granted 通知。
4. 获取 `requestMu`，保证同时只向用户展示一个请求。
5. 发布“请求中”通知。
6. Session 被 `AutoApproveSession` → 允许。
7. Session/Tool/Action/Path 组合已有持久授权 → 允许。
8. 创建 pending channel、发布 PermissionRequest，等待用户或取消。

Hook approval 绑定 ToolCallID，不能被同 Context 的另一次工具调用复用。

### 4.3 解决竞态

`Grant`、`GrantPersistent`、`Deny` 都调用 `resolve`。它使用 `pendingRequests.Take(id)` 原子删除，只有第一个调用者获胜；只发布一次最终通知，只向 buffered response channel 写一次。持久授权也只在 GrantPersistent 赢得竞态后记录，避免“Deny 已赢，但迟到的 persistent grant 污染未来请求”。

PermissionKey 是 SessionID+ToolName+Action+Path，因此同 Session 对某目录的某动作授权不会无意扩散到别的动作或路径。

## 5. Hook 执行协议

当前事件只有 `PreToolUse`。Hook 配置包含 command、matcher 正则、timeout 等。Runner 初始化时编译 matcher；坏正则跳过。调用时：

1. 按工具名筛选 matcher；空 matcher 匹配全部。
2. 按 command 字符串去重。
3. 构造环境变量和 stdin JSON payload，包含事件、Session、cwd/project dir、工具名和原始输入。
4. 所有匹配 Hook 并行运行，但结果按配置顺序聚合。
5. 每个 Hook 通过 Crush 内嵌 POSIX shell 执行，超时后允许 1 秒退出；仍不 yield 则放弃 goroutine，且不再读取可能仍被写的 Buffer，避免数据竞争。

退出/输出语义：

- exit 2：拒绝当前工具，stderr 是理由。
- exit 49：拒绝并终止整个 turn，stderr 是理由。
- 其他非零：记录 warning，但不阻止工具。
- exit 0：解析 stdout JSON；支持 Crush 与 Claude Code 兼容格式，可给 decision、reason、context、updated_input、halt。

聚合规则：deny > allow > none；halt 一旦出现一直为真；reason/context 按配置顺序换行拼接；updated_input 对顶层 JSON 浅合并，后 Hook 覆盖同名键。无效 patch 被忽略。

## 6. Hook 包装与权限的精确顺序

真实调用链是：

```text
模型 ToolCall
  → hookedTool.Run
    → 并行执行并聚合 PreToolUse Hooks
    → deny/halt：不调用内部工具
    → updated_input：改写 ToolCall.Input
    → allow：在 Context 标记本 ToolCallID 已批准
    → innerTool.Run
      → 工具内部 permission.Request
        → 识别 Hook approval，跳过人工询问
      → 执行副作用
    → 把 Hook context 追加到工具文本结果
    → 把 Hook 审计信息合并进 response.Metadata
```

因此 Hooks **先于权限**。Hook 静默不等于允许，仍进入普通权限流程；显式 allow 才预批准；deny/halt 连内部工具都不调用。`halt` 会设置工具响应 `StopTurn=true`，SessionAgent 在 step finish 将其视作 end-turn。

子 Agent 内部工具不包 Hook，是为了避免一次 `agent` 调用触发 N 次用户 Hook；父层的 `agent`/`agentic_fetch` 工具本身仍被包一次。

## 7. 安全边界与响应约定

工具通常把可恢复的用户输入错误转成 `fantasy.NewTextErrorResponse`，让模型读到并修正；基础设施故障才返回 Go error。权限拒绝也应是工具错误响应，不应使整个进程崩溃。媒体响应需验证 base64，非法媒体转错误结果。

路径工具必须以 workingDir 为语义根，显式处理绝对路径、`..`、符号链接和 `crush://`。View 对 `crush://` 只能查嵌入 FS，不能把它拼到磁盘路径。

## 8. 测试不变量

- Hook allow 精确批准同一次 ToolCall；silent 不批准；deny 不运行 inner。
- Hook 并行执行、按配置序聚合、命令去重、matcher 生效、超时/不 yield 不阻塞主流程。
- Permission 多个订阅者同时响应时 first-wins；persistent grant 只有赢家可写。
- Bash 链式命令始终请求权限；拒绝不执行；输出截断保持合法 UTF-8。
- View 精确处理边界、超长行、最大内容、超大图片和 builtin Skill。
- Glob 不经 symlink 逃逸且结果有上限；Grep 的不同实现结果一致。
- Edit 保留 CRLF/metadata，多匹配默认拒绝；MultiEdit 顺序应用并报告部分成功。
- MCP 错误会清空旧工具/resources/prompts；并发 renew 只进行一次；恢复后能力完整。
- crush_info 不泄密且输出确定；crush_logs 脱敏、限长、容忍坏行。

## 9. 从零复刻建议

1. 定义 AgentTool 接口与 Context keys，先实现纯只读 `view/glob/grep`。
2. 实现 Permission Service 和 first-wins 测试，再做 write/edit/bash。
3. 加 History/FileTracker/LSP 写后通知。
4. 实现 Hook Runner 的纯解析与聚合测试，再加 shell timeout，最后包装 Tool。
5. 按 `buildTools` 的“全量构造→allowlist→MCP filter→排序→Hook 包装”顺序组装。
6. 最后实现 MCP 生命周期和子 Agent 工具，因为它们同时依赖工具、权限、会话和 Provider。
