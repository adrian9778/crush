# 02. 启动、命令与配置

## 1. 从 `main` 到 TUI

`main.go` 导入 `godotenv/autoload`，因此 `.env` 可在进程启动早期进入环境；空白导入 `internal/dns` 用于平台 DNS 行为。若设置 `CRUSH_PROFILE`，后台在 `localhost:6060` 暴露 pprof。随后进入 `cmd.Execute()`。

`cmd.Execute()` 先暂时丢弃早期 slog，原因是日志目录依赖尚未加载的配置；之后为终端版本输出添加彩色图案，最后由 Fang/Cobra 解析命令、处理信号并运行。

根命令未指定子命令时进入交互 TUI：解析 `--session`/`--continue`，创建 Workspace，构造 `common.Common` 与 `ui.UI`，安装 Bubble Tea 输入过滤器并运行程序。

## 2. CLI 命令地图

`internal/cmd` 包含：

- `root.go`：交互模式、Workspace 初始化、全局 flag；
- `run.go`：非交互提示、stdin/附件、流式输出和退出条件；
- `server.go`：独立 Server 生命周期；
- `models.go`：列出与刷新 Provider/Model；
- `login.go`、`logout.go`：API key/OAuth 登录状态；
- `session.go`：会话列表、导出等管理操作；
- `stats.go` 与 `stats/`：数据库统计和展示；
- `dirs.go`、`projects.go`、`logs.go`：数据目录、项目与日志辅助；
- `schema.go`：配置 JSON Schema；
- 平台文件处理 Windows 特例。

`crush run` 与 TUI 的区别不是另一套 Agent。它创建同样的 Workspace，通过消息和 `RunComplete` 等待一次 turn，按参数选择文本/JSON/流式输出。

## 3. Workspace 初始化顺序

`setupWorkspace` 根据 `CRUSH_CLIENT_SERVER` 选择路径。本地初始化的关键顺序：

1. `ResolveCwd` 得到工作目录；
2. `config.Init` 加载并构造 `ConfigStore`；
3. 将 `--yolo`、`--channels` 写入仅运行时 Overrides；
4. 创建数据目录和忽略全部内容的 `.gitignore`；
5. 注册最近项目；
6. `db.Connect` 获取按数据目录复用的数据库连接；
7. 初始化文件日志；
8. 按配置发现 Skills，并决定是否镜像全局状态给 TUI；
9. `app.New` 装配核心服务；
10. 用 `AppWorkspace` 适配成 UI 接口；
11. cleanup 负责按逆向生命周期释放资源。

远程初始化会连接或启动 Server，创建远端 Workspace，再启动 SSE 同步。协议版本与能力不兼容时会在此阶段失败。

## 4. 配置对象

`internal/config/config.go` 定义纯数据结构：

- `Config`：顶层 Providers、MCP、LSP、Agents、Permissions、Options、Hooks；
- `ProviderConfig`：Provider 身份、API 参数、模型表、OAuth、额外 header/option；
- `SelectedModel`：模型 ID、Provider、推理级别及调用参数；
- `Agent`：模型选择、允许工具、系统提示等；
- `MCPConfig`：stdio/http/sse 类型、命令、URL、环境和 headers；
- `LSPConfig`：命令、参数、环境、文件类型与初始化选项；
- `Options`/`TUIOptions`：数据目录、通知、主题、紧凑模式等；
- `HookConfig`：事件、匹配器、命令、超时。

`Config` 应被视为不可随意原地改写的已解析快照。持久化与运行时覆盖由 `ConfigStore` 管理。

## 5. 配置发现与合并

`config.Load` 是一条较长的管线，阅读时分阶段看：

1. 计算全局、项目、工作树和兼容配置路径；
2. 同时发现首选的 `crushrc` 与已弃用但兼容的 `crush.json`；
3. 对 `crushrc` 使用嵌入 Shell 与 ConfigBuilder 执行；
4. 将多个来源做深度合并，记录顶层字段来源；
5. 设置工作目录、数据目录、默认 Options；
6. 读取/刷新 Catwalk Provider 元数据与 Hyper Provider；
7. 展开环境变量与 Shell 表达式，但对错误做脱敏；
8. 应用 Provider、LSP 默认值；
9. 解析 large/small model，处理同名模型歧义；
10. 建立默认 Agent 配置、工具 allow list；
11. 校验 Hooks 等约束；
12. 将加载路径、变量 resolver、Provider 元数据和快照封装进 `ConfigStore`。

工作树场景会计算 project boundary 与 worktree root，避免错误地只看当前子目录或跨越仓库边界。Skills 路径也按全局、项目和兼容目录分层发现。

## 6. `crushrc` 为什么是 Bash

`internal/shellconfig` 注册 `provider`、`model`、`mcp`、`lsp`、`permissions`、`hook`、`options` 等 builtin。加载配置时 Context 中存在 `ConfigBuilder`，builtin 才写入配置；普通 Bash 工具执行时没有 Builder，因此这些命令不应误改配置。

执行流程是：

```text
load crushrc -> shell.Run -> builtin registry -> ConfigBuilder -> Config
```

各 builtin 文件负责 flag 解析、值转换和 Builder 调用；`register.go` 完成集中注册。新增配置语言能力必须同时考虑 shell builtin、JSON 兼容结构、Schema、默认值、合并与测试。

## 7. `ConfigStore`

`ConfigStore` 解决三个纯 `Config` 无法解决的问题：并发读写、落盘、运行期状态。

主要职责：

- 暴露当前配置、工作目录、Resolver 和已加载路径；
- 保存 `RuntimeOverrides`，例如 yolo 和临时启用 channels；
- 使用文件锁与原子写更新全局或项目 scope；
- 更新模型偏好、最近模型、UI 选项和 Provider key；
- 管理 OAuth token 刷新，利用 provider 级锁避免多进程重复刷新；
- 保存磁盘快照，检测配置是否被外部修改；
- 从磁盘 reload，并重新构造派生状态；
- 在安全范围内同步内存配置与文件配置。

原子写先写同目录临时文件、同步并 rename；Windows 对 rename 的短暂失败有重试适配。不要绕过 Store 直接 `os.WriteFile` 配置，否则会破坏锁、快照和内存一致性。

## 8. Provider 与模型选择

Catwalk 提供已知 Provider/Model 元数据，Crush 再与用户配置合并。模型既可写 `model-id`，也可写 `provider/model-id`；如果模型 ID 本身含 `/`，只有首段确实是已配置 Provider 时才拆分。无 Provider 前缀且多个 Provider 含同名模型会报歧义。

Coordinator 构造实际 Fantasy Provider 时会按类型选择 Anthropic、OpenAI、Google、Bedrock、OpenRouter、Vercel、OpenAI-compatible、Copilot、Hyper 等实现，并注入 base URL、API key、headers、OAuth 和模型能力。部分 Provider/模型还决定使用 Responses API 还是 Messages/Chat Completions API。

## 9. 配置相关辅助包

- `discover`：探测 Ollama、LM Studio、llama.cpp、LiteLLM、OMLX 等本地服务并丰富 Provider 信息；
- `oauth`：通用 token 类型及 Copilot、Hyper、MCP OAuth 流程；
- `home`：可靠获得用户主目录；
- `projects`：维护最近工作目录；
- `lock`：跨平台文件锁；
- `env`：环境变量抽象，便于测试；
- `update`/`version`：版本与更新检查。
