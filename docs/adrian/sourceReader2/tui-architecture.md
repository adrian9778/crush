# 六、TUI（终端用户界面）架构深度解析

## 6.1 TUI 整体概述

Crush 的 UI 层由 Charm 自研的两个核心库驱动：

- **Bubble Tea v2**：Elm 架构的终端 UI 框架，提供消息循环、命令系统和子进程集成
- **Ultraviolet (uv)**：高性能屏幕缓冲区渲染引擎，支持直接绘制带 ANSI 样式的文本到屏幕矩形区域

### 混合渲染管线

Crush 不是纯粹的 string-based TUI，也不是纯 screen buffer — 而是两者的混合体：

| 渲染方式 | 组件 | 说明 |
|----------|------|------|
| **Screen-based (Ultraviolet)** | UI 顶层所有模块 | `uv.NewStyledString(str).Draw(scr, rect)` — 直接绘制到屏幕缓冲区，矩形区域布局由 `uiLayout` struct 管理（header/main/sidebar/editor/status/pills） |
| **String-based** | Chat、List、Autocomplete 等子组件 | 渲染为 string，再粘贴到 screen buffer 上作为覆盖层 |

最终 View() 方法：创建 buffer → 调用所有组件的 Draw() → `canvas.Render()` 展平为字符串返回给 Bubble Tea。

## 6.2 顶层模型 — 唯一 Bubble Tea Model

Crush 架构的核心设计原则：**UI model 是整个应用唯一的 Bubble Tea model**。所有子组件（Chat、Completions、Attachment 等）**不参与**标准 Elm 消息循环，它们是有状态的结构体，通过命令式方法被主模型直接操纵：

- **Chat / List**：没有 Update 方法。主模型直接调用 HandleMouseDown()、ScrollBy()、SetMessages()、Animate()
- **Attachments / Completions**：非标准的 Update 签名（返回 bool 表示"是否消费了消息"），充当守卫而非完整 model
- **Sidebar**：不是独立的 model，是 UI.go 中的 drawSidebar() 方法

### UI struct 核心字段

```go
type UI struct {
    width, height int            // Terminal dimensions (updates on tea.WindowSizeMsg)
    
    // Layout computed each tick
    layout uiLayout               // Rectangle areas: Header/Main/Sidebar/Editor/Pills/Status
    
    // Core state machines
    state      uiState           // Transitions: uiOnboarding → uiInitialize → uiLanding → uiChat  
    focus      uiFocusState      // uiFocusNone | uiFocusEditor | uiFocusMain
    
    // Key sub-components
    chat     *Chat                 // Wraps list.List for message view
    textarea textarea.Model        // Input editor
    dialog   *dialog.Overlay       // Overlay dialog stack (permissions, quit, etc.)
    
    // Non-standard Update signature components  
    completions  component         // Autocomplete popup (filterable list)  
    attachments  component         // File attachment management  
    
    // Other sub-components: sidebar via drawSidebar() method on UI itself!
}
```

### Focus 状态机

```
uiFocusNone (onboarding / landing screen)
      │ focusEditor (user starts typing in textarea)
      ▼
uiFocusEditor ← User types here, Enter key → sendMessage()
      │ submit → back to...
      ▼ 
uiFocusMain ← User pressed Tab — focus moves from editor up into message view for scrolling/navigation. Escape returns to editor!
```

## 6.3 Message Routing — Giant Switch Pattern

所有消息通过主模型的 `Update()` 中的巨大 switch 分发。没有子组件的消息系统：

| Message Type | Source | Action |
|-------------|--------|--------|
| `tea.WindowSizeMsg` | Bubble Tea core | Recalculate layout rectangles |
| `tea.KeyMsg` | Keyboard event | Route: textarea (if focus==Editor) or chat list |
| `chat.UpdateScroll(int)` | Mouse wheel on chat area | Call internal Chat scroll methods directly  
| `UpdateAvailableMsg` | app.CheckForUpdates() goroutine | Show version update notification dialog
| `notify.Notification` | Agent run completion → pubsub broker | Show toasts / tool results / permission prompts 
| `tea.Cmd` return values | Sub-component imperative method calls during Update() | Trigger side effects like starting new agent turn |

### Chat 渲染管线

Chat 封装了 Bubble Tea's `list.List`：
- ID-to-index map（按 ID 查找更新特定消息）
- Mouse tracking (drag, double/triple click)
- Animation management  
- Follow flag (auto-scroll to bottom)

```go
func (m *Chat) Draw(scr uv.Screen, area uv.Rectangle) {
    // Chat internally wraps list.List with its own state; renders the list into a string which is then painted onto screen buffer!
    uv.NewStyledString(m.list.Render()).Draw(scr, area) 
}
```

### Layout Rectangle Fields（uiLayout）

| 字段 | 用途 |
|------|------|
| `Header` | 顶部栏——显示 session name + model pills |  
| `Status` | 底部状态栏（连接状态、spinner 等） |
| `Sidebar` | 左侧 workspace + session 列表面板  
| `Main` | 中心聊天区域 |
| `Editor` | 底部文本输入框  
| `Pills` | model/agent 选择 pills  

## 6.4 Chat 消息系统

### Interface 分层继承

```go
// Base (from list package)
type Item interface { Render(width int) string }

// MessageItem extends base + adds identification & content access  
type MessageItem interface {
    Item + RawRenderable + Identifiable      // Can be found by ID later for scroll updates
    Content() []message.Part                 // Actual message content
}

// ToolMessageItem — specific tool messages (bash/edit/grep etc) 
type ToolMessageItem interface {
    MessageItem
    GetToolName() string                       // e.g., "view", "bash"
    GetStatusText() string                     // "running" / "completed" / "error"  
}
```

可选能力通过嵌入实现：`Focusable`, `Highlightable`, `Expandable`, `Animatable`, `Compactable`, `KeyEventHandler`

### Tool 渲染器映射表

| 源文件 | 渲染的工具 | 说明 |
|--------|-----------|------|
| `chat/bash.go` | Bash, JobOutput, JobKill | 终端输出 + 语法高亮代码块 |
| `chat/file.go` | View, Edit, MultiEdit, Download | 显示 diff / file preview inline |
| `chat/search.go` | Glob, Grep, LS, Sourcegraph | 搜索结果列表展示 |  
| `chat/fetch.go` | Fetch, WebSearch | URL 抓取结果与格式化 |
| `chat/agent.go` | Agent, AgenticFetch | Agent-to-agent 通信结果 |
| `chat/diagnostics.go` | Diagnostics | LSP diagnostics 可视化 |
| `chat/lsp_restart.go` | LSPRestart | LSP restart state display  
| `chat/mcp.go` | MCP tools (mcp_ prefix) | MCP server tool call results inline |
| `chat/todos.md` | Todos | Structured task list rendering |
| `chat/assistant.go` | Assistant messages | thinking blocks + content blocks |
| `chat/user.go` | User messages | User input with attachments display  
| `chat/generic.go` | Fallback for unknown tools | 默认回退渲染器  

### Central Factory: `NewToolMessageItem(chat/tools.go)`

根据工具名路由到具体的渲染类型——这是所有消息渲染的统一入口。

### 渲染性能要点

1. **缓存机制**：每个 chat item 内部缓存渲染输出，数据变化时失效（`cachedMessageItem` in `chat/messages.go`）
2. **语法高亮懒加载 + memoization**：Chroma style 从主题构建是昂贵的操作（需要解析 lexer），通过 `common.ChromaStyle` memoize。lexer 查找在 `xchroma.MatchLexer` 中 memoized，永远不要在渲染路径直接调用 chroma.MustNewStyle / lexers.Match 
3. **list.TotalHeight**：渲染每个 item——只用于精确 scrollbar 几何。用 bounded list.Overflows 检查 overflow（比 TotalHeight 更轻量且可预算）。Resize mid-drag 时 chat 抑制 scrollbar，在 settle 时增量预热缓存（list.Prewarm）
4. **lazy rendering at list level**：list.List 只渲染可见 items——没有全局渲染缓存。如果渲染耗时应在 item 层内部 memoize

## 6.5 Dialog 系统 (`dialog/`)

Dialogs draw last and overlay everything else in the screen hierarchy.

### Dialog Interface

```go  
type Dialog interface {
    ID() string                                   // Unique identifier  
    HandleMsg(msg tea.Msg) Action                 // Actions: none | dismiss | confirm | waitingForUserInput / show input field for text entry 
    Draw(scr *uv.Screen, area uv.Rect)            // Full drawing onto screen buffer — always runs last!  
}
```

常用的 dialog 类型：permission approval、API key picker、model/session/commands picker、quit confirmation、file picker、reasoning level selector

### Overlay——栈式管理

```go  
type Overlay struct {
    stack []Dialog   // Push/Pop/Contains operations 
}

func (o *Overlay) Push(d Dialog) { o.stack = append(o.stack, d) }  // Top most dialog receives input events 
func (o *Overlay) Pop() { ... }                                    // Remove top-most — next one down becomes active again 
```

### Dialog 渲染规则（防止反复出现的 bug）

这些是防 wrapping/overflow bug 的必须遵循的规则：

1. **尺寸用 content area，不用 outer width**: `innerWidth = m.width - t.Dialog.GetHorizontalFrameSize()`。全屏 width 会多出一两列导致经典 "last few chars wrap" bug
2. **Padding 而非 Margin**: Margin 在边框外面（推过 frame 边界）；padding 在内部对每行生效  
3. **分段渲染 styled text segments**：逐个渲染样式段落 `styleA.Render(x) + styleB.Render(y)` ——不要拼接 raw strings 后包整体样式。内部的 reset code 会丢弃外部颜色
4. **使用共享 helpers**: `renderDialogHelp()` (keybind hints)、`dialogInputTextWidth()` (text inputs)、`common.DialogTitle` (truncates titles without wrapping)、`joinScrollbar`、`applyInfoColumnVisibility`
5. **Clamp width/height to drawable area**：`max(0, min(maxW, area.Dx()- frame))` 确保小终端里 dialog 始终在可视区域内

## 6.6 Styling 系统 (`styles/`)

### 三层架构设计

```
Layer 1: quickstyle.go (稳定基色)  
    quickStyle(opts quickStyleOpts) → Styles struct  
    - Pure token-driven design tokens (primary, secondary, fgBase, bgBase, success, error)    
    - Never hardcode specific charmtone colors except for Chroma syntax highlighting  
       
Layer 2: themes.go (具体主题实现)        
    CharmtonePantera() → Styles {  
        s := quickStyle(palette)  
        // Only override differing colors from token defaults
        s.Editor.PromptBangIconFocused.   = ...Salt...Hazy      // Specific theme overrides 
        return s 
    }  
    
    ThemeForProvider(provider string) → Styles  // Get appropriate theme by provider 
    
Layer 3: styles.go (定义整体结构体 schema)  
    type Styles struct {   
        Header, Pills, Dialog, Help Editor, Notification, ... // Nested groups defining what quickStyle produces — shape of style system
    }  
```

### Shared Context — common.Common

`common.Common` holds `*app.App` and `*styles.Styles`。线程化穿到所有需要 app 状态或样式访问的组件中。每个新组件必须接收并存储它。
