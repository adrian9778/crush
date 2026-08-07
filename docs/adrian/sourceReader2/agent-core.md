# 二、Agent 核心引擎深度解析

## 2.1 Agent 架构总览

Crush 的 agent 系统是它的核心。它不是简单的 OpenAI chat completion wrapper —— 而是一个复杂的 **agent loop engine**，管理着：

- 双模型策略（大模型处理复杂任务，小模型处理快速响应和标题生成）
- Tool orchestration（动态工具链组装、权限检查、hook 拦截）
- Loop detection（检测 Agent 是否陷入重复调用）
- Auto-summarization（当上下文窗口过大时自动压缩对话历史）
- Prompt queuing（队列机制，确保 agent 忙时新请求被挂起而非丢弃）

```
┌─────────────── Coordinator ──────────────────────────┐
│                                                        │
│   agents map:                                        │
│     "coder" → SessionAgent                           │
│     "task"  → SessionAgent (optional second agent)   │
│                                                        │
│   currentAgent = SessionAgent (default: coder)       │
│                                                        │
│   Key methods:                                       │
│   ├── Run(ctx, sessionID, prompt)                    │
│   ├── RunAccepted(ctx, accept, sessionID, prompt)    │
│   ├── Cancel(sessionID)                              │
│   ├── QueuedPrompts(sessionID)                       │
│   └── Summarize(ctx, sessionID)                      │
├───────────────────────────────────────────────────────┤
│                                                        │
│         ═══════ SessionAgent ═══════                 │
│                                                        │
│   sessionAgent:                                      │
│     largeModel: Model (complex tasks)              │
│     smallModel: Model (fast title/gen/reasoning)  │
│     systemPrompt: string (Go template built)        │
│     tools: []fantasy.AgentTool (dynamic list)       │
│                                                        │
│   messageQueue: map[sessionId][]Call (pending queue) │
│   activeRequests: map[sessionID]*activeCancel         │
│   dispatchMu:     map[sessionID]*sync.Mutex           │
│   cancelMark:     map[sessionID]uint64               │
│                                                        │
│   Key methods:                                        │
│   ├── Run(call SessionAgentCall) → *Result, error    │
│   ├── runAgent(ctx, systemPrompt, tools, messages)  │
│   ├── summarize(ctx, sessionID, opt, authRefreshCb) │
│   └── generateTitle(ctx, sessionID)                │
└───────────────────────────────────────────────────────┘
```

## 2.2 Coordinator — 多 Agent 管理器

Coordinator 是顶层控制器，负责：

- 初始化和生命周期管理
- 在 agent instances（按 agent name + model pair 区分）之间切换
- 处理 queued prompts when busy
- 维护 skills tracker（active vs all discovered skills）

### 2.2.1 Coordinator 初始化流程

```
NewCoordinator(ctx coordinatorOpts)
    │
    ├─ 1. Skill Discovery (pre-fetched by caller, but fallback inline)
    │     └── discoverSkills(cfg) → (allSkills, activeSkills)
    │         ├── Dedup skills by name
    │         └── Filter: include list, disable list
    │          → skillTracker = NewTracker(activeSkills)
    │
    ├─ 2. Prompt Loading
    │     └── coderPrompt(templates/coder.md + working dir)
    │
    ├─ 3. Build SessionAgent
    │     └── c.buildAgent(ctx, prompt, agentConfig, false, skills...)
    │         ├── Get config for the specified Agent (coder or task)
    │         ├── loadProviderAndModels(session, cfg.Config())
    │         │   → large: model from "large" section of configuration
    │         │   → small: model from "small" / auto-discovered default
    │         └── Create SessionAgent instance
    │
    └─ 4. Register agent in c.agents map

Key fields of coordinator struct:
    currentSession map[string]SessionName (active session names)
    agents         map[string]SessionAgent  (initialized at first Run call)
```

### 2.2.2 Coordinator.Run() — 一次 Agent Turn 的入口

```go
func (c *coordinator) Run(ctx context.Context, sessionID string, prompt string, attachments ...message.Attachment) (*fantasy.AgentResult, error) {
    // 1. If first call ever and not yet initialized: init CoderAgent
    if c.currentAgent == nil && err := c.InitCoderAgent(ctx); err != nil {...}

    return c.currentAgent.Run(ctx, SessionAgentCall{
        SessionID: sessionID,
        Prompt:    prompt,
        Attachments: attachments,
    })
}
```

### 2.2.3 Coordinator 双模型策略

Crush 使用两个 LLM model —— 一个 large（复杂任务），一个 small（快速响应）：

| 模型类型 | 用途 | 典型配置 |
|----------|------|----------|
| **Large Model** | 实际的工具调用、代码生成、长对话推理 | gpt-5.6-luna, claude-opus-4 (reasoning) |
| **Small Model** | Title generation, summarization, quick answers, thinking mode prompts | gpt-4o-mini, claude-haiku-4 |

配置示例：
```jsonc
"models": {
    "large":  { "provider": "openai", "model": "gpt-5.6-luna" },
    "small":  { "provider": "anthropic", "model": "claude-haiku-4" }
}
```

### 2.2.4 SessionAgent — 单会话 Agent 引擎

`Run(call)` 是整条 agent pipeline 的核心函数。它内部走如下步骤：

```
SessionAgent.Run(call)
    │
    └── run(ctx, call)          // internal method with full logic
        │
        ├── Phase 1: Pre-run Setup
        │     ├─ loadContextFiles(workingDir) → context files (AGENTS.md etc.)
        │     ├── buildSystemPrompt(workingDir, contextFiles) → Go template rendering
        │     ├── mcpTools list
        │     ├── skillTracker.activeSkills()
        │     └── tools = filterOutInteractiveOnly(calls, system/tools, skills)
        │
        ├── Phase 2: Session Management
        │     ├─ getOrCreateSession(call.SessionID) → SQLite
        │     ├── loadExistingMessages(session.ID)
        │     ├── maybeAutoSummarize(messages) → trigger if context window > threshold
        │
        └── Phase 3: Model Interaction
              ├── dispatchWithTimeout(ctx, timeout)
              │     ├─ runAgent(systemPrompt, tools, messages, largeModel.Call())
              │     └→ Fantasy AgentResult or error
              ├── handleFantasyStream(result) → text delta events pubsub
              ├── checkLoopDetection(result) → retry if loop detected
              ├── maybeFallbackSmallerModel(err) → auto-switch to small model
              └── persistMessages(sessionID, result)
```

### 2.2.5 contextFiles 加载机制

Crush 从工作目录自动加载上下文文件（project-level instructions）：

```go
// defaultContextPaths (config/config.go)
defaultContextPaths = []string{
    ".github/copilot-instructions.md",
    ".cursorrules",
    ".cursor/rules/",
    "CLAUDE.md",
    "CLAUDE.local.md",
    "GEMINI.md",
    "gemini.md",
    "crush.md",
    "crush.local.md",
    "AGENTS.md",
    ...
}

// Loaded during run: prompt.LoadContextFromPaths(paths + config overrides) → contextText
```

## 2.3 Fantasy LLM 集成层

Fantasy (`charm.land/fantasy`) 库是 Crush 与各类 LLM 之间的**统一适配层**：

### Fantasy API 抽象

```go
// Fantasy provides (for each supported vendor):
type LanguageModel interface {
    Call(ctx context.Context, req ModelCall) (*AgentResult, error)  // non-streaming
    CallWithStream(ctx context.Context, req ModelCall) (*AgentStreamingResult, error)  // streaming
}

type AgentTool struct {            // What the LLM can call: name(description), schema(JSON), handler(func(...))
    Name        string
    Description modelDescription    // Text shown to LLM (injected into system prompt)
    Type        modelFunctionType      // OpenAI Function type / Anthropic Tool spec
    Handler     AgentToolHandlerFunc  // Go function that gets called when LLM calls this tool
}
```

### Fantasy Provider Registry（fantasy providers mapping）

| fantasy.Type | Go Package | API Protocol | Authentication | Note |
|--------------|-----------|-------------|---------------|-----|
| `openai` | fantasy/providers/openai | OpenAI Chat Completions | API Key / OAuth | Default for most LLMs |
| `anthropic` | fantasy/providers/anthropic | Anthropic Messages | API Key | Native tool_use format |
| `azure` | fantasy/providers/azure | Azure OpenAI | API Key | OpenAI-compatible endpoint |
| `bedrock` | fantasy/providers/bedrock | AWS Bedrock | AWS Credentials | + AWSAuthRefresh (shell command to refresh creds) |
| `google` | fantasy/providers/google | Google Gemini | API Key / OAuth | Native Gemini SDK |
| `openrouter` | fantasy/providers/openrouter | OpenRouter gateway | API Key | Routes to multiple vendors |
| `vercel` | fantasy/providers/vercel | Vercel AI SDK | API Key | Streaming text protocol |

### Fantasy ModelCall 请求结构

```go
type ModelCall struct {
    Messages              []Message                // Conversation history + assistant messages
    SystemPrompt          string                   // Go-template rendered system prompt
    ProviderOptions       OpenAIFunctionParameters // Optional extra provider-specific params
    ModelID               string                   // e.g. "gpt-5.6-luna"
    MaxTokens             int                      // Max output tokens
    Temperature           *float64                 // Sampling temp [0.0, 1.0]
    TopP                  *float64                 // Nucleus sampling threshold
    TopK                  *int64                   // (for Anthropic-style) top-k parameter
    FrequencyPenalty      *float64                 // Penalty for repeating tokens
    PresencePenalty       *float64                 // Penalty for new topics
    Tools                 []AgentTool              // Dynamic tool list for current turn
    ProviderOptions       map[string]any           // Custom parameters specific to each provider
    ResponseFormat        string                   // text, json_object, etc.
}
```

## 2.4 Session Agent Run — 核心执行循环详细分析

我们逐步拆解 `sessionAgent.Run()`：

### Phase 1: Pre-run Setup (call preparation)

```go
// Load context files from workingDir
contextFiles := prompt.LoadContextFromPaths(
    c.cfg.WorkingDir(),     // → .github/copilot-instructions.md, CLAUDE.md, etc.
    defaultContextPaths,    // → AGENTS.md, crush.md, etc.
)

// Build system prompt (Go template + runtime args)
systemPrompt := a.configSystemPrompt(workingDir, contextFiles, skills)
// Template: internal/agent/templates/coder.md.tpl (rendered with Go templates)

// Load MCP tools from initialized server list
mcpTools = loadMCPTools()  // dynamic list from all connected MCP servers
```

### Phase 2: Session Management

```go
// Get or create session in SQLite
session, err := c.getSess(ctx, call.SessionID)
if session.ID == "" {
    session, err = c.createDefaultSession(ctx)
} else {
    if !c.sessions.IsAgentToolSession(session.ID) {
        return fmt.Errorf("cannot continue an agent tool session: %s", session.ID)
    }
}

// Load existing messages for this session from the database
existingMessages := loadExistingMessages(c.messages, session.ID)

// Auto-summarize check (see section 2.6 below)
if shouldAutoSummarize(existingMessages, largeContextWindowThreshold) {
    c.summarize(ctx, session.ID)
}
```

### Phase 3: Model Interaction — runAgent()

```go
func (a *sessionAgent) runAgent(
    ctx context.Context,
    systemPrompt string,
    model fantasy.ModelProvider,
    tools []fantasy.AgentTool,
    messages []fantasy.Message,
) (*fantasy.AgentResult, error) {
    
    // Create the fantasy agent with our configured model
    agent := fantasy.NewAgentWithModel(model, fantasy.Config{
        SystemPrompt:  systemPrompt,
        MaxTokens:     call.MaxOutputTokens(default),
        Temperture:    call.Temperature(0.7 default),
        TopP:          call.TopP(0.9 default),
        ...
    })

    // Execute the agent run (streaming)
    result, err := agent.CallWithStream(ctx, fantasy.ModelCall{
        Messages:  messages,
        Tools:     tools,
        SystemPrompt: systemPrompt,
        ProviderOptions: a.config.ProviderExtraHeaders / ExtraBody,
    })

    if err == nil {
        return result, nil
    }

    // Check for auth error (HTTP 401) — trigger OnAuthRefresh retry chain
    if _, isAuthErr := fantasy.AsProviderError(err); isAuthErr && call.OnAuthRefresh != nil {
        refreshErr := call.OnAuthRefresh(ctx, err)
        if refreshErr == nil {
            // Retry with new token
            return a.runAgent(ctx, systemPrompt, model, tools, messages)
        }
    }

    // Check for "model not found" — fallback to small model? 
    if isModelNotFoundError(err) {
        result, fallback := maybeFallbackSmallerModel(ctx, call)
        if fallback { return result, nil }
    }

    // Check loop detection: did the agent repeat the same tool too many times?
    if err = detectLoop(result); err != nil {
        result.ErrMessage = "You are stuck - try a different approach"
        return a.runAgent(ctx, systemPrompt, model, tools, messages)  // retry
    }

    return result, err
}
```

## 2.5 Loop Detection（循环检测）

当 Agent 在连续多次 turn 中使用相同的 tool calls 做相似的事情时，它可能陷入循环。Loop detection 自动干预：

```go
type loopDetector struct {
    currentAgent   fantasy.AgentName       // Current active agent (e.g., "coder")
    maxAttemptsPerTool int                 // Max consecutive uses of the same tool

    seen []fantasy.Message     // History for cycle detection
    lastTurnMessages []*fantasy.Message  // Last turn's messages for comparison
    
    // Tracks how many times each specific tool name has been called
    // without changing its input args, exceeding max attempts.
}

func detectLoop(detector *loopDetector) error {
    // Compare the current agent's most recent tool calls to its previous ones
    if len(seen) > 0:
        // Check if we're repeating the same pattern
        for i := range detector.seen[:len(seen)-3]:  // Look at last 3 turns + current
            if equals(tool, detector.seen[i].ToolCall()):
                attempts++
                if attempts >= detectors.maxAttemptsPerTool:
                    return fmt.Errorf("loop detected - tool %s was used more than %d times", tool.Name(), maxAttemptsPerTool)
```

## 2.6 Auto-Summarize（自动压缩）机制

当对话上下文窗口接近模型限制时，Crush 自动触发 summarizer：

**触发条件：**

| Condition | Threshold (from agent.go lines 53-57) | Description |
|-----------|--------------------------------------|-------------|
| `largeContextWindowThreshold` | **200,000 tokens** | Total context + buffer exceeds size limit |
| `largeContextWindowRatio` | **< 0.2 of window** | Remaining context < 20% of model's max windows |

**执行流程：**

```go
func (a *sessionAgent) summarize(ctx context.Context, sessionID string, opt fantasy.ProviderOptions, authRefreshFn func(*fantasy.ProviderError)) error {
    // Load messages for the session
    messages := loadMessages(sessionID) 

    // Build summary prompt from template (templates/summary.md) + conversation history
    summaryMsg = prompt.Summary(messages, systemPrompt)  // Rendered using Go templates
    
    return a.runAgent(
        ctx,          // use small model via "fallback" configuration option
        summaryMsg,   // "Please summarize the following conversation..."
        messages,     // only last N messages to summarize (not entire history)
        opt,          // Provider options (maxTokens set for compact output)
        authRefreshFn,
    )
    │   └── On completion → replace session messages with:
    │       [system_prompt]
    │       [message: "Previous summary..." (the generated summary)]
    │       [user_message ...]  // keep recent ones
    │       [assistant_message ...]  
}
```

### 2.7 Prompt Queuing（请求排队机制）

当 Agent busy (processing a turn) 时，新的用户请求不会被忽略——它们会被排队：

```go
func (a *sessionAgent) isBusySession(sessionID string) bool {
    // Check if there's an active request OR any queued requests for this session
    hasActive := isActiveRequest(session)        // activeRequests.Get(sessionID) != nil
    hasQueued := queueLength(session) > 0       // messageQueue.Get(sessionID) == len([]Call{})
    return hasActive || hasQueued
}

func (a *sessionAgent) EnqueuePrompt(call SessionAgentCall): {
    if !isBusySession(sessionID) {
        // No need to queue - just run it directly!
        a.runWithDispatch(ctx, call)
        return
    }
    
    // Queue: add to pending list + notify pubsub for UI update
    messageQueue.Put(sessionID, append(currentQueue, call))
    sendQueuedEvent(sessionID)  // UI shows "X more in queue"
}

// When an active run completes (in deferred cleanup), it automatically processes next queued item:
func onRunComplete(queued []SessionAgentCall): {
    if len(queued) > 0:
        next = queued[0]
        a.run(ctx, next)  // Process the next queued prompt
    }
}
```

### 2.8 Accept Queue — Client/Server fire-and-forget 模式

当 `crush run` 在 server 模式下调用时，需要一个可靠方式来确认一个 turn 完成：

```go
type AcceptedRun struct {
    SessionID     string
    acceptSeq     uint64          // monotonic counter, compared by cancelMark below
    closeMu       sync.Mutex      // protect the transition: Accepted -> cancelled / active
}

// When user sends command from client → server:
c.BeginAccepted(sessionID)  // returns *AcceptHandle + increment acceptedRuns[sessionID]++
                              // This prevents race between cancel and startDispatched turns

// When turn completes:
acceptedHandle.close()  // decrements acceptedRuns[sessionID]--
```

## 2.9 Agent Title Generation（会话标题生成）

每次新 session，Crush 会调用 small model 生成简洁的标题：

```go
func (a *sessionAgent) GenerateTitle(ctx context.Context, sessionID string, userPrompt string) {
    // Extract first line from agent's response as title candidate
    titleRaw := extractFirstLine(result.Content)  // e.g. "Helping you build a REST API" → "Building a REST API"

    // Clean up with template-based title prompt + remove thinking tags <think>
    title = cleanTitle(titleRaw)       // Regex removal: </?think>, whitespace stripping
    
    // Update session.title in SQLite via c.sessions.UpdateSession(ctx, sessionID, newTitle)
    c.sessions.UpdateSessionTitle(sessionID, title)
    
    // Publish update events: pubsub.Publish(UpdatedEvent, UpdateSessionTitleMsg(title))
    a.events.Publish(RestartAgentTitles(), title)  // UI knows to show the new title in sidebar
}
```
