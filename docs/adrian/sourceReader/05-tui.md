# 05. TUI 源码导览

## 1. 技术栈与核心约束

TUI 使用 Bubble Tea v2 处理消息循环，Lip Gloss v2 处理样式，Ultraviolet 处理二维屏幕缓冲。它采用混合渲染：顶层用 rectangle 向 ScreenBuffer 绘制，列表/Markdown 等子组件先生成字符串，再画入指定区域。

整个 UI 只有一个标准 Bubble Tea Model：`internal/ui/model.UI`。Chat、List、Attachments、Completions 等不是嵌套 Elm Model；它们提供命令式状态方法，顶层 `UI.Update` 统一路由消息。这是阅读和扩展 UI 的首要不变量。

## 2. `UI` 顶层状态

`model/ui.go` 的 `UI` 保存：终端宽高、`uiLayout` 区域、页面状态、焦点、Workspace、Chat、textarea、Dialog Overlay、Completions、Attachments、键位、通知和各种临时状态。

页面状态大致为：

```text
uiOnboarding -> uiInitialize -> uiLanding -> uiChat
```

- Onboarding：首次启动、Provider/API key 等引导；
- Initialize：项目上下文初始化；
- Landing：无活动 Session 的欢迎页；
- Chat：当前 Session 的消息与编辑器。

焦点独立于页面状态：`uiFocusEditor` 把键盘送入 textarea，`uiFocusMain` 控制 Chat 滚动/选择，`uiFocusNone` 用于 dialog 或过渡。

## 3. Update 消息路由

`UI.Update` 很长，但可按优先级阅读：

1. Dialog Overlay 最先拦截消息；
2. 窗口尺寸、焦点、鼠标等全局消息；
3. Workspace 领域事件：Session、Message、Agent、Permission、Question、LSP、MCP、Skills；
4. 异步命令的完成消息；
5. 根据 UI state 和 focus 路由键盘；
6. 触发 Chat 动画、textarea、completions 或提交；
7. 返回合并后的 `tea.Cmd`。

`tea.Cmd` 只执行 IO/耗时工作并返回消息，不应在命令闭包里直接改 UI state。真正的状态修改回到 `Update` 完成。

## 4. 渲染管线

```text
View
 -> 创建 uv.ScreenBuffer
 -> 计算/读取 uiLayout
 -> Draw header/main/editor/sidebar/pills/status
 -> Dialog Overlay 最后绘制
 -> canvas.Render() 变成终端字符串
```

布局按终端宽高计算 Rectangle。紧凑模式、小终端、Sidebar 可见性、编辑器动态高度都会改变区域。任何区域宽度必须扣除 border/padding；ANSI 字符串宽度必须用 `ansi.StringWidth/Cut/Truncate`，不能用 `len` 或字节切片。

## 5. Chat 与 List

`model.Chat` 包装 `list.List`，额外维护 message ID 到 index 的映射、follow 自动跟随、鼠标拖选、双击/三击、动画和滚动条预热。

`list.List` 是通用懒渲染列表：只渲染可见 item，维护 viewport、item 高度、scroll offset、focus/highlight。Item 最低只需 `Render(width) string`，可选 RawRenderable、Focusable、Highlightable 等能力。

聊天渲染是热点路径。昂贵 item 应自行缓存并在数据或宽度变化时失效。`TotalHeight` 会渲染全部 item，只应用于精确滚动条几何；判断是否溢出用有界 `Overflows`。Resize 拖动期间避免每帧全量计算，结束后用 `Prewarm` 渐进填充缓存。

## 6. 消息 Item 层次

`chat/messages.go` 定义：

- `MessageItem`：可渲染、有 ID、可输出 raw；
- `ToolMessageItem`：增加 tool call/result/status；
- 可选能力：Focusable、Highlightable、Expandable、Animatable、Compactable、KeyEventHandler；
- `cachedMessageItem` 等嵌入结构复用缓存、高亮、焦点逻辑。

Assistant item 处理普通文本、thinking、streaming Markdown、错误和 usage；User item 处理提示与附件。Tool item 工厂根据工具名选择 renderer：Bash、文件、搜索、网络、Agent、LSP、Todos、MCP；未知工具落到 Generic，因此新增工具不会完全无法显示，但专用 renderer 能提供更清楚的摘要、diff 与展开内容。

## 7. Streaming Markdown

流式 Markdown 不能每来一个 token 就无条件完整重渲染。`streaming_markdown.go` 跟踪稳定与未完成片段，对 fenced code、列表、段落边界做增量处理。最终完成后用 Common Markdown renderer 生成稳定输出。

语法高亮会根据 Theme 构建 Chroma style；该构建和 lexer 匹配已在 `common.ChromaStyle`、`xchroma.MatchLexer` 缓存。不要在 item 每次 Render 中直接创建 style 或全表匹配 lexer。

## 8. Dialog

`dialog.Overlay` 管栈，最顶层 Dialog 接收消息并最后绘制。Dialog 接口提供 ID、HandleMsg 和 Draw；Action 表示关闭、替换、提交等结果。

现有 Dialog 覆盖模型选择、Session、Commands、Permission、API key、OAuth/AWS SSO、MCP Auth、文件选择、推理级别、通知、退出，以及 Question 的确认/单选/多选/文本/表单变体。

宽度规则：Lip Gloss v2 的 `Width(n)` 是包含 border/padding 的总宽。内容宽应减 `GetHorizontalFrameSize()`；内缩使用 Padding 而非 Margin；帮助、标题、input、scrollbar 应使用共享 helper；最终宽高 clamp 到 drawable area。违反这些规则会在窄终端出现最后几个字符换行或边框溢出。

## 9. Editor、Completions 与 Attachments

textarea 是用户输入状态。Completions 根据触发前缀提供命令、文件等候选，拥有 filterable list，但消息是否消费由顶层决定。Attachments 管理粘贴/选择的文本、图片或文件，提交时转换为 `message.Attachment`。

提交过程通常：验证非空输入/附件 → 确保 Session → 清空或保留编辑器状态 → `Workspace.SendMessage` → UI 根据 Message 事件进入 streaming。失败要恢复可编辑内容或显示通知，不能静默丢失用户输入。

## 10. Styles 与 Theme

`styles.Styles` 是庞大的语义样式树。`quickStyle(opts)` 从 primary、secondary、fgBase、bgBase、success、error 等 token 生成基础样式；它必须保持 token 驱动，不应混入具体 Charmtone 颜色。`themes.go` 定义具体主题并只对确实无法由 token 表达的细节做覆盖。

组件拿 `*common.Common` 中的 `*Styles`，不自行构建散落颜色。新增 Theme 的正确路径是提供 palette、调用 quickStyle，再在 Theme 函数里做最小覆盖，并接入 Provider/主题选择。

## 11. 其他 UI 包

- `common`：Common 容器、Markdown、diff、highlight、scrollbar、按钮和能力判断；
- `diffview`：统一/左右分栏 diff、语法和行号；
- `notification`：OSC、native、bell、noop 等通知后端及平台选择；
- `image`：Kitty graphics 等终端图片；
- `logo`：字符 Logo 与随机布局；
- `anim`：spinner/帧动画；
- `util`：跨组件消息和小工具；
- `xchroma`：带缓存的 lexer 匹配。

## 12. UI 修改检查表

1. 是否只在顶层 `Update` 改状态；
2. IO 是否放入 `tea.Cmd`；
3. Dialog 是否最先消费输入；
4. 焦点是否正确恢复；
5. ANSI 宽度是否用 ansi 包；
6. 小终端是否 clamp；
7. border/padding 是否从内容宽扣除；
8. 高频 Render 是否缓存；
9. 本地与 ClientWorkspace 事件是否都有相同表现；
10. Catwalk golden/组件测试是否需要更新。
