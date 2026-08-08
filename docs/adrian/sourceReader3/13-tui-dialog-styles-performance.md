# 对话框、样式、终端能力与性能边界

## 1. Overlay：为什么对话框是栈

`dialog.Overlay` 保存 `[]Dialog`，末尾是最前面的活动对话框。`OpenDialog` push，`CloseFrontDialog` pop，`BringToFront` 调整顺序；Draw 按数组顺序画，因此后画的覆盖先画的。

Dialog 接口很小：

```go
type Dialog interface {
    ID() string
    HandleMsg(tea.Msg) Action
    Draw(uv.Screen, uv.Rectangle) *tea.Cursor
}
```

Dialog 不直接操作顶层 Workspace，而是返回 Action。顶层 `handleDialogMsg` 负责选择模型、切 Session、权限响应、运行命令、配置切换等。这保证 side effect 仍由唯一主循环调度。

异步打开的权限窗使用 `OpenDialogWithGrace`：425ms 安静期或最多 1500ms 内吸收仍在飞行的旧按键，防止用户打开窗前按下的 Enter 意外批准权限。同类窗 500ms 内连续重开则跳过 grace，避免连续权限请求每次吞按键。

## 2. Dialog 家族

| 文件 | 用途和核心状态 |
|---|---|
| `models.go/models_list.go/models_item.go` | provider/model 分组、过滤、configured 标记、模型选择和认证分支 |
| `sessions.go/sessions_item.go` | Session filter、选择、删除确认、重命名；当前 busy Session 保护 |
| `commands.go/commands_item.go` | 内置命令、自定义命令、skills、MCP prompts；有参数时转 Arguments |
| `arguments.go` | 动态参数表单、required、焦点输入切换和最终替换 |
| `permissions.go` | tool 名称、参数 JSON、session/always/once/deny 等决定 |
| `filepicker.go` | bubbles file picker，选中后由 Action Cmd 检查大小并读取附件 |
| `reasoning.go` | 当前模型可选 reasoning effort |
| `notifications.go` | auto/native/OSC/bell/disabled 配置 |
| `api_key_input.go` | API key 输入、异步验证、loading/error |
| `oauth*.go` | device code、Copilot、Hyper OAuth 状态机 |
| `aws_sso.go` | AWS SSO 命令和浏览器认证结果 |
| `mcp_auth.go` | 需要认证的 MCP 列表、逐项 OAuth、timeout/cancel |
| `quit.go` | quit/continue，并根据 busy 状态改变提示 |
| `question_*` | yes/no、single、multi、free text、confirm、editor、batch tabs |
| `inline_editor.go` | 顶层 editor 区可嵌入组件的共同接口 |

Question 系列值得单独理解：单问题可替换 textarea 成 inline editor；多问题由 `QuestionForm` 管理 tabs、每页 answer 和最终 confirm。失焦时空间不足可折叠，重新聚焦恢复。choice base 复用高度、hover、滚动和 free-text fill-in。

## 3. 创建一个新 Dialog 的标准步骤

1. 定义稳定、唯一的 ID。
2. 保存 `*common.Common` 或 `*styles.Styles`、输入/list/help、宽高、loading/error。
3. constructor 初始化 keymap、filter list、输入样式。
4. `HandleMsg` 只改变本地 UI 状态或返回 `Action/ActionCmd`。
5. `Draw` 先从 area 夹取总宽高，再算 inner width 和各块高度。
6. 用 `RenderContext` 生成统一 title gradient、body 和 help。
7. 居中窗调用 `DrawCenterCursor`；onboarding 用 `DrawOnboardingCursor`。
8. 若有输入，通过 `InputCursor` 加上 frame offset。
9. 在 `UI.openDialog`/`handleDialogMsg` 注册入口和 Action。
10. 添加小终端、长文本、鼠标、Escape、loading 的测试。

## 4. 最容易出错的尺寸规则

lipgloss v2 的 `Width(n)` 是总 box 宽度，border 和 padding 已包含在内。正确公式：

```text
outer width
- Dialog.View.GetHorizontalFrameSize()
= inner/content width
```

进一步给 input 时，还要扣 `InputPrompt` frame、`input.Prompt`（如 `> `）和一个 cursor cell；公共函数是 `dialogInputTextWidth`。

必须遵守：

- 内缩用 `Padding`，不要用 `Margin`；margin 位于 width 外部，会把 dialog 推出边框。
- 样式段分别 `style.Render(x)` 后拼接。若给“含内部 reset 的整段字符串”再套外层前景色，内部 reset 后的后半段会丢色。
- help 用 `renderDialogHelp`，它贪心放 key hints、去悬空 separator、必要时加省略号。
- title 用 `common.DialogTitle` 截断；TitleInfo 放不下时整体隐藏，不切坏有背景的 badge。
- list 用 `sizeDialogList`，只在溢出时为 scrollbar 预留一列；`joinScrollbar` 合并。
- secondary info 太挤时 `applyInfoColumnVisibility` 整列隐藏，而不是每行参差截断。
- outer width/height 必须 clamp 到 `area.Dx/Dy`，并允许 inner size 变成 0。

## 5. Styles 三层架构

### 5.1 `styles.Styles`：完整样式契约

它是一棵按语义分组的结构：ANSI palette、Header、Editor、Messages、Tool、Dialog、Status、Completions、Attachments、Pills、Sidebar、ModelInfo、Resource、Files、Logo、Diff、Markdown、Chroma 等。

组件读取语义字段，如 `Tool.ErrorMessage`，不要直接引用某个具体颜色。这样主题切换才不会漏掉组件。

### 5.2 `quickStyle(opts)`：主题无关的构造器

`quickStyleOpts` 是设计 token：primary/secondary、前景层级、背景层级、success/error/warn、selection 等。`quickStyle` 用它们构造巨大 Styles。这里原则上不能硬编码 Charmtone 颜色（尚未完全 token 化的 Chroma 例外）。

### 5.3 `themes.go`：具体 palette 和少量 override

当前 `CharmtonePantera` 与 `HypercrushObsidiana` 先调用 quickStyle，再对真正主题特有的细节覆盖。`ThemeKeyForProvider` 将 Hyper provider 映射成 hypercrush，其余为 charmtone；`ThemeForProvider` 返回具体 Styles。

UI 缓存 `themeKey`，相同主题跳过昂贵重建。真正 apply 时不仅替换 `com.Styles`，还要更新 textarea、attachments renderer、completions、spinner，并清 Markdown/Chroma/chat/sidebar/image 等依赖旧主题的 cache。

## 6. Markdown、Chroma 和 Diff 样式

`common.MarkdownRenderer` 与 Quiet 版本按 `(Styles,width)` 缓存 Glamour renderer。普通版用于回答，quiet 版用于较低视觉权重区域。主题变化必须 `InvalidateMarkdownRendererCache`。

`common.ChromaStyle` 从 `Styles.ChromaTheme()` 构造 style，但构造昂贵，所以 memoize。lexer 匹配也通过 `ui/xchroma` 缓存；热路径禁止直接反复 `chroma.MustNewStyle` 或 `lexers.Match`。

Diff 样式把 insert/delete/context/hunk、line numbers、背景、Chroma 合成 `diffview.Style`。Golden 同时覆盖 light/dark，说明主题不是“换一个前景色”这么简单。

## 7. ANSI 和终端列宽

终端字符串含不可见 escape sequence，Unicode rune 也不等于显示列。项目规则是：任何可能含 ANSI 的 Cut、Strip、Truncate、StringWidth 都使用 `github.com/charmbracelet/x/ansi`。

典型错误：

- `len(s)` 计算宽度：ANSI 被计入，中文/emoji 宽度错误。
- `s[:n]` 截断：可能切断 UTF-8 或 SGR sequence。
- 用 strings trim 处理彩色列：会留下 reset/背景泄漏。

`common/ansi16.go` 还负责把程序输出的标准 16 色重映射到主题 palette、清理 cursor 控制、模拟 `\r` 覆盖行为。这对 shell progress output 尤其关键。

## 8. Ultraviolet ScreenBuffer 与 draw cache

屏幕绘制采用 cell buffer：StyledString 解析 ANSI 后画入 rectangle，后画层覆盖先画层。好处是 sidebar、popup、dialog 不需通过空格拼接定位，鼠标坐标与绘制几何一致。

Chat 又维护一层最终 ScreenBuffer cache：渲染字符串、ANSI method 和 bounds 未变时把缓存 cell 复制到目标 screen。resize 时暂时隐藏精确 scrollbar，等待 120ms settle 后分批预热每项 cache，避免用户拖动窗口时每一帧都计算总高度。

`View` 最后统一 CRLF→LF，并去每行尾空格，减少 Bubble Tea diff renderer 的无意义输出。

## 9. 图像显示

`Capabilities` 从环境和终端响应收集 color profile、cells、pixels、Kitty/Sixel、focus report、OSC99。查询通过 raw escape sequence Cmd 发出；tmux 下 Kitty query/transmit 需要 passthrough。

`image.Encoding` 有两种：

- `EncodingKitty`：按 cell size 等比缩放，Kitty graphics transmit-and-put，再用 Unicode placeholder + diacritic 标出 image ID/行列。
- `EncodingBlocks`：不支持 Kitty 时用 `go-ansi-paintbrush` 转字符块和颜色。

cache key 是 image ID + cols + rows；尺寸不同必须重新 fit/transmit。cache 有 RWMutex，因为图像 Cmd 与 UI render 可能跨 goroutine。主题或上下文重置时需清理。

## 10. 通知后端

支持：

- native：beeep，部分平台 build tag 为 stub。
- OSC：优先 OSC 99，降级 OSC 777；适合 SSH/macOS/支持终端。
- bell：终端响铃。
- noop：禁用。

显式配置优先。auto 模式下 SSH 用 OSC；macOS 或无 native 用 OSC；其他本地环境只有在支持 focus event 时用 native，否则 noop。只有终端能报告 focus 且窗口当前失焦才发送，避免用户正在看 UI 时重复通知。

## 11. 性能设计清单

TUI 性能瓶颈通常不是业务 IO，而是每个 token 导致的重新排版：

1. `Update` 内不做 IO；Workspace HTTP 查询必须 Cmd + cache。
2. message Updated 不刷新 busy/queue；只在 run boundary 刷新。
3. List 只渲染 viewport，并缓存预切分 lines。
4. item mutation 必须 bump version；完成 item 冻结。
5. Assistant 分 thinking/content/error cache。
6. Streaming Markdown 复用安全稳定前缀。
7. Chroma style/lexer、Markdown renderer memoize。
8. resize 用 settle + Prewarm；是否溢出用 `Overflows`，不要每帧 `TotalHeight`。
9. sidebar 预渲染并虚拟滚动。
10. 高频 wheel/motion 由 `model.Filter` 约 16ms 合并，方向反转时立即放行。
11. 动画仅对可见/仍 running item tick；不可见动画可暂停，重新可见时恢复。
12. 样式主题未改变时不重建。

缓存正确性的共同原则：cache key 必须包含影响输出的一切（width、source hash、expanded/focus/compact、theme），mutation 必须显式失效。宁可一次 cache miss，不可显示旧内容。

## 12. 测试和 Golden 的阅读方法

- Dialog：`overlay_test.go` 验证 stack/grace；permissions/question tests 验证交互与小尺寸。
- Styles/UI：`pill_border_test.go`、layout tests 捕获一列溢出和边框断裂。
- ANSI：`common/ansi16_test.go` 覆盖 SGR、回车和 cursor 清理。
- Image：`image_test.go` 覆盖 fit/cache/encoding。
- Notification：`notification_test.go` 覆盖后端策略。
- Diff：数百 golden 是 width/height/offset 的二维规格，不应轻易整体 update。
- 性能：chat resize、thinking streaming benchmarks 应查看 ns/op 和 allocs/op，而不只看“能跑”。

Golden 失败时先判断是期望视觉变化还是宽度算错。若最后几个字符换行，优先查是否把 outer width 当成 inner width、是否错误用了 margin、是否漏扣 scrollbar/prompt/cursor。

## 13. 完整复刻的最后阶段

在聊天主流程可用后，按以下顺序补齐本篇能力：

1. Overlay + 最简单 Quit dialog。
2. RenderContext 和公共 sizing helpers。
3. Models/Sessions/Commands 三种 filter list dialog。
4. Permission 与异步 grace。
5. question inline/batch editors。
6. quickStyle + 一个主题；所有组件只读语义 Styles。
7. Markdown/Chroma/Diff cache。
8. ANSI 安全宽度和 shell 输出。
9. capabilities、Kitty image、notification。
10. 加 resize、长对话、长 thinking benchmarks；通过全部 golden 后再做视觉微调。

如果实现结果“功能正确但终端偶尔闪、卡、边框断”，问题大多就在尺寸 frame 计算、ANSI 列宽或 cache 失效协议，而不是 Bubble Tea 本身。
