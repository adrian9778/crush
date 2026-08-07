# 四、工具系统与钩子系统深度解析

## 4.1 Tools — Agent 的工具链体系

### 4.1.1 工具体系总览

Crush 有 **25+ 内置工具**，分布在 `internal/agent/tools/`。每个工具由三部分组成：

```
tool/<name>/
├── <name>.go       → Tool handler implementation (Go code)
├── <name>.md       → Prompt description visible to LLM ("You can do X with this tool")
└── <name>_test.go  → Unit tests
```

LLM 只看到 `.md`（描述性文本 + JSON Schema），而实际逻辑由 `.go` handler 执行。

### 4.1.2 所有工具分类

| 类别 | 工具名 | 对应文件 | Description |
|------|--------|----------|-------------|
| **文件操作** | View | view.go | Read file content (with offset/limit support) |
| | Write | write.go | Create/overwrite files |
| | Edit | edit.go | Find-and-replace in file (atomic edit) |
| | MultiEdit | multiedit.go | Multiple find-and-replace ops atomically |
| | Download | download.go | Fetch URL → local file |
| **搜索** | Grep | grep.go | regex + literal text search (uses ripgrep) |
| | Glob | glob.go | Path matching pattern search |
| | LS | ls.go | Directory listing as tree structure |
| | Sourcegraph | sourcegraph.go | Cross-repo public code search |
| **Bash 执行** | Bash | bash.go | Shell command execution (with pipe/redirection parsing) |
| | JobOutput | job_output.go | Get stdout/stderr from background shell |
| | JobKill | job_kill.go | Terminate background process group |
| **代码理解** | Definition | lsp_definition.go | Find symbol definition via LSP |
| | References | references.go | Find all references to a symbol | 
| | Rename | lsp_rename.go | Semantic rename-all-refs across project |
| | ReplaceSymbol | lsp_replace_symbol.go | Replace/insert/delete entire functions/classes |
| | CallHierarchy | lsp_call_hierarchy.go | Caller/callee tree visualization | 
| | Symbols | lsp_symbols.go | Document symbol outline (functions, types, methods) |
| | LSPRestart | lsp_restart.go | Restart an LSP client for diagnostics refresh |
| | Diagnostics | diagnostics.go | LSP error/warning/hint display |
| **Web** | Fetch | fetch.go | HTTP fetch (text/markdown/html, max 100KB) |
| | WebSearch | web_search.go | DuckDuckGo web search integration |
| | AgenticFetch | agentic_fetch_tool.go | URL fetching with AI processing/summarization |
| **其他** | Todos | todos.go | Structured task list management (agent-internal) |
| | Question | question.go | Ask user structured questions (yes_no, single/multi_choice, free_text) |
| | CrushInfo | crush_info.go | Show project status info to agent |
| | CrushLogs | crush_logs.go | Read Crush's internal application logs |

### 4.1.3 LSP 工具的实现模式

每个 LSP 工具通过 `lsp_helpers.go` 中定义的辅助函数与正在运行的 LSP server 通信：

```go
// All LSP tools use this pattern:
func definition(...) (result, error) {
    // Step 1: Start/reuse the appropriate LSP client for the file type
    manager.Start(ctx, filePath)  // triggers powernap auto-discovery + spawn if needed
    
    // Step 2: Route to specific LSP method via jsonrpc2 Client
    client := findClientByLanguage(fileLang)  // e.g., gopls → go files
    response, err := client.Call(ctx, "textDocument/definition", params)
    
    // Step 3: Decode and format the result for consumption by the LLM 
    return decodeLSPResponse(response), nil
}
```

**内部调用链路：** `lsp_definition.go` → `lsp_helpers.go:LSPClient.Call(...)` → `Client.ResolveSymbol()` → `jsonrpc2.Client.Call(textDocument/definition, *DefinitionParams) → []*Location | error` → 格式化成人类可读的字符串。

- **LSP call hierarchy**: LSP tool that shows incoming + outgoing calls for a symbol. Uses both jsonrpc2 client and the internal tree structures (caller/callee pairs represented in file:line format).
- **lsp_replace_symbol.go**: The most complex of these tools. It replaces, inserts before/after, or deletes an entire function/method/class by name using LSP document symbols. This is safer than `edit` because it finds symbol ranges via the language server instead of doing approximate text matching.

### 4.1.4 MCP Tools（动态注册的外部工具）

MCP 工具在运行时动态注册为 standard Crush tools，与内置工具无缝集成：

```go
func listMCPTools(server *mcpServer) []fantasy.AgentTool {
    // Call `server.ListTools()` to get the available tools from server
    for _, tool := range server.ListTools() {  // → []*mcptool.Tool from MCP protocol
        go fantasy.NewAgentTool(
            Name:    "mcp_" + tool.Name,           // prefix all MCP tools with "mcp_"
            Handler: handleMCP(tool),               // generic handler → routes to specific server
            Schema:  tool.InputSchema,              // JSON schema for input validation
        )
    }
}

func createResourcesLister(server *mcpServer) fantasy.AgentTool {
	return NewAgentTool(
	    Name: "list_mcp_resources",
        Description: "Lists all MCP server resources",
        Schema: struct{},
   		Halder: func(ctx context.Context, args struct{}) ([]*ResourceItem, error) { ... }
   	)
}

func resourceReader(server *mcpServer) fantasy.AgentTool {
	return NewAgentTool(  // Reads the contents of the named MCP server resource (same pattern)
        Name: "read_mcp_resource",
        Schema: struct{ URI string, MIMEType string }, 
		HandlerFunc: func(ctx context.Context, args ResourceReaderArgs) ([]*ResourceRead, error) {
            resource := resources.Find(args.URI)
            return resource.Read()  // Return the content as text (or base64 for binary)
        },
    )
}
```

### 4.1.5 Bash 工具的实现细节

Bash 是**最复杂的工具之一** — 它不只是执行 `os/exec`，而是包含了一个完整的 shell 执行环境：

```bash
exec "ls -la"
// or with pipes, redirections, etc:
exec "grep -r 'hello' . | head -5 > output.txt"

// Output can also be requested as a command-line response (not in bash tool)
```

Shell execution pipeline:
```
bash.go (handler) 
    │
    ├── shellcommand.Parse()  → parses the input command into AST nodes:
    │   ├── PipeNode("grep -r'hello", "cat")  → pipes between commands  
    │   ├── RedirectionNode(os.Stdout, "path/to/out.txt" ) → file redirection
    │   └── CmdNode("ls" + args[1:]) → individual parts of a command (with quotes preserved)
    │
    ├── coreutils.go (built-in shell builtin cd, echo, mkdir, cat etc.)
    ├── exec_unix.go or exec_windows.go (OS-specific execution)
    │   └── coreutils.Run(ctx, ExecOptions{ 
        Cmd: "my-cmd",
        Args: []string("--arg1", "val1"), 
        Env: ["HOME=/user/vincent", ...],
        Cwd: "/Users/john/path/to/project",  // working directory
        Stdin: string/stdout/stderr pipes for inter-process communication, 
        BackgroundJobID: optional ID for process_group management
    })
    │
    └── persist_message.go (optional): save command result as message history
```

### 4.1.6 Agent-to-Agent Communication（Agent tool）

The `agent` tool is used for multi-agent collaboration — agent A can spawn a new sub-agent in a different directory:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Prompt name (e.g., "task", "reviewer") |
| `prompt` | string | Instructions / prompt to pass to the agent |
| `directory` | string | Working directory for the sub-agent |

Flow: Coordinator dispatches → calls another SessionAgent instance with its own tools/loops — no special magic beyond calling `.Run(subAgent, call)`.

## 4.2 Hooks — 策略引擎

### 4.2.1 Hook system overview

Hooks are user-defined shell commands that fire on agent tool events (currently only "PreToolUse"). They allow you to implement policy rules (security checks, compliance gates, linting).

```go
// Hook config in .crushrc:
hook "pre_tool_use" "/path/to/security_check.sh" --matcher '{"tool": "bash"}'

// In the hook script itself:
# Read tool inputs from stdin as a specific JSON format
cat /dev/stdin  # { tool, input, context }  
echo "deny\nreason: dangerous command\nhalt=false"
exit 2           # Deny! The agent will not call this tool.

# Exit code meanings based on exit codes from running the shell hook script:
#   0-48   = Allow / no opinion / halt
#   49     = HaltExitCode (blocks entire turn)
#   ≥50    = General error
```

### Hook Event Types (currently one type — in a future expansion may have more types of events):

| Constant name | Event Description | When it fires |
|--------------|------------------|---------------|
| `eventPreToolUse` ("pre_tool_use") | Before the tool is executed. **Runs before permission checks**. The hook can deny or modify the input before actual execution happens. The agent sees an "input rewrite" if it changes something. | When a tool call is about to start |

### 4.2.3 Hook result format (hook script → hook engine)

The hook engine parses output from the shell command in one of three forms:

| Output | Meaning |
|--------|---------|
| `"allow"` or `"none"` (or no output / exit 0 | The tool call is allowed to proceed unchanged. |
| `"deny\nreason: <reason_text>"` or just `"deny"` | Block the tool call and return an error message for user to see. |

The `updated_input` JSON field can patch input parameters if needed before execution.

```go
type HookResult struct {
    Decision        string  // "allow" or "deny" or "none" (meaning the hook has no opinion)
    Reason          string  // Deny/halt reason if applicable
    Halt            bool             // Any one can stop this turn from continuing? True only if user says so explicitly.
    UpdatedInput    string   // JSON overrides to apply on the tool call's input before execution. Optional; can be empty. (shallow merge, overwrites keys that already exist in original). 
}
```

### 4.2.4 Hook aggregation across multiple hooks

If you configure multiple hooks matching a given event:

| Input | Output meaning | Explanation |
|-------|----------------|-------------|
| `"deny"` + `"none"` or no match (no match means there was one or more hooks configured for this event and none matched it) | `"denied_by_hook": true` | Deny wins over allow, which wins over none. Halt takes precedence always. |

```go
// Pseudocode from hooks.go:aggregate()
// Multiple Hook results are combined by the hook engine in config order (the registered order).
func aggregate(results []HookResult, origInput string) AggregateResult {
    var decision Decision     // Allow = 1st non-none wins; Deny always beats.
    var halt bool             // Any hook can set this — sticky across all results |
    for _, r := range results {   // Results are processed per configured order:
        switch r.Decision {
        case "allow": if decision != "deny" {decision = "allow"}  // only updates if no deny seen so far
        case "deny": decision = "deny"; reason += "\n" + r.Reason; halt = r.Halt (if true)
        case "none": continue   // No decision by this one -> do nothing 
```

### 4.2.5 Hook metadata propagation to tools

After hooks run, they attach a `hook_result` string field for the message — including info about which hook(s) fired and the overall result (Decision/halt/deny). The UI can display these indicators on UI tool results with `HookCount`, Decision, Halt flag set correctly if any hook(s) blocked or modified something. If no hooks ran it's not populated at all.

### 4.2.6 Hook Runner: parallel execution

```go
func runHooks(ctx context.Context, event string, input string, tools []AgentTool) (AggregateResult, error) {
    // Spawn goroutines for each hook command in parallel via errgroup 
    var g errgroup.Group
    resultCh := make(chan HookResult, len(hooks))   // Each result comes from stdout parser
    
    for _, h := range configuredHooksForEvent {   // Iterate over all hooks registered with this event type:
        go func( hook string) {
            outData, _ := exec hook command (with timeout context.WithTimeout(ctx, 30s))
            r.Parse(stdout output to HookResult structure  
            resultCh <- r
        }(h.Name()) // Run it and send its individual decision back as structured data
    
    wait for all goroutines to finish with errgroup.Wait() before returning aggregated results
```

## 4.3 Loop Detection — Agent's self-defense mechanism

When an agent keeps using the same tool over and over without making progress, loop detection kicks in and intervenes:

```go
func detectLoop(detector *loopDetector) error {
    // Compare the current active tools for this session to its previous tool calls and check if they have repeated more than threshold times. 
    if detector.lastTurn == nil or len(detector.lastTurn.Messages) < 2: 
        return nil   // Can't detect loops with too short history (need at least two turns).  
    
    current := activeAgent.ActiveToolCalls()   // e.g., [Grep(pattern='foo'), View(file='bar.go'), Edit(target='bar.go', old='baz')] 
    
    detector.seen = append(detector.seen, lastTurnMessages...)  // Append previous turn's messages.
    
    for i := range len(seen) - 1:   // Look at the recent history and count how many times each tool was called with similar input arguments:  
        count := 0
        for j := 1; j <= detector.maxAttemptsPerTool; j++ { 
            if equals(current, seen[i+j]): count++
        }
        if count >= detector.maxAttemptsPerTool:
            return fmt.Errorf("You are using the same tool (%s) too many times!", current[0].Name())    // Agent will receive a warning in its response that it's stuck and needs to take different approach
```
