# 03. 配置、Provider、模型发现与认证

> 状态：待复核生成稿｜生成日期：2026-08-14
> 基准提交：`5712d4839a6a10e9940804d511bb322dbe73a511`｜工作区：clean（开始分析时）
> 源码范围：`internal/config/`、`shellconfig/`、`discover/`、`oauth/`、`schema.json`
> 生成方式：源码、测试、配置与 Schema 静态分析

## 快速摘要

### 架构总览（模块与依赖）
ConfigStore 汇总 `crushrc`、JSON、默认 Provider 元数据和 CLI override；Coordinator 消费最终模型配置，OAuth 层维护外部认证状态。

### 核心调用序列（逐步逻辑）
1. 发现配置路径。2. 加载并深合并配置。3. 校验/选择 Provider 与模型。4. ConfigStore 原子替换并通知运行对象刷新。

### 易错点与边界条件
不要原地修改 Config 指针；`crushrc` builtin 仅在 ConfigBuilder context 生效；Token、header 与 DSN 必须保持 Secret 边界。

> 配置不是“读一个 JSON 文件”这么简单。Crush 会发现多级文件、执行 `crushrc`、深度合并、解析 shell 变量、补默认 Provider、发现本地模型、选择 large/small model，并支持运行中安全写回与刷新。

## 1. 配置系统的分层

```text
磁盘来源
  系统配置 -> 全局配置 -> 项目祖先配置 -> 当前项目配置
  每个 JSON 旁还可有 crushrc shell 配置
       ↓ 按优先级深度合并
原始 Config（保留用户模板）
       ↓ 默认值、环境解析、Provider 元数据、模型发现
运行时 Config 快照
       ↓ ConfigStore 原子发布
App / Agent / UI 并发只读
```

要点：磁盘的“用户意图”与运行时“已解析配置”不同。例如 `api_key: "$OPENAI_API_KEY"` 在运行时变成真实值，但持久化时不应把密钥展开结果覆盖模板。

## 2. 核心数据类型

关键文件：`internal/config/config.go`。

### 2.1 `Config`

顶层包含 Provider 映射、`Models`（large/small 选择）、MCP、LSP、agents、hooks、permissions 与 `Options`。映射字段通常使用并发安全容器/克隆策略，发布后的 Config 被视为不可变快照。

`IsConfigured()` 的语义不是“文件存在”，而是已有可用 Provider 与模型选择，足以初始化 agent。

### 2.2 `SelectedModel`

字段：

- `Model`：Provider API 使用的模型 ID。
- `Provider`：对应 Provider map key。
- `ReasoningEffort`、`Think`：推理模型开关。
- `MaxTokens`、temperature、top-p/top-k、frequency/presence penalty：选择级覆盖。
- `ProviderOptions`：模型级透传选项。

同一个模型 ID 可存在于多个 Provider；CLI 接收 `provider/model` 消歧。模型 ID 自身可以有 `/`，只有第一个分隔段在已知 Provider 时才作为 Provider 名。

### 2.3 `ProviderConfig`

| 字段 | 含义 |
|---|---|
| `ID` / `Name` | 内部 ID 与显示名 |
| `Type` | Catwalk 推理协议类型，如 OpenAI/Anthropic |
| `BaseURL` | API 根地址 |
| `APIKey` | 已解析密钥 |
| `APIKeyTemplate` | 原始 shell 模板，用于认证失败后重新解析 |
| `OAuthToken` | OAuth access/refresh token |
| `Disable` | 禁用 Provider |
| `Models` | Catwalk 模型元数据 |
| `AutoDiscoverModels` | 是否调用模型列表端点 |
| `ExtraHeaders` | 会做 shell 展开的请求头；空结果会被丢弃 |
| `ExtraBody` | OpenAI-compatible 请求体原样合并，不做递归展开 |
| `ProviderOptions` | Provider 级透传配置 |
| `AWSAuthRefresh` | Bedrock 认证失效时运行的刷新命令 |
| `FlatRate` | 订阅/固定费率，不累计普通 token 成本 |

Provider 的配置 map key 是用户引用的稳定身份；结构内 `ID` 在归一化阶段补齐。

### 2.4 其他配置

- `MCPConfig`：stdio/sse/http，command/args/env/URL、工具黑白名单、timeout、headers，以及 OAuth client/token/callback port。
- `LSPConfig`：command、args、env、filetypes、root markers、初始化参数、timeout。
- `Permissions.AllowedTools`：可写 `tool` 或更细的 `tool:action`。
- `Options`：数据目录、context/skills 路径、TUI、调试、自动摘要、工具禁用、指标、LSP、通知、归因等。
- `RuntimeOverrides`：只在本进程生效，如 yolo 和 channels，绝不写入配置文件。

## 3. 配置文件发现

关键函数：`lookupConfigs`、`GlobalConfig`、`ProjectConfigs`、`projectBoundary`。

来源包括：

1. OS 级系统配置（Unix 为 `/etc/crush/crush.json`）。
2. 用户全局配置/全局数据配置。
3. 从项目边界到 cwd 的项目配置，使更靠近 cwd 的文件优先。
4. JSON 配置的 shell sibling，以及主格式 `crushrc`。

项目边界考虑 Git worktree，`worktreeRootCache` 避免重复计算。另有全局与项目技能目录发现，支持 `.agents/skills` 等约定位置。

配置路径即使尚不存在也可能被快照跟踪；这样用户稍后创建全局 `crushrc` 时，staleness 检测仍能发现。

## 4. `Load` 的完整管线

关键文件：`internal/config/load.go`。

可按以下阶段理解：

1. 规范 workingDir/dataDir，并迁移旧环境或旧字段。
2. `lookupConfigs` 得到有序路径。
3. 对每个存在的文件读取字节。
4. JSON 直接进入合并输入；shell 文件交给 `shellconfig.LoadShellConfig` 执行并转为一个 JSON object。
5. `loadFromBytes` 用深度合并得到 Config；后面的值覆盖前面的同路径值，map 递归合并。
6. 保存各目录显式出现过的顶层 key，用于处理覆盖/禁用语义。
7. `setDefaults` 设置绝对数据目录、上下文路径、TUI、通知、归因等默认值。
8. 加载 Catwalk 已知 Provider 元数据；可自动更新缓存。
9. `configureProviders` 合并默认 Provider 与用户 Provider，解析 API key/base URL/headers，补模型。
10. 对允许自动发现的 Provider 调用 discover 层，并与显式模型合并；显式模型获胜。
11. 应用默认 LSP 列表。
12. 解析/校验 large 与 small 模型；无效的旧选择可回退默认值并标记 fallback。
13. 校验 hooks 等跨字段规则。
14. 建立 `ConfigStore`，保存来源内容哈希/快照供热重载判断。

错误信息会经过清理，避免 shell 展开失败时把敏感值直接暴露到日志。

## 5. `crushrc` 如何工作

关键目录：`internal/shellconfig/`。它不是另写一套 parser，而是使用 Crush 自己的 Bash 解释器。

### 5.1 安全门：`ConfigBuilder` context

加载时创建 `ConfigBuilder{root: map[string]any}`，通过私有 context key 注入 shell。`provider`、`model`、`mcp`、`lsp`、`permissions`、`hook`、`option` builtins 只有检测到 builder 才生效；普通 bash 工具执行时它们是 no-op，避免用户代码意外修改配置状态。

脚本在自身目录执行，继承环境，并获得 `CRUSH_VERSION`。它支持 `source`、变量、条件与 `$(command)`。每次加载最多 30 秒，并服从上层更早的 deadline；超时/取消返回错误，旧配置快照保持不变。

执行结束后 builder marshal 成单一 JSON object，再进入统一合并管线。因此 JSON 和 shell 两种配置最终只有一套语义。

### 5.2 builtins

- `provider add/remove`：创建/更新 Provider，支持 type、key、URL、headers、extra body、模型发现等。
- `model add/remove`：Provider 必须先存在；相同模型 ID 重加会替换。
- `model large/small`：设置选择与采样参数；无参数时打印当前选择。
- `mcp add/remove`：stdio/http/sse、OAuth、headers、工具列表。
- `lsp add/remove`：命令、文件类型、root marker 等。
- `permissions allow/deny`：累积并去重；deny 优先移除 allow。
- `hook add/remove/clear`：按事件维护命令。
- `option`：带类型表的 bool/string/list/UI/attribution 设置；list 支持 reset。

参数解析由声明式 `flagSpec` 完成，统一处理 string、bool、int、float、key/value 与 JSON object，避免每个 builtin 重复造轮子。

示例：

```bash
provider add local \
  --type openai \
  --base-url http://localhost:11434/v1 \
  --api-key ollama \
  --discover-models true

model large local/qwen3-coder
permissions allow view
option disable_metrics true
```

## 6. 默认 Provider 元数据与缓存

关键文件：`internal/config/provider.go`、`catwalk.go`、`hyper.go`。

Catwalk 提供 Provider/Model 清单。缓存存放在全局 cache 目录，包含数据与 ETag：

- 首次或允许自动更新时请求远端。
- `Not Modified` 使用本地缓存。
- 网络失败在有缓存时尽量降级使用缓存。
- `sync.Once` 避免同一进程重复同步。
- `UpdateProviders(pathOrURL)` 支持显式更新源。
- `DisableDefaultProviders` 为 true 时完全忽略内置清单，要求用户完整声明 URL、模型和认证。

用户 Provider 与默认 Provider 合并时，用户字段优先，但未给出的模型能力、价格、上下文窗口等可从 Catwalk 补齐。

## 7. 本地/兼容 Provider 模型发现

关键目录：`internal/discover/`。

### 7.1 公共流程

discover 维护按 Provider 类型/特征注册的 `Enricher`。典型过程：

1. 根据配置的 BaseURL、API key、headers 构造请求。
2. 调用 OpenAI-compatible `/v1/models` 或厂商接口得到模型 ID。
3. 交给特定 enricher 获取上下文窗口、附件/推理能力等。
4. 与用户 `Models` 合并：用户声明同 ID 时覆盖发现结果。
5. 网络不可达一般不应破坏所有配置；错误被记录，显式模型仍可用。

已支持的本地生态包括 Ollama、LM Studio、llama.cpp、LiteLLM、OMLX。

### 7.2 Ollama

`ollamaEnricher` 对每个模型调用 Ollama show API，从 `model_info` 中抽取 context length。`extractContextLength` 兼容不同架构的键名/数字表示，并选择有效上下文值。测试覆盖缺字段、不同数字类型和真实响应形状。

实用配置：

```bash
provider add ollama \
  --type openai \
  --base-url http://localhost:11434/v1 \
  --api-key ollama \
  --discover-models true
```

若自动发现不可靠，可显式声明：

```bash
provider add ollama --type openai --base-url http://localhost:11434/v1 --api-key ollama --discover-models false
model add ollama/qwen3-coder:latest --context-window 32768 --default-max-tokens 8192
model large ollama/qwen3-coder:latest
model small ollama/qwen3:8b
```

## 8. 模型选择与临时覆盖

加载时 `resolveSelectedModels` 校验 Provider 存在、未禁用且模型 ID 可找到。若没有选择则 `defaultModelSelection` 从可认证/可用 Provider 中选 large 和 small。

`crush run -m ... --small-model ...` 不应直接改用户文件：它在运行时匹配 Provider/Model，并更新当前 agent。匹配规则：

- 只写 model ID：在所有启用 Provider 中查找。
- `provider/model`：限定 Provider。
- 0 个匹配报 not found；多个匹配要求显式 Provider。
- 模型 ID 中的 slash 只有首段恰好是 Provider key 时才拆分。

## 9. `ConfigStore`：并发与持久化核心

关键文件：`internal/config/store.go`、`atomicwrite.go`、`scope.go`。

### 9.1 为什么不能原地修改 Config

UI、agent、MCP 与 server 可能并发读配置。Store 使用 copy-on-write：

1. `writeMu` 串行化写者。
2. `cloneForWrite()` 深克隆当前 Config。
3. typed mutator 修改克隆。
4. 原子发布新指针，旧读者继续安全读取旧快照。

发布后的 map/slice 不再原地修改，这是避免 data race 的核心约束。

### 9.2 Scope

- `ScopeGlobal` 写全局数据配置，始终有路径。
- `ScopeWorkspace` 写项目配置；无 workspace path 返回 `ErrNoWorkspaceConfig`。

Token 等敏感、跨项目认证通常写 Global；项目模型偏好等可按调用者选择 Scope。

### 9.3 两类更新 API

- `SetConfigField(s)`：通用 JSONPath 写磁盘，随后重新加载，因为 Store 不知道该字段的派生影响。
- typed `update`：先准确修改克隆，再只持久化对应字段，无须全量 reload。

磁盘更新流程：进程内 mutex -> 跨进程 flock -> 读取当前 JSON（不存在视为 `{}`）-> sjson 修改 -> 临时文件 -> chmod 0600 -> fsync/rename 原子替换。Windows rename 有短暂重试预算。

写盘成功但 reload 失败时，写操作已经不可撤销：返回/记录警告并继续保留旧内存快照，下一次明确 reload 可恢复一致。

### 9.4 Staleness 与 reload

Store 为所有候选配置路径记录内容状态；文件新增、删除或内容变化都会被判定为 stale。`ReloadFromDisk` 在写锁下完整构造新 Config，只有全部成功才发布，因此坏掉或挂起的 `crushrc` 不会破坏当前可用配置。

## 10. Provider 认证

### 10.1 普通 API key

API key、BaseURL 和 headers 使用 shell variable resolver，可支持 `$VAR` 与命令替换。原模板保存在 `APIKeyTemplate`，当请求遇到认证错误时可重新解析，以适应外部命令产生的短期凭据。

不要把展开后的 secret 写回磁盘，也不要在错误中打印 resolver 的完整环境。

### 10.2 Hyper 设备流

`crush login` 默认登录 Hyper：

1. `InitiateDeviceAuth` 请求 device/user code。
2. 复制 user code，用户确认后打开 verification URL。
3. `PollForToken` 按间隔轮询，受 expiresIn context 限制。
4. 得到 refresh token 后 `ExchangeToken` 换 access token。
5. `IntrospectToken` 验证 active。
6. `SetProviderAPIKey(ScopeGlobal, "hyper", token)` 原子保存。

所有 HTTP body 都限制读取大小，非 200 响应携带状态与有限 body。

### 10.3 GitHub Copilot 设备流

优先读取 GitHub/Copilot 已有磁盘 token 并刷新；没有时请求 device code，轮询授权，再用 GitHub token 换 Copilot token。轮询处理 `authorization_pending`、间隔和过期；账号无 Copilot 权限返回专门错误并提示注册地址。Provider 还会注入 Copilot 要求的 headers。

### 10.4 OAuth Token 结构与刷新并发

通用 `oauth.Token` 保存 access/refresh、过期时间，并可带 OAuth client 注册信息与 auth/token endpoint。ConfigStore 的刷新逻辑同时防两类竞争：

- 进程内用 singleflight，多个请求只交换一次 refresh token。
- 跨进程先重读磁盘，若别的进程已经写入更新 token，就采用它而不再拿旧 refresh token 交换。

旋转 refresh token 时尤其重要：旧 token 可能只能用一次。

### 10.5 MCP OAuth 2.1

关键文件：`internal/oauth/mcp/handler.go`。

HTTP MCP 可启用 OAuth。Handler 基于 MCP Go SDK 的 authorization-code handler：

1. 从固定端口或 40704~40713 找 callback port。
2. 生成 `http://localhost:<port>/callback`。
3. 优先使用显式 client ID/secret；否则恢复此前动态注册 client；再否则执行 DCR。
4. 有可刷新 token 时注入 InitialTokenSource，启动阶段静默刷新。
5. 新 token/刷新 token 经 saving token source 比较后持久化。
6. 后台启动 context 不允许打开浏览器，返回 `ErrInteractiveAuthRequired`；只有用户明确触发并用 `WithInteractive` 才能授权。
7. 远程 Client/Server 可 suppress server 浏览器，由客户端在自己的桌面打开 URL。

Callback receiver 对并发 Authorize 做串行控制，同一次登录只打开一个 tab；校验 state/issuer，HTML 输出转义不可信错误文本。callback 页面模板修改必须遵守其 `AGENTS.md` 的 Prettier 与单行 template action 要求。

## 11. Logout

`logout` 通过远程配置 API 删除全局 `providers.<id>.api_key` 和 `.oauth`。无参数时只列出带 OAuth token 的已登录平台；有多个时询问。删除是配置层操作，不应直接修改运行时 Provider 指针，否则 C/S 下其他客户端无法收到一致更新。

## 12. 常见错误边界

- shell 配置超时：拒绝新配置，继续用旧快照。
- Provider 被禁用：不参与默认选择和 CLI 匹配。
- 自动发现失败：保留显式模型；提示用户检查 BaseURL。
- 模型重名：要求 `provider/model`。
- workspace scope 没有目标文件：返回明确错误，不偷偷写全局。
- 原子 rename 失败：不发布新内存快照。
- OAuth 后台缺 token：报告 needs-auth，禁止悄悄弹浏览器。
- token refresh 竞争：先采用更新的磁盘 token，避免复用旧 refresh token。

## 13. 从零复刻步骤

1. 定义纯数据 `Config`、`ProviderConfig`、`SelectedModel`。
2. 先只支持一个 JSON 路径和明确覆盖规则，写合并测试。
3. 加多级路径发现，并测试祖先到 cwd 的优先级。
4. 实现 defaulting 与 validation，确保磁盘模型和运行时模型分离。
5. 实现 ConfigStore copy-on-write 与原子 JSONPath 写。
6. 加候选路径快照与全有或全无 reload。
7. 接入默认 Provider 元数据缓存，再接模型选择。
8. 抽象 Discoverer，先实现 OpenAI `/models`，最后写 Ollama enricher。
9. 接入 shell 解释器和 context-gated builder；加 30 秒 timeout。
10. 实现 API key 模板解析及错误脱敏。
11. 实现单一 OAuth 设备流，再抽通用 Token 与跨进程刷新策略。
12. 最后实现 MCP OAuth，分别测试恢复、刷新、无 DCR、错误 issuer、并发授权和无浏览器环境。

## 14. 推荐源码阅读路径

```text
internal/config/config.go
internal/config/load.go
internal/config/store.go + atomicwrite.go + scope.go
internal/config/provider.go + catwalk.go + hyper.go
internal/shellconfig/builder.go + load.go + register.go
internal/shellconfig/provider.go + model.go + 其余 builtins
internal/discover/discover.go + 各 enricher
internal/oauth/token.go
internal/oauth/hyper/* + copilot/*
internal/oauth/mcp/handler.go + savingtokensource.go + callback/*
internal/cmd/login.go + logout.go + models.go
```

测试优先阅读 `shellconfig_precedence_test.go`、`reload_crushrc_test.go`、`refresh_singleflight_test.go`、`model_pin_test.go`、`scopeb_race_test.go`、各 discover 测试与 `oauth/mcp/handler_test.go`。它们是配置优先级和并发契约最准确的说明。
