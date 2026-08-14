# 聊天与工具渲染：从 Message 到终端像素

> 状态：待复核生成稿｜生成日期：2026-08-14
> 基准提交：`5712d4839a6a10e9940804d511bb322dbe73a511`｜工作区：clean（开始分析时）
> 源码范围：`internal/ui/chat/`、`list/`、`model/chat.go`、`completions/`、`attachments/`
> 生成方式：源码、渲染测试与 golden 资产静态分析

## 快速摘要

### 架构总览（模块与依赖）
领域 Message 经 UI factory 变成 MessageItem；Chat 包装按行 viewport 的 List；各工具 renderer、Markdown、diff 和附件组件生成最终终端字符串。

### 核心调用序列（逐步逻辑）
1. Workspace 事件更新 Message。2. UI 创建或替换 Item。3. List 计算可见范围。4. Item 使用缓存渲染。5. Chat 把字符串画入 ScreenBuffer。

### 易错点与边界条件
流式 Markdown 需稳定前缀；工具 factory 必须有 fallback；ANSI 选择和截断不能按字节；TotalHeight 不应进入每帧 resize 热路径。

## 1. 数据流总览

```text
message.Message
  ├─ User ─────────────→ UserMessageItem
  ├─ Assistant content → AssistantMessageItem
  ├─ Assistant tools ──→ ToolMessageItem × N
  └─ Tool results ─────→ 按 ToolCallID 更新既有 ToolMessageItem
                              ↓
                         model.Chat
                              ↓
                           list.List
                              ↓
                    visible string → ScreenBuffer
```

一条 Assistant Message 可能对应多个 UI item。UI ID 索引使 streaming 更新不必重建整张列表。

## 2. List 的契约和实现

`list.Item` 必须实现：

- `Render(width) string`：在给定宽度下生成内容。
- `Version() uint64`：任何会改变渲染的 mutation 都必须 `Bump()`。
- `Finished() bool`：流式/动画中为 false，永久稳定后为 true。

可选能力：

- `RawRenderable`：不带 list/focus 前缀的渲染，复制和高亮要用。
- `Focusable`：选中变化时改变样式。
- `Highlightable`：保存跨行选择范围。
- `MouseClickable`：item 内部点击。
- chat 层再加 `Identifiable`、`Animatable`、`Expandable`、`KeyEventHandler`。

`Versioned` 是内嵌的单调计数器。最常见 bug 是字段变了却没 `Bump`，导致列表继续展示旧 cache。

### 2.1 行级 viewport

List 用 `(offsetIdx, offsetLine)` 表示首个可见 item 和该 item 被裁掉的行数，而不是单一像素/行 offset。这样超长消息也能局部滚动。`ScrollBy` 穿过 item 边界并计入 gap，最终夹到顶部或底部。

`Render` 只遍历可见 item，而且每次 append 都受剩余 viewport 行预算限制。因此超长 item 不会把数万行临时塞入输出。

### 2.2 两级缓存

List cache key 是 item 指针 + width + version。entry 保存完整 string、预切分 lines、height、frozen 标志。render callback 仍会先运行，以便 focus/highlight 改变并 bump version。

当 `Finished()==true`，entry 可冻结，后续连 `Render` 都不调用。拖拽选择期间对范围内 item 使用 `freezeSuppressed`，否则 frozen 内容无法显示实时选区。

`TotalHeight` 会渲染所有 item，只应用于精确滚动条；`Overflows` 从底部累计，超过 viewport 立即停止；resize 后 `Prewarm` 分批填缓存，避免单帧卡顿。

## 3. model.Chat：List 的业务外壳

`model.Chat` 增加：

- `id → index` map，快速找 message/tool item。
- `follow`：用户在底部时随 streaming 自动滚动；主动向上滚动会取消。
- selected item、展开、按键分发。
- 鼠标 down/drag/up、双击选词、三击选行、跨 item 高亮和复制。
- animatable item 的启动、暂停和 tick 路由，包括 nested tools。
- scrollbar 显隐定时器。
- resize settle、增量 cache warm。
- 最终字符串到 `uv.Screen` 的 draw cache。

`SetMessages` 用于全量恢复；`AppendMessages` 保持对象身份；`RemoveMessage` 同步修复索引和选择。主题切换时 `InvalidateRenderCaches` 必须同时清 item cache、list cache、screen cache。

## 4. MessageItem 的组合方式

`chat/messages.go` 通过嵌入而非继承复用行为：

- `cachedMessageItem`：按 width 缓存 RawRender，并另存带逐行前缀的 render。
- `focusableMessageItem`：focused 变化 bump version。
- `highlightableMessageItem`：选择范围、highlighter 和 bump。
- `list.Versioned`：列表缓存的失效协议。

内容最大宽度通常限制为 120 列，左侧 border/padding 总计 2 列；Edit/MultiEdit diff 等需要全宽。使用 ANSI 感知的宽度函数，不能用 `len`。

## 5. User 与 Shell

`UserMessageItem` 渲染用户文本和只读 attachments。已发送附件不显示删除按钮；技能、图像、文本分别使用不同图标。用户块支持 focus、高亮和缓存。

bang 模式的 shell 不走普通 ToolCall：`ShellItem` 先以 pending 状态加入 Chat，命令在 `tea.Cmd` 中流式执行，`shellStreamMsg` 追加输出，`shellResultMsg` 固化 exit code，并将 ShellCommand 持久化。恢复 Session 时跳过这条持久化 user item 的重复展示。

原始 shell 输出可能包含 ANSI、回车刷新和光标控制。`common.RemapANSI16` 把 16 色映射到当前主题；`StripCursorControl` 去除危险控制；模拟 carriage return 后再按终端宽度处理。

## 6. Assistant 的三段式渲染

Assistant item 分为 thinking、content、error/refusal 三段，各有独立 `assistantSection` cache。流式 content 变化不会迫使昂贵的 thinking 重渲染，反之亦然。

### 6.1 Thinking 状态机

- collapsed：最多显示 10 行。
- tail-window：长 thinking 展开时只显示最后 200 个“Glamour 渲染后”的行，并提示更早内容被隐藏。
- full-expanded：完整展示。

短 thinking 跳过 tail-window，形成两态切换。对渲染后的结果裁切避免从 Markdown fenced code/list/table 中间撕裂。完成时显示思考耗时；仍在生成时运行渐变动画和 turn timer。

### 6.2 错误和拒绝

普通 error、取消、provider content filter 都有不同视觉。Content filter 若持久层只有 finish reason，TUI 填充统一的拒绝标题和说明，保证实时与恢复 Session 一致。

### 6.3 Streaming Markdown 的稳定前缀算法

完整 Markdown 每个 token 都重新走 Goldmark/Glamour，成本会随文本长度增长。`streamingMarkdown` 缓存已确定不再受后文语义影响的稳定前缀：

1. 只在空行之后寻找候选边界。
2. 边界前不能有未闭合 fenced code。
3. 避开 list continuation、table、blockquote、HTML block、link reference、setext heading 等会跨边界解释的结构。
4. 新 content 必须以旧 stablePrefix 开头，宽度也必须相同；否则 reset 并全量 render。
5. 新边界出现时只 render 新的稳定 chunk，再 render 尾部，将两段按一个空行拼合。
6. 缓存 fence parity 和是否出现 list marker，使后续扫描只处理 delta，而非重新扫描全文。

这是保守优化：有任何不确定就全量渲染。Glamour renderer 内部有状态且非线程安全；生产 UI 单线程，但并行测试需通过 `common.LockMarkdownRenderer` 序列化。

相关测试：`incremental_glamour_test.go` 覆盖等价性，`assistant_section_cache_test.go` 覆盖分段失效，`streaming_thinking_bench_test.go` 和 resize benchmark 防性能回退。

## 7. ToolMessageItem 的状态机

`ToolStatus`：awaiting permission、running、success、error、canceled。`baseToolMessageItem` 保存 ToolCall、Result、messageID、renderer、animation、expanded、compact 和 cache。

`ToolRenderOpts` 将状态打包给具体 renderer。Header 通常包含状态图标、动作和主参数；body 按 pending/result/error/permission denied 分支。默认结果最多显示约 10 行，展开后显示全部。

`SetToolCall`、`SetResult`、`SetStatus`、`SetCompact` 必须清相关 cache 并 bump。运行中的 item `Finished=false`；成功/失败/取消且无动画后才可冻结。

## 8. Tool factory 完整路由

`NewToolMessageItem` 是唯一工厂，应首先按 tool name 选择 renderer，最后写入父 `messageID`：

| 类别 | 文件 | 展示重点 |
|---|---|---|
| Bash / JobOutput / JobKill | `bash.go` | command、description、job/shell ID、输出、exit/error |
| View / Write / Edit / MultiEdit / Download | `file.go` | 路径、行范围、代码、统一 diff、多编辑 note、下载目标 |
| Glob / Grep / LS / Sourcegraph | `search.go` | pattern/path/query、结果列表及截断 |
| Fetch / WebFetch / WebSearch | `fetch.go` | URL/query、内容或搜索结果 |
| Agent / AgenticFetch | `agent.go` | 任务 prompt、子 Session 的 compact nested tool 树 |
| Diagnostics | `diagnostics.go` | 诊断级别、位置、消息 |
| Todos | `todos.go` | 完成比例、每项状态、刚开始任务 |
| Question | `question.go` | 问题、选项和回答摘要 |
| References | `references.go` | 符号引用列表 |
| Definition | `definition.go` | 定义位置 |
| Rename | `rename.go` | 旧/新名称和改动结果 |
| ReplaceSymbol | `replace_symbol.go` | symbol 与替换内容 |
| CallHierarchy | `call_hierarchy.go` | caller/callee 层级 |
| Symbols | `symbols.go` | workspace/document symbols |
| LSPRestart | `lsp_restart.go` | server 和重启状态 |
| MCP | `mcp.go` | 从 `mcp_server_tool` 拆 server/tool，展示 JSON 输入输出 |
| Docker MCP | `docker_mcp.go` | Docker MCP 特殊 discovery/enable/disable 结果 |
| Generic | `generic.go` | 未识别工具的名称、JSON 参数、通用 result |

匹配优先级很重要：Docker MCP 特判先于普通 `mcp_` 前缀；未知项必须回退 Generic，不能让新工具导致 UI 崩溃。

## 9. Tool body 的共享规则

`tools.go` 集中了 header、状态图标、JSON 参数、错误、hook 信息、nested tree 等公共拼装。`tool_result_content.go` 统一处理文本结果：按 inner width 加背景/缩进，截断时显示剩余行数。`unified_diff.go` 解析工具返回的 patch；`common.DiffFormatter` 注入当前主题的 Diff 样式和 Chroma style。

所有 renderer 都应：

- 对输入 JSON 解码失败做容错，退回可读原文。
- 使用 `ansi.StringWidth/Truncate/Cut` 计算终端列。
- 先扣 border/padding 得到 content width。
- pending 显示动画，空结果显示统一 placeholder。
- permission/hook deny 不误标成普通工具运行错误。
- compact nested 模式只显示单行关键信息。

## 10. Diffview

`diffview.DiffView` 是 builder：设置 before/after/file name、Unified/Split、width/height、context lines、line numbers、x/y offset、tab width、theme 和 ChromaStyle，最后 `String()`。

流水线：规范换行 → tab 展开 → `udiff` 计算 hunks → split 对齐（如需要）→ 检测行号位数和代码宽度 → 应用视口 offset → 语法高亮 → 逐行背景与行号。

大量 golden 以 width 1..110、height、x/y offset、dark/light、窄屏、多 hunk、无行号/无高亮覆盖布局边界。复刻不要手写少量示例代替这些 golden，它们本质是终端几何规范。

## 11. Completions

输入 `@` 时 UI 记录 textarea 索引和屏幕起点，`Completions.Open` 用一个 Cmd 并行读取文件和 MCP resources。完成后合并为 `FilterableList`：

- fuzzy filter；额外按 basename/路径 segment 优先级排序。
- 上下键选择，Tab/Enter 确认，Escape 关闭。
- 选择文件后读文件，检查 5 MiB，生成 attachment。
- MCP resource 则异步读取资源内容，再生成 attachment/插入内容。
- popup 向 editor 上方绘制，右边界超屏时左移。

`FilterableList` 保留原始 items，查询变化重算 fuzzy match 并把 match positions 交给 item 高亮。中文/多字节字符串需将 byte position 转成可见字符位置。

## 12. Attachments

Attachments 接收 `message.Attachment` 消息并追加。编辑状态 chip 由 icon、截断 basename、删除按钮组成；进入删除模式后按钮位置显示数字，避免布局跳动。鼠标 hit-test 使用最近一次 Render 记录的 remove X 范围。

文件上限为 5 MiB；允许图像扩展名 jpg/jpeg/png。模型不支持 image 时不应开放图片入口。粘贴大段文本可写为临时 paste attachment；粘贴路径/剪贴板图片也走 attachment 消息。

## 13. 文本选择

鼠标 down 记录起始 item/行/列；drag 计算规范化范围，并通过 list render callback 为每个受影响 item 设置局部 highlight；up 结束 freeze suppression。双击用 rune/word 边界选词，三击选整行。复制调用每项 `RawRender`，按选择矩形裁切后拼接。

选区坐标一定是“终端列”，不是字节下标。ANSI escape、宽字符、组合字符都要求 x/ansi 与 Ultraviolet cell buffer 参与计算。

## 14. 从零复刻的验收顺序

1. 实现 Versioned Item + 可见行 List。
2. 实现 Chat ID map、follow、scroll/selection。
3. 实现 User/Assistant 纯文本和 full-session 恢复。
4. 实现 Created/Updated streaming，确认 item 指针不替换。
5. 加 Markdown，再加分段 cache 和稳定前缀 cache。
6. 实现 base Tool、Generic，再逐个专用 renderer。
7. 实现 ToolResult 按 ID 回填和 nested agent tools。
8. 加鼠标高亮、展开、compact。
9. 加 completions/attachments/shell/diff/image。
10. 跑 list、chat、diffview 全部单测与 golden，最后跑 benchmarks 对比分配和耗时。
