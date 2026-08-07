# Crush 本地开发与使用记录

本文记录 Crush 的本地编译、基础使用和模型 Provider 配置，重点说明如何通过 Ollama 使用本地模型。

# 第一部分：编译失败排查

## 结论

当前遇到的失败并不是项目源码无法编译，而是 Go 默认使用的缓存目录在当前受限环境中不可写。

将构建缓存切换到可写目录后，项目可以成功编译，命令退出码为 `0`。

## 默认编译错误

直接执行：

```bash
go build .
```

出现以下错误：

```text
open /Users/vincent/Library/Caches/go-build/...: operation not permitted
```

Go 还会尝试更新模块下载缓存中的 stat cache：

```text
go: writing stat cache:
open /Users/vincent/workspace/12_GO/pkg/mod/cache/download/...:
operation not permitted
```

这两条信息都表明 Go 正在尝试写入当前执行环境不允许写入的目录。

## 根因

当前 Go 环境使用的目录为：

```text
GOCACHE=/Users/vincent/Library/Caches/go-build
GOMODCACHE=/Users/vincent/workspace/12_GO/pkg/mod
GOPATH=/Users/vincent/workspace/12_GO
```

其中默认构建缓存 `GOCACHE` 不可写，导致 `go build` 直接失败。

模块缓存 `GOMODCACHE` 中的依赖可以读取，但 Go 无法写入新的 stat-cache 临时文件，因此会额外输出 `operation not permitted` 警告。这条警告在本次验证中没有阻止最终构建。

## 解决方法

将 Go 构建缓存设置为当前环境允许写入的目录：

```bash
mkdir -p /tmp/crush-go-build-cache

GOCACHE=/tmp/crush-go-build-cache \
CGO_ENABLED=0 \
GOEXPERIMENT=greenteagc \
go build .
```

上述命令已经验证成功，退出码为 `0`。

如果需要在当前终端中连续执行多次构建，可以先设置环境变量：

```bash
export GOCACHE=/tmp/crush-go-build-cache
export CGO_ENABLED=0
export GOEXPERIMENT=greenteagc

go build .
```

## 项目编译环境要求

根据项目的 `go.mod` 和开发说明，编译环境应满足：

- Go 版本为 `1.26.5`；
- 设置 `CGO_ENABLED=0`；
- 设置 `GOEXPERIMENT=greenteagc`。

可以使用以下命令检查本机环境：

```bash
go version
go env GOCACHE GOMODCACHE GOPATH
go env CGO_ENABLED GOEXPERIMENT GOTOOLCHAIN
```

如果在不受沙箱限制的普通终端中仍然出现缓存权限错误，可以修复对应缓存目录的所有权和权限，或者继续使用自定义的可写 `GOCACHE`。

## 最终判断

本次验证结果如下：

| 检查项 | 结果 |
| --- | --- |
| 当前 Go 版本 | `go1.26.5 darwin/arm64` |
| 默认 `go build .` | 失败，构建缓存不可写 |
| 使用 `/tmp` 作为 `GOCACHE` | 成功 |
| 是否发现源码编译错误 | 否 |
| 主要问题类型 | 本地执行环境与缓存目录权限 |

因此，排查这类错误时应先区分“编译器报告的源码错误”和“Go 工具链无法读写缓存或依赖目录”。本次属于后者。

# 第二部分：Crush 基础使用

## 1. Crush 是什么

Crush 是一个运行在终端中的 AI 编程助手。它可以维护多组 Session，在对话过程中读取和修改代码、运行 Shell 命令、调用 LSP，并通过 MCP 接入外部工具。

项目支持 Anthropic、OpenAI、Gemini、Bedrock、OpenRouter、Ollama 等多种模型来源，也支持兼容 OpenAI API 的自定义 Provider。官方项目介绍和最新安装方式可参考 [Crush GitHub 仓库](https://github.com/charmbracelet/crush)。

## 2. 启动方式

在需要处理的项目目录中启动交互界面：

```bash
cd /path/to/project
crush
```

指定工作目录启动：

```bash
crush --cwd /path/to/project
```

开启调试日志：

```bash
crush --debug
```

跳过所有工具权限确认：

```bash
crush --yolo
```

`--yolo` 会自动允许模型提出的工具操作，可能执行 Shell 命令或修改文件，仅应在完全了解风险的隔离环境中使用。

## 3. 非交互运行

直接提交一次任务：

```bash
crush run "解释这个项目的启动流程"
```

通过管道提供内容：

```bash
cat README.md | crush run "将这份文档翻译成中文"
```

临时指定主模型：

```bash
crush run \
  --model ollama/qwen3:8b \
  "检查当前目录中的 Go 代码"
```

如果多个 Provider 中存在相同模型 ID，应使用完整的 `provider/model` 形式避免歧义。

## 4. Session 操作

继续指定 Session：

```bash
crush --session SESSION_ID
```

继续最近一次 Session：

```bash
crush --continue
```

查看当前版本支持的 Session 子命令：

```bash
crush session --help
```

## 5. 查看模型与日志

列出所有已知和已配置模型：

```bash
crush models
```

只搜索包含 Ollama 或某个模型名的结果：

```bash
crush models ollama
crush models qwen
```

标准输出不是终端时，`crush models` 会输出方便脚本处理的 `provider/model` 列表。

查看最近日志：

```bash
crush logs
crush logs --tail 500
crush logs --follow
```

调试 Provider、模型发现或 Ollama 连接时，推荐使用：

```bash
crush --debug
```

# 第三部分：配置文件

## 1. 推荐使用 `crushrc`

当前源码同时支持两种配置格式：

- `crushrc`：推荐格式，是一段由 Crush 内嵌 Shell 执行的 Bash 配置；
- `crush.json`：旧版静态 JSON，仍然兼容，但已经不再是首选格式。

当前分支的配置命令和优先级以项目内的 [配置文档](../config/README.md) 与 [内置 crush-config Skill](../../internal/skills/builtin/crush-config/SKILL.md) 为准。公开 README 中可能仍保留旧版 JSON 示例，使用当前源码时应优先采用 `crushrc`。

## 2. 配置文件位置与优先级

Unix/macOS 上常用位置：

```text
项目级：./.crushrc
项目级：./crushrc
全局：  $XDG_CONFIG_HOME/crush/crushrc
全局：  ~/.config/crush/crushrc
```

项目配置优先于全局配置，离当前目录更近的项目配置优先。若同一个目录同时存在 `crushrc` 和 `crush.json`，两者会合并，冲突字段由 `crushrc` 覆盖，同时日志中会出现警告。

`~/.local/share/crush` 是程序状态和数据库目录，不是用户 `crushrc` 的默认加载位置，不建议手动编辑其中的机器状态文件。

## 3. `crushrc` 基本语法

```bash
#!/usr/bin/env bash

provider add <provider-id> [参数]
model add <provider-id>/<model-id> [参数]
model large <provider-id>/<model-id>
model small <provider-id>/<model-id>

permissions allow view ls grep
option debug true
```

由于它是 Bash，可以使用变量、条件、`source` 和命令替换：

```bash
source "$HOME/.config/crush/secrets.sh"

if [[ "$HOSTNAME" == "development-machine" ]]; then
  option debug true
fi
```

`crushrc` 是可信代码，会以当前用户权限执行。不要直接使用来源不明的配置文件。

# 第四部分：使用 Ollama

## 1. 工作原理

Ollama 对外提供部分 OpenAI-compatible API。Crush 将模型请求发送到：

```text
http://localhost:11434/v1
```

Ollama 官方文档确认 `/v1/chat/completions`、`/v1/responses` 等兼容接口使用该 Base URL，详情参见 [Ollama OpenAI compatibility](https://docs.ollama.com/api/openai-compatibility)。

Crush 当前实现还会使用两类接口：

- `GET /v1/models`：发现当前 Ollama 中已经安装的模型；
- `POST /api/show`：补充每个模型的上下文窗口等原生 metadata。

配置中的 Base URL 可以包含 `/v1`；调用 `/api/show` 时 Crush 会自动去掉末尾的 `/v1`。

## 2. 准备 Ollama

安装 Ollama 后，先确认服务已经启动。不同平台可通过 Ollama 桌面程序启动，或者执行：

```bash
ollama serve
```

拉取一个具备较好工具调用和代码能力的模型。模型名称会随 Ollama 仓库更新，以下仅为示例：

```bash
ollama pull qwen3:8b
```

查看本机已经安装的模型：

```bash
ollama list
```

先直接测试模型：

```bash
ollama run qwen3:8b
```

Ollama 命令的最新说明参见 [Ollama CLI Reference](https://docs.ollama.com/cli)。

## 3. 检查 Ollama API

确认 OpenAI-compatible 模型列表能够访问：

```bash
curl http://localhost:11434/v1/models
```

测试 Chat Completions：

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3.6",
    "messages": [
      {"role": "user", "content": "请只回复 OK"}
    ]
  }'
```

只有这两步正常后，再排查 Crush 配置，可以避免把 Ollama 服务问题误认为 Crush 问题。

## 4. 推荐配置：自动发现模型

创建全局配置：

```bash
mkdir -p "$HOME/.config/crush"
```

在 `~/.config/crush/crushrc` 中添加：

```bash
#!/usr/bin/env bash

provider add ollama \
  --name "Ollama" \
  --type ollama \
  --base-url "http://localhost:11434/v1" \
  --discover-models true
```

如果 Provider 没有手工模型列表，当前实现即使不写 `--discover-models true` 也会自动尝试发现；显式写出该参数更容易让读者理解配置意图。

启动 Crush 后检查：

```bash
crush models ollama
```

自动发现有约 3 秒的整体超时。若 Ollama 第一次启动较慢、远程网络较慢或接口不可达，且又没有手工模型定义，Crush 会跳过这个没有可用模型的 Provider。

## 5. 选择主模型和小模型

`large` 是主要编码模型，`small` 用于摘要等辅助任务：

```bash
provider add ollama \
  --name "Ollama" \
  --type ollama \
  --base-url "http://localhost:11434/v1" \
  --discover-models true

model large ollama/qwen3.6:latest
model small ollama/qwen3.6:latest
```

本地只部署一个模型时，large 和 small 可以使用同一个模型。机器资源充足且安装了多个模型时，可以为 small 选择体积更小、速度更快的模型。

也可以在非交互调用中临时覆盖，而不修改持久配置：

```bash
crush run \
  --model ollama/qwen3:8b \
  --small-model ollama/qwen3:8b \
  "总结当前项目"
```

## 6. 手工注册模型

如果自动发现不可用，或者希望明确声明上下文窗口和输出上限，可以手工配置：

```bash
provider add ollama \
  --name "Ollama" \
  --type ollama \
  --base-url "http://localhost:11434/v1" \
  --discover-models false

model add ollama/qwen3:8b \
  --name "Qwen 3 8B" \
  --context-window 32768 \
  --default-max-tokens 8192 \
  --can-reason true

model large ollama/qwen3:8b
model small ollama/qwen3:8b
```

上下文窗口必须结合实际模型、量化版本、显存/内存和 Ollama 运行参数确定，不应照抄示例中的数字。手工模型 ID 与自动发现结果冲突时，当前实现以用户手工定义为准。

如果使用视觉模型，可以在确认 Ollama 与当前模型确实支持图片输入后声明：

```bash
model add ollama/qwen3-vl:8b \
  --name "Qwen 3 VL 8B" \
  --context-window 32768 \
  --supports-images true
```

## 7. API Key 是否需要配置

本机默认 Ollama 通常不要求 API Key，因此 Crush 允许本地 Provider 的 API Key 留空，日志中可能出现提示但不会仅因此跳过 Ollama。

如果通过要求 Bearer Token 的远程反向代理访问 Ollama，可以设置：

```bash
provider add ollama-remote \
  --name "Remote Ollama" \
  --type ollama \
  --base-url "https://ollama.example.com/v1" \
  --api-key "${OLLAMA_API_KEY:?OLLAMA_API_KEY is required}" \
  --discover-models true
```

也可以使用额外 Header：

```bash
provider add ollama-remote \
  --type ollama \
  --base-url "https://ollama.example.com/v1" \
  --extra-header X-API-Key "$OLLAMA_API_KEY"
```

不要把真实密钥直接提交进项目级 `.crushrc`。应通过环境变量、受权限保护的 `source` 文件或密码管理器提供。

## 8. 旧版 `crush.json` 示例

新配置推荐使用 `crushrc`。如果必须兼容旧版本，可以使用：

```json
{
  "$schema": "https://charm.land/crush.json",
  "providers": {
    "ollama": {
      "name": "Ollama",
      "type": "ollama",
      "base_url": "http://localhost:11434/v1",
      "discover_models": true,
      "models": [
        {
          "id": "qwen3:8b",
          "name": "Qwen 3 8B",
          "context_window": 32768,
          "default_max_tokens": 8192
        }
      ]
    }
  },
  "models": {
    "large": {
      "provider": "ollama",
      "model": "qwen3:8b"
    },
    "small": {
      "provider": "ollama",
      "model": "qwen3:8b"
    }
  }
}
```

旧版公开示例常把 `type` 写为 `openai-compat`，当前源码同时识别专用的 `ollama` 类型。使用 `ollama` 类型的额外好处是 Crush 会调用 `/api/show` 为发现到的模型补充上下文窗口信息。

## 9. Ollama 常见问题

### `crush models ollama` 找不到模型

依次检查：

```bash
ollama list
curl http://localhost:11434/v1/models
crush --debug
crush logs --follow
```

确认模型已 pull、服务端口正确、Base URL 包含 `/v1`，并检查日志中是否出现 `Model discovery failed` 或 `Skipping provider with no models`。

### 连接被拒绝

通常表示 Ollama 没有启动，或者 Crush 与 Ollama 不在同一网络环境。容器中的 `localhost` 指向容器自身，不一定是宿主机；需要改成宿主机可访问地址，并正确配置 Ollama 的监听地址和防火墙。

### Provider 被跳过

自定义 Provider 必须至少满足：

- `base_url` 可解析且非空；
- 存在手工模型，或者模型发现成功；
- `type` 是 Crush 支持的类型。

API Key 对本地 Ollama不是强制条件，但远程代理可能要求。

### 回答正常但不会调用工具

Crush 是 Agent，模型需要可靠支持 tool/function calling。部分小模型虽然可以聊天，却可能无法稳定生成工具调用参数。可以：

1. 换用明确支持工具调用的模型；
2. 升级 Ollama 和模型版本；
3. 先用简单任务测试 View、Grep 等工具；
4. 查看调试日志，区分模型未请求工具和工具执行失败。

### 输出被截断或摘要过于频繁

检查手工声明的 `--context-window` 和 `--default-max-tokens` 是否符合实际模型能力。声明过大可能导致 Ollama 内存不足或请求失败；声明过小会让 Crush 更早触发摘要并限制输出。

### 自动发现超时

当前配置加载阶段对自定义 Provider 模型发现设置约 3 秒总超时。远程 Ollama 延迟较高时，建议关闭自动发现并使用 `model add` 手工登记模型，避免每次启动依赖远程发现。

## 10. 推荐的最小完整配置

适合本机 Ollama 的 `~/.config/crush/crushrc`：

```bash
#!/usr/bin/env bash

provider add ollama \
  --name "Ollama" \
  --type ollama \
  --base-url "http://localhost:11434/v1" \
  --discover-models true

model large ollama/qwen3:8b
model small ollama/qwen3:8b

permissions allow view ls grep
option provider-auto-update false
```

`option provider-auto-update false` 只关闭 Crush 对 Catwalk 默认 Provider 目录的自动更新，不会关闭 Ollama 的本地模型发现。若希望关闭 Ollama 模型发现，应在 Provider 上使用 `--discover-models false`，同时至少用 `model add` 手工注册一个模型。

修改配置后执行：

```bash
crush models ollama
crush --debug
```

确认模型出现在列表中，再进入项目运行 `crush`。

# 参考资料

- [Crush 官方 GitHub 仓库](https://github.com/charmbracelet/crush)
- [Crush 当前配置文档](../config/README.md)
- [Crush 内置配置 Skill](../../internal/skills/builtin/crush-config/SKILL.md)
- [Ollama OpenAI compatibility](https://docs.ollama.com/api/openai-compatibility)
- [Ollama CLI Reference](https://docs.ollama.com/cli)
