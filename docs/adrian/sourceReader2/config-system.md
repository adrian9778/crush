# 三、配置系统深度解析

## 3.1 配置系统的核心设计思想

Crush 有一个**多来源、分层级、动态合并**的配置管线。这不是简单的 YAML/JSON 文件 —— 而是一个完整的 DSL (Domain-Specific Language) 执行引擎，支持：

- **三种配置来源并存 + deep merge**: crushrc（Bash）、crush.json（JSON）、运行时 override
- **Bash scripting DSL**: 用 `provider()`, `model()`, `mcp()`, `lsp()` 等 Bash 命令写配置
- **Shell expansion**: `$VAR` / `${ENV_VAR}` 在加载时自动解析为环境变量
- **Atomic writes**: 多进程并发安全地更新配置文件（避免 race condition）
- **Hot reload**: crushrc 文件变更时自动重新加载配置

## 3.2 三种配置来源

### Source A: crushrc (Bash 脚本) — 推荐的现代方式

```bash
# ~/.crush/crushrc or workspace/.crushrc (项目级配置)

provider "openai" api_key "$OPENAI_API_KEY" --base-url "https://api.openai.com/v1"
provider "anthropic" api_key "$ANTHROPIC_API_KEY"


# Select which provider and model to use
model "gpt-4o"  # default for large tasks
model "claude-opus-3" --type anthropic  # override for specific providers

# Override default reasoning settings (for large models)
model "claude-sonnet-4" thinking false max_tokens 8192


mcp "my_server" stdio --command "npx" --args "-y my-mcp-server"

lsp "gopls" file_types "go" root_markers "go.mod"

# Hook system: run shell commands before tool execution
hook pre_tool_use "security_check.sh" --matcher '{"tool": "bash"}'


permissions tools_allow bash,view,edit # deny all other tools by default
```

### Source B: crush.json (JSON config) — 兼容旧格式，推荐迁移到 crushrc

```jsonc
{
  "providers": {
    "openai": {
      "id": "openai",
      "name": "OpenAI",
      "base_url": "https://api.openai.com/v1",
      "api_key": "$OPENAI_API_KEY"
    },
    "anthropic": {
      "id": "anthropic",
      "name": "Anthropic",
      "type": "anthropic",
      "api_key": "$ANTHROPIC_API_KEY"
    }
  },
  "models": {
    "large": {
      "provider": "openai",
      "model": "gpt-4o",
      "max_tokens": 16384,
      "temperature": 0.7
    },
    "small": {
      "provider": "gpt-4o-mini"
      "provider": "anthropic",
      "model": "claude-haiku-3",
    }
  },
  "mcp": {
    "my_server": {
      "type": "stdio",
      "command": "npx",
    "lsp": {
      "gopls": { 
        "file_types": ["go"],
        "root_markers": ["go.mod"]
    }
  },
  "permissions": {
    "skip_permission_requests": true,
    "tools_allow": ["view", "edit", "write", "bash"]
  },
  "skills": {
    "enable": ["awesome_skill"],
    "disable": ["experimental_tool"],
    "include": ["my_custom_skills/"]
  }
  }
}
```

### Source C: CLI flags / env var overrides — 最高优先级

这些设置在配置文件解析完成**之后**应用作为最后一个 step，可以覆盖所有来源的任何值：

| CLI Flag | Env var Equivalent | Description |
|----------|-------------------|-------------|
| `--provider "openai"` | `_PROVIDER_` | Which provider to use for the default model |
| `--model gpt-5.6-luna` | `-MODEL_` | Model ID to use instead of config one |
| `CRUSH_SERVER_IDLE_TIMEOUT=120` | - | Server idle-timeout before shutdown (seconds) |
| `CRUSH_SERVER_DETACH_GRACE=15` | - | Grace period after client detaches (seconds) |

## 3.3 crushrc DSL 语法详解：内置命令（builtins）

crushrc 是一个 Bash 脚本，通过 shell builtins 暴露了一个配置 DSL。每个 builtin 注册为 `shell.RegisterBuiltin()` 时会在执行 crushrc 时将它们挂载起来。它们的实现都在 `shellconfig/` 下面。

### Provider Builtin

```bash
provider "<id>" api_key "$KEY" [with options]
  --base-url "<url>"    → override API endpoint
  --discover-models     → auto-discover model list from provider APIs
  --flat-rate           → no-cost mode for subscriptions
  --extra-headers 'X-Custom: value'    → custom HTTP headers (supports shell expansion)
```

Example of provider built-in:
```bash
provider "openai" api_key "$OPENAI_API_KEY"

# The full provider config is constructed from the bash command line. Each flag adds
# a key-value pair to a ProviderConfig map[string]any that gets merged with defaults.
```

### Models Builtin  
```bash
model "gpt-4o" [with options]
  type anthropic | openai | azure | bedrock | google | vercel
  max_tokens 8192    → set maximum output tokens (max 200,000)
  temperature 0.7    → sampling temperature ∈ [0.0, 1.0]
  think true         → enable Anthropic thinking/reasoning mode  
```

### MCP Builtin 
```bash
mcp "<name>" stdio --command "npx" -args "-y my-mcp-server"
mcp "<name>" sse --url "http://localhost:3000/mcp"
  → type: [stdio \| sse \| http]  (default: stdio) 
```

### LSP Builtins
```bash
lsp "<id>" file_types "go" root_markers ".git go.mod"
  --command "gopls"     → executable name for server
  --args "-c config.json"  → extra args
  --env "GOMOD=/path/to/go.mod"  → environment variables for this server  
```

### Permission Builtins
```bash 
permissions skip        → disable all permission checks (auto-approve everything)
permissions deny all    → deny all tool calls by default
permissions allow <tools>  → explicitly allow specified tools
# Example: permissions allow bash,edit,view → Only these three are allowed; others blocked
```

### Hook Builtins
```bash
hook "<event>" "<script>" [--matcher '{"tool": "bash"}']
  event = only "pre_tool_use" so far
  matcher (optional) JSON filter — only fire this hook when tool matches the pattern  
```


### Option / Global Setting Builtin

```bash
options data_directory ".crash"     → override SQLite data directory path  
options progress true              → show spinner in non-interactive mode

# Or directly in crush.json:
#   "options": { "data_directory": "...", "progress": true, "auto_lsp": true }
```

## 3.4 Config Merge Pipeline — 精确合并逻辑

```go
// config/load.go: MergeConfigStore() 
func MergeConfigStore(shellJSON, rawJSON, overrides) *ConfigStore {
    // Step 1: Parse crushrc (already converted to JSON by shellconfig/loader)  
    cfg := fromShellJSON(shellJSON)  // Base layer
    
    // Step 2: Deep-merge with crush.json (if exists)
    for key in rawJSON: merge(cfg, rawJSON[key])  // Shallow merge for top-level keys; deep for ProviderConfig/MCP/LSP maps
    
    // Step 3: Apply CLI/env overrides last (highest priority)
    for key in overrides: applyOverride(cfg, key, value)
    
    return cfg  
}

// Precedence order (lowest → highest):
//   1. shellconfig base configuration (crushrc)
//   2. crush.json file values                    ← deep merges into shellconfig
//   3. config overrides from CLI flags/env vars  ← shallow merge, wins always
```

### Atomic Writes

`atomicwrite.go` 提供多进程安全的配置文件写入：写到一个临时文件，然后 `os.Rename()` (原子操作)。Unix/Windows 分别实现以处理不同平台的锁需求。

## 3.5 Auto-Discovery of Models (`config/provider.go`)

Crush 默认自动从 LLM provider API 中获取可用模型列表：

```go
func (p *ProviderConfig) ToProvider() catwalk.Provider {
    // If AutoDiscoverModels is nil/true AND Models list empty → fetch from /v1/models endpoint
    // Otherwise return only user-configured models
}

// For providers without auto-discovery endpoints: discovered models are 
// merged with user-specified ones (user-specified always take precedence)
```

## 3.6 Scope-based Configuration

Different config values can be scoped to different contexts:

| Scope Type | Description |
|-----------|-------------|
| **User** | `~/.crush/crushrc` — Global defaults across all workspaces  
| **Project** | Working directory `.crushrc` or `crush.json` — Workspace-specific overrides
| **Session** | Per-session model/provider overrides (set dynamically during conversation via TUI)

## 3.7 JSON Schema (`schema.json`)

Crush 生成完整的 JSON Schema 用于 IDE 自动补全和验证：

```bash
crush schema > /tmp/crush.schema.json   # Output current version's JSON schema
```

