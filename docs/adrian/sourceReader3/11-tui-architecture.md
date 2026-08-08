# TUI 架构：从 Bubble Tea 主循环到屏幕布局

> 目标：让第一次接触终端 UI 的读者，能够按本文搭出 Crush TUI 的骨架。本文只讲顶层架构、状态、消息路由和布局；聊天项的渲染见 `12-tui-chat-rendering.md`，对话框、主题和性能见 `13-tui-dialog-styles-performance.md`。

## 1. 先建立正确的心智模型

Crush 的 UI 不是传统 GUI，也不是一棵由许多 Bubble Tea `Model` 递归组成的组件树。它只有一个正式的 Bubble Tea Model：`internal/ui/model.UI`。

Bubble Tea 使用 Elm 风格循环：

```text
外部事件/tea.Cmd 返回值
        ↓
UI.Update(msg) ── 修改纯 UI 状态，并返回下一批 tea.Cmd
        ↓
UI.View() ── 创建 ScreenBuffer，调用 Draw，展平为终端字符串
        ↓
终端显示；按键、鼠标、服务事件再次进入 Update
```

关键纪律：

- `Update` 是唯一的状态路由中心。
- IO、HTTP、磁盘、耗时查询不能直接放在 `Update` 中；包装成 `tea.Cmd`，完成后返回消息。
- `tea.Cmd` 不应偷偷修改 Model；它产生消息，再由下一轮 `Update` 修改状态。
- `Chat`、`List`、`Attachments`、`Completions` 等是有状态的普通结构体，不是独立 Elm 子树。顶层 UI 调用它们的 `SetSize`、`ScrollBy`、`Update` 或 `Draw`。
- 多个命令用 `tea.Batch` 并发返回；需要先后顺序时用 `tea.Sequence`。

这套集中式结构牺牲了一点 `UI.Update` 的长度，却避免了焦点、弹窗、鼠标坐标、布局尺寸分散到嵌套 Model 后难以同步。

## 2. 目录地图

| 目录 | 职责 |
|---|---|
| `internal/ui/model/` | 顶层 `UI`、布局、按键、Chat 外壳、侧栏、状态栏、会话事件 |
| `internal/ui/chat/` | 用户、助手、工具调用、shell 等聊天项 |
| `internal/ui/list/` | 按行滚动、惰性渲染、选择、缓存的通用列表 |
| `internal/ui/dialog/` | 栈式覆盖层及所有对话框 |
| `internal/ui/common/` | 共享上下文、Markdown、语法高亮、ANSI、滚动条、布局元素 |
| `internal/ui/completions/` | `@` 文件/MCP 资源补全 |
| `internal/ui/attachments/` | 输入框上方的附件 chips |
| `internal/ui/styles/` | 主题 token 到所有具体样式的映射 |
| `internal/ui/diffview/` | unified/split diff 计算和渲染 |
| `internal/ui/image/` | Kitty 图像协议与字符块降级渲染 |
| `internal/ui/notification/` | OSC、系统通知、bell、noop 后端 |
| `internal/ui/anim/` | 渐变动画和 spinner |
| `internal/ui/logo/` | 大小 Logo |
| `internal/ui/xchroma/` | Chroma lexer 查找缓存 |

共享依赖 `common.Common` 只持有 `Workspace` 和 `*styles.Styles`。所有需要应用数据或样式的组件都接收它或其中的样式指针，避免全局状态。

## 3. UI 的状态分两条轴

### 3.1 页面状态 `uiState`

`internal/ui/model/ui.go` 定义四种页面：

1. `uiOnboarding`：没有完成模型配置，通常立即打开模型选择。
2. `uiInitialize`：项目目录需要初始化，询问是否创建项目上下文。
3. `uiLanding`：已配置但尚未进入会话；显示落地页和编辑器。
4. `uiChat`：完整聊天页。

构造时：配置未完成进入 onboarding；项目需初始化进入 initialize；其余进入 landing。加载或创建 Session 后进入 chat。`setState` 同时设置焦点并重新计算布局；回到 landing 会关闭 compact 状态。

### 3.2 焦点状态 `uiFocusState`

- `uiFocusNone`：没有可交互区域。
- `uiFocusEditor`：textarea 或 inline question editor 接收键盘。
- `uiFocusMain`：聊天列表接收选择、滚动、展开和复制命令。
- `uiFocusSidebar`：可滚动侧栏接收导航。

焦点不是视觉装饰：它决定按键交给谁、消息项是否显示 focused 前缀、鼠标点击如何转换坐标，以及 `Draw` 最终返回哪个光标。切换时要成对调用 textarea/chat 的 `Focus`、`Blur`。

## 4. `UI` 里保存了什么

`UI` 很大，但可按职责分组理解：

- 应用：`com`、当前 `session`、修改文件、初始 Session 参数。
- 几何：终端 `width/height`、`uiLayout`、compact/details 状态。
- 状态机：`state`、`focus`、key map、keyboard enhancements。
- 主组件：`chat`、`textarea`、`dialog`、`status`、`header`、`attachments`、`completions`。
- 输入模式：普通、YOLO、bang shell、外部编辑器、inline question editor。
- 会话展示：todos、prompt queue、pills、文件、LSP、MCP、skills。
- 缓存：workspace busy/yolo/model、LSP 状态、侧栏字符串、chat draw cache。
- 交互：鼠标拖选、点击时间、补全起点、历史草稿。
- 终端集成：能力探测、透明背景、进度条、通知后端。

新实现不要一次复制全部字段。先实现页面状态、焦点、尺寸、textarea、Chat、Overlay；随后按功能逐组加入。

## 5. 构造和启动

`New` 的顺序很值得照搬：

1. 创建 textarea，设置样式、无限字符、动态高度、最小 3/最大 15 行并聚焦。
2. 从配置读取 scrollbar 模式，创建 `Chat`。
3. 创建 key map、completions、todo spinner。
4. 创建 attachments，并注入删除按键和全部 attachment 样式。
5. 创建 header、Overlay、各状态 map 和默认 noop notification。
6. 从 Workspace 同步读取一次 YOLO、agent ready/model 作为首帧种子；后续读走异步缓存。
7. 设置 prompt、随机 placeholder、Status、compact 配置和初始页面。

`Init` 返回一批启动命令：加载自定义命令/skills、加载提示历史、刷新 LSP、加载指定或最近 Session、读取 Hyper credits、刷新 busy 状态、检查待认证 MCP。它没有直接执行 IO。

## 6. Update：按优先级路由消息

不要把 `Update` 看成一个杂乱 switch；它实际按以下层次工作。

### 6.1 终端环境

`tea.EnvMsg`、颜色、窗口大小、像素大小、终端版本、设备属性、Kitty 响应、OSC 响应更新 `Capabilities`。窗口 resize 会：记录尺寸、启动 Chat resize 阶段、重算布局。焦点/失焦用于决定是否发桌面通知。

### 6.2 Workspace/pubsub 领域事件

UI 订阅的关键事件包括：

- `pubsub.Event[session.Session]`：Session 删除则新建；当前 Session 更新则刷新 todos/pills 和 spinner。
- `pubsub.Event[message.Message]`：Created 追加，Updated 原地更新 streaming item，Deleted 删除；其他 Session 的消息尝试解释为 agent 子会话。
- `pubsub.Event[history.File]`：更新修改文件并启动有关 LSP。
- `app.LSPEvent` / `workspace.LSPEvent`：请求异步刷新 LSP 状态。
- `skills.Event`：刷新 skill 状态和命令目录。
- `mcp.Event`：刷新 state、prompts、tools 或 resources；需要认证时自动开窗。
- `permission.PermissionRequest/Notification`：打开权限窗或显示处理结果。
- `question.Request/Notification`：打开 inline/batch question，或处理取消通知。
- `workspace.ConnectionEvent`：断线显示警告；恢复后重载当前 Session，因为断线期间事件不可补发。

事件处理中有一个重要性能策略：message `UpdatedEvent` 可能每个 token 到来一次，所以不能每次同步查询 Workspace；busy 和 queue 只在回合边界失效，并通过 TTL + generation 的异步缓存刷新。

### 6.3 对话框优先拦截

鼠标、键盘、spinner 等消息只要有 dialog，就优先交给 Overlay 的最上层 Dialog。对话框返回的不是直接副作用，而是 `Action`，随后 `UI.handleDialogMsg` 将 `ActionSelectModel`、`ActionPermissionResponse`、`ActionRunCustomCommand` 等翻译为真正的 Workspace 调用和 UI 状态改变。

### 6.4 输入组件优先级

没有对话框时，路由大致是：

1. active inline editor（问题表单）先处理。
2. 补全窗口开启时先消费上下选择、确认、关闭。
3. attachments 的删除模式可消费数字、Escape 等。
4. 按 focus 将按键交给 editor、chat 或 sidebar。
5. 全局按键处理模型选择、Session、帮助、compact、quit 等。

鼠标点击先算是否落在 attachment 删除按钮，再按矩形切换焦点；chat 坐标需减去 `layout.main.Min`。拖动实现跨消息文本选择，双击选词、三击选行。

## 7. 消息如何变成聊天项

`setSessionMessages` 是全量恢复路径：先构造所有 ToolResult map，再把每条 Message 展开成多个 `chat.MessageItem`，恢复嵌套 agent tools，最后一次设置 Chat。

实时路径分三类：

- User Created：生成 User item；shell command 已在 bang UI 中实时显示，持久化副本跳过。
- Assistant Created/Updated：一个 Assistant item 加零到多个 Tool item；流更新尽量调用 item mutator，保留对象身份和缓存。
- Tool Created：通过 `ToolCallID` 找到对应 Tool item，`SetResult` 更新状态。

Assistant 完成且 `FinishReasonEndTurn` 时追加稳定的 AssistantInfo item，显示模型、provider 和耗时。

Agent 工具有子 Session。UI 从子 Session ID 解析父 tool-call ID，将子消息里的 tool calls/results 转换为 compact nested tool items，并递归支持 agent 内再调用 agent。

## 8. 布局与绘制管线

### 8.1 `uiLayout` 是矩形集合

它保存 `area/header/main/pills/editor/sidebar/status/sessionDetails`。`generateLayout(w,h)` 使用 rectangle layout 切分，不靠拼接大字符串猜位置。

公共外框先预留底部 help/status，应用区域四边留白。不同页面：

- onboarding/initialize：4 行 header + main + status。
- landing：header + main + 动态 editor + status。
- chat 宽屏：左侧 main/editor，右侧固定约 32 列 sidebar，底部 status。
- chat compact：1 行 header + main/pills + editor + status；details 是覆盖层。

编辑器高度为 textarea 动态高度 + 2 行（附件和间距）。inline question 存在时改用其 `Height(width)`。pills 从 main 底部切出空间。终端小于 breakpoint 或用户强制时进入 compact。

### 8.2 为什么尺寸要算两遍

`updateLayoutAndSize`：先依据当前 textarea 高度生成 layout，再调用 `textarea.SetWidth`。宽度变化可能导致软换行，使 textarea 高度变化，因此检测后重新 layout + size 一次。这一 reconciliation 是避免 resize 抖动的关键。

### 8.3 Hybrid Rendering

`View` 创建 `uv.NewScreenBuffer(width,height)`；`Draw` 把各组件画入矩形；`canvas.Render()` 最终展平成字符串。子组件可先生成字符串，再用 `uv.NewStyledString(str).Draw(scr, rect)` 画入 ScreenBuffer。

绘制顺序：页面主体 → status/help → completions 浮层 → debug 标记 → dialogs。Dialog 必须最后画才能覆盖一切。`Draw` 返回 textarea/inline/dialog 的硬件 cursor；`View` 还设置 alt screen、背景、鼠标模式、窗口标题、focus report 和 Windows Terminal progress bar。

## 9. 侧栏、header、status 和 pills

- `header.go`：compact/landing 顶栏，展示 Logo、目录、上下文、credits 和 details 提示。
- `sidebar.go`：宽屏 Session 标题、工作目录、模型、files、LSP、MCP、skills；预渲染字符串并维护独立 offset/scrollbar。
- `status.go`：短/完整 key help 与有 TTL 的 info/warn/error/success。
- `pills.go`：todo 和 prompt queue 的摘要/展开列表；高度改变必须重算 chat rectangle。
- `workspace_cache.go`：异步缓存 busy、queue、model、YOLO；generation 防止旧 HTTP 响应覆盖新状态。

## 10. 推荐复刻顺序

1. 实现只有 `UI{width,height,state,focus}` 的 Bubble Tea Model 和空 ScreenBuffer。
2. 加 `uiLayout`、四页面矩形和 resize 双遍 reconciliation。
3. 加 textarea、focus 和最小按键集。
4. 加 `list.List` 与 `Chat`，先只支持纯文本 user/assistant。
5. 接 Session/Message pubsub，严格区分 created/updated/deleted。
6. 加 Tool item factory 和 tool result 关联。
7. 加 Overlay/Dialog Action 协议。
8. 加 completions、attachments、问题 inline editor。
9. 加 sidebar/header/status/pills 和 Workspace 异步缓存。
10. 最后加入主题切换、ANSI、图像、通知和性能缓存。

## 11. 关键测试应该告诉你什么

- `model/ui_test.go`：状态、消息路由、帮助、通知选择等顶层行为。
- `model/layout_test.go`：小终端、动态 editor、compact 几何不越界。
- `model/session_test.go`、`session_busy_test.go`：恢复 Session 与 busy/queue 一致性。
- `model/chat_draw_cache_test.go`：Chat 屏幕缓存命中/失效。
- `model/permission_test.go`：异步权限窗不会误吞/误用按键。
- `model/filter_test.go`：高频鼠标 wheel/motion 合并，避免消息洪水。

复刻时应先让这些行为测试成立，再追求视觉完全一致。
