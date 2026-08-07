# 七、LSP 与 MCP 集成深度解析

## Part A: LSP (Language Server Protocol) Manager

### 7.1.1 LSP Manager 架构概览

Crush 的 LSP Manager 基于 **powernap**（Charm 自研库，类似 Neovim nvim-lspconfig + auto-start）实现懒加载。核心原则：**只在需要时才启动语言服务器**（根据文件类型自动匹配），不需要时保持休眠。

```
┌─────────────── lsp.Manager ───────────────┐
│                                            │
│ clients:      map[string]*Client           │  Running LSP client instances
│ unavailable:  map[string]time.Time         │  Failed servers with retry delay
│ cfg:          *config.ConfigStore          │
│ manager:      *powernap.Manager            │  Base server definitions from powernap defaults
│ callback:     func(name, client)           │  Notification when client starts (used by Coordinator to register LSP tools)
│ lookPath:     func(string)(string,error)  │  exec.LookPath replacement for testing
└────────────────────────────────────────────┘

核心方法：
├── Start(ctx, path)        — According to file type, lazily starts the appropriate LSP server
├── TrackConfigured(ctx)    — Notify UI about configured-but-not-initialized servers (without starting them)
├── Clients()               — Return all active client instances
└── NewManager(cfg)         — Constructor; merges defaults from powernap with user-configured configs
```

### 7.1.2 LSP Start Flow — Step by Step

When the user opens or edits a file:

```go
func (m *Manager) Start(ctx context.Context, filePath string) {
    // Guard: file must be under working directory
    abs := filepath.Abs(filePath)
    if !hasPrefix(abs, m.cfg.WorkingDir()) {
        return   // Reject — outside workspace scope
    }

    // Try every known server concurrently (from powernap defaults + user config)
    for name, srv := range manager.GetServers() {
        go func(name string, srv *ServerConfig) {
            // 1. Auto-start permission: if not user-configured, requires Options.AutoLSP flag
            isUserCfg := m.isUserConfigured(name)
            autoLSP := m.cfg.Config().Options.AutoLSP
            if !isUserCfg && (autoLSP == nil || *autoLSP == false) {
                return   // Not configured by user AND auto-LSP disabled
            }

            // 2. Check existing client in the map (another goroutine might have started it already)
            if c, ok := m.clients.Get(name); ok {
                switch c.GetServerState() {
                  case StateReady, StateStarting, StateDisabled: return
                }
            }

            // 3. Decision tree for actual startup
            if isUserCfg {
                // User explicitly configured; only start if this file matches
                if !handles(srv, abs, m.cfg.WorkingDir()) { return }
            } else {
                // Auto-discovered default: more conservative — skip generic names
                if !m.canAutoStart(name, abs, m.cfg.WorkingDir(), srv) { return }
            }

            // 4. Double race-check before actually creating client
            if c, ok := m.clients.Get(name); ok && c.IsReadyOrStarting() { return }

            // 5. Finally: create and launch the actual LSP process
            client, err := lsp.New(
                name, cfg, m.cfg.Resolver(), m.cfg.WorkingDir(), debugMode,
            )
            m.clients.Put(name, client)
            m.callback(name, client)   // Coordinator adds LSP tools to agent!
        }(name, srv)
    }
}
```

### 7.1.3 canAutoStart — Auto-Start Safety Filters

Generic/common commands that shouldn't auto-start as LSP servers (they could be build tools, runtimes, etc.):

```go
var skipAutoStartCommands = map[string]bool{
    "buck2", "buf", "cue", "dart", "deno", "dotnet", "dprint",
    "gleam", "java", "julia", "koka", "node", "npx", "perl",
    "plz", "python", "python3", "racket", "rome", "rubocop",
    "scarb", "solc", "stylua", "swipl", "tflint",
}
// Also skips commands with too broad file-type coverage (e.g., a formatter that works on every language).
```

### 7.1.4 Root Marker Matching

User-configured LSPs match projects via root markers:

```go
// User's crushrc:
lsp "gopls" root_markers ".git go.mod"

handles(srv, filePath, projectRoot):
    For each marker in srv.RootMarkers:
        Walk parents of filePath
        If any parent directory contains the marker file/directory → match!
```

### 7.1.5 LSP Client and Protocol

`internal/lsp/client.go` wraps a jsonrpc2 connection to a language server's child process (stdio):

```go
type Client struct {
    name         string
    state        ServerState   // Unstarted → Starting → Ready / Error
    conn         *jsonrpc2.Conn  // Live JSON-RPC channel with the LSP server
    diagnosticsCallback func(Diagnostics)  // Called when diagnostic messages arrive
    Methods: Hover, Definition, References, Rename, ... (all standard LSP endpoints)
}
```

### 7.1.6 How LSP Tools Get Registered to Agent

In `app.go` New():

```go
app.LSPManager.SetCallback(func(name string, client *lsp.Client) {
    if client == nil {
        updateLSPState(name, lsp.StateUnstarted, nil, nil, 0)   // Notify UI only
        return
    }
    client.SetDiagnosticsCallback(updateLSPDiagnostics)          // Forward diagnostics to UI in real-time
    updateLSPState(name, client.GetServerState(), nil, client, 0)

    // Coordinator uses this callback to register LSP tools (lsp_definition, lsp_references, etc.) with the agent!
})
```

Also: `go app.LSPManager.TrackConfigured(ctx)` starts a background goroutine that announces all configured-but-not-started servers to the UI so they appear in the sidebar as "ready but inactive."

### 7.1.7 Auto-Summary of LSP Design Decisions

LSPs start **automatically** when their target file type matches (no manual intervention needed). Once started, they stay alive for the lifetime of the workspace (they are re-used for new files). The powernap integration handles power management — if an LSP becomes unresponsive it is automatically restarted on next access.

---

## Part B: MCP (Model Context Protocol) Integration

### 7.2.1 What is MCP and Why?

MCP (Model Context Protocol) by Anthropic enables Agent-to-external-service communication through a standardized protocol. Crush implements a full MCP **client** that discovers, connects to, and registers tools from MCP servers as first-class agent tools:

```
Agent Turn: "Get the weather for Tokyo"
    │
    ▼ LLM decides to call tool: mcp_weather_server_get_weather
    │
    ▼ Agent dispatches through mcp/tools.go handler
    │
    ▼ Routes to specific MCP server (registered during startup)
    │
    ▼ Server runs "npx weather-cli get --city Tokyo"
    │
    ▼ Result: {"temp": 22, "condition": "sunny"}
```

### 7.2.2 MCP Lifecycle — 4 Phases

**Phase 1: Arming the init state** (called before any agent runs)
```go
func ArmInit() { mcpInitialized = make(chan struct{}) }
```

**Phase 2: Background initialization** (spawned in app.go New())
```go
go mcp.Initialize(ctx, app.Permissions, store)

// Inside Initialize():
For each MCP config in user's settings:
    Create server (stdio process or SSE/HTTP connection)
    Call server.initialize(ctx)  → sends JSON-RPC "initialize" request
    Wait for response with capabilities + tool list
    If successful: register all tools as fantasy.AgentTool instances (prefixed "mcp_")
```

**Phase 3: Blocking wait point** (for non-interactive modes)
```go
func WaitForInit(ctx context.Context) error {
    return initializedWaiter()   // Blocks until init goroutine finishes. timeout on top of ctx cancellation.
}
// Used by: App.RunNonInteractive — waits for all MCP tools to be ready before starting the agent run.
```

**Phase 4: Runtime invocation**
When an `mcp_*` tool is called by the LLM:
```go
func handleMCPCall(ctx context.Context, calls []ToolCall) ([]*ToolResult, error) {
    For each call:
        Parse "mcp_serverName_toolName" → extract server name and tool name
        Lookup server from global map (populated during phase 2)
        Forward call to that server's CallTool method via MCP protocol
        Return result aggregated back to agent framework
}
```

### 7.2.3 Three Connection Types

| Type | Config format | Use case |
|------|--------------|----------|
| **stdio** | `command` + `args` | Local CLI tools that communicate via stdin/stdout JSON-RPC (most common) |
| **SSE** | `url` with type "sse" | Remote HTTP server streaming MCP events back using Server-Sent Events |
| **HTTP** | `url` with type "http" | Standard request/reply MCP over plain HTTP POST endpoints |

Example in crushrc:
```bash
mcp "my_server" stdio --command "npx" --args "-y my-mcp-server"
mcp "remote_api" sse --url "http://localhost:3000/mcp"
```

### 7.2.4 Resources and Channels in MCP

Beyond tools (which the LLM calls to get data), MCP also exposes:

- **Resources**: Static/read-only data sources (like documentation, config schemas) — browseable via `list_mcp_resources` and readable via `read_mcp_resource`
- **Channels**: Server-to-client notification streams (push-style updates when certain events occur remotely)

### 7.2.5 MCP Close Sequence

During app teardown:

```go
func Close(ctx context.Context) error {
    errors collected during shutdown of each connected MCP server
    return combined error of all failures
}
// Called from app.cleanupFuncs during Shutdown()
```

---

## Part C: LSP vs MCP Comparison

| Dimension | LSP (Language Server) | MCP (Model Context Protocol) |
|-----------|----------------------|------------------------------|
| **Protocol** | Microsoft JSON-RPC over stdio | Anthropic's open standard across platforms |
| **Purpose** | Code understanding — go-to-definition, rename all refs, linting | External data/tool access — APIs, databases, CLI tools |
| **Discovery** | Auto-discovery via powernap + root markers | Manual configuration only — must specify command / URL explicitly in crush.json or crushrc |
| **Lifecycle** | Lazy start by file type (on-demand) | Explicitly configured; all servers initialized at startup |
| **Tool registration** | Automatic through Coordinator's callback from LSP manager | Automatic through Initialize() goroutine that scans all discovered MCP servers during app.New() and registers their available tools dynamically as fantasy.AgentTools with name prefix "mcp_" + server_name + "_" + tool_name_from_server_listing! |
