# 五、后端与 Workspace 生命周期深度解析

## 5.1 Backend 层概述

`internal/backend/` 是整个系统的**业务逻辑核心**，完全脱离 HTTP/WS 协议独立。它被 `server/`（HTTP REST layer）和 `workspace/`（ACP protocol）两层消费：

```
                    HTTP REST endpoints
                    (server/controllerV1)
                              │
                   ┌──────────┴─────────┐
                   ▼                    ▲
           backend.Backend         ACP protocol adapter
                   │                    ▲ (also uses Backend methods)
      ═══════ Transport-agnostic core ══════
                   │
              Workspace[App]
                   │
            app.App (app layer)
```

### 5.1.2 Backend struct 核心字段

```go
type Backend struct {
    workspaces *csync.Map[string, *Workspace] // All active workspaces keyed by ID 
    pathIndex     map[string]string   // resolved abs path → workspace ID for dedup (same dir = same workspace)
    pending         int                // # of CreateWorkspace calls in slow-init phase (not yet registered)
    shutdownTimer   *time.Timer        // Idle-shutdown timer: fires after lingerDelay seconds if no new create arrives 
    closing         bool               // Latched true on shutdown decision; prevents further creates 
    retired         map[string]struct{}  // IDs of clients that explicitly announced exit (never pruned)
    
    mu              sync.Mutex         // Guards mutating above fields atomically: pending, pathIndex, shutdownTimer  
}
```

## 5.2 Workspace — 工作空间（单目录 = 单 workspace）

每个 `Workspace` 就是 Crush 管理的一个具体**代码工作区**（对应一个文件系统路径）。它是 App + metadata 的复合体：

```go
type Workspace struct {
    *app.App            // Inherits all app services: sessions, messages, agents, permissions, etc.  
    ID     string      // Unique namespace for this workspace
    Path   string      // The resolved absolute path of the working directory  
    
    Cfg        *config.ConfigStore // This workspace's config snapshot (isolated to workspace level — different workspaces can have different configs!)
    Skills     *skills.Manager         // Skill discovery results scoped to this workspace
    
    ctx        context.Context           // Workspace-scoped run context; all goroutines derived from it for lifecycle management via cancel() on teardown
    cancel     context.CancelFunc   
    runMu      sync.Mutex                // Guard closing/tear down and prevent new agent runs during shutdown  
    closing    bool                      // Flag to gate dispatch of agent runs after teardown starts 
    runWG      sync.WaitGroup            // Track dispatched agent goroutines: Shutdown waits for them all
    
    clientsMu  sync.RWMutex              // Guards the below `clients` map atomically
    clients     map[string]*clientState // Connected clients (by HTTP client ID) with their SSE event stream connections 
    activeSessions map[string]string   // Currently viewing sessions per active client. Cleared automatically when entries are removed.  
}  
```

## 5.3 Workspace 的生命周期 — 完整状态机

### Phase 1: Create（创建阶段）

当一个 HTTP client 调用 `POST /v1/workspaces`时：

```go
func (s *Backend) CreateWorkspace(clientID string, path string, opts ...WorkspaceCreateOption) (*workspace.Workspace, error) {
    s.mu.Lock()
    
    // Check that the server isn't shutting down (closing == true prevents new creation once shutdown starts).  
    if s.closing { return nil, ErrServerNotIdle }  // Can accept creates anymore during shutdown period
    
    // Deduplication check: if a workspace already exists for this path
    if wID, ok := s.pathIndex[path]; ok {   // resolvedPath (absolute with symlinks followed + cleaned)  
        err = fmt.Errorf("workspace at the same path already created") // Can't open two workspaces at exactly the exact location 
   
    // If we're about start a slow config / database init:
    s.pending++  // Incrementing so teardown doesn't trigger shutdown prematurely if it fires while initialization in flight
    
    s.mu.Unlock()  // Release lock — init operations take time (can take many seconds) 
   
    ┌── Slow Init Path (outside mu lock to avoid blocking other creates):
    │   configStore = loadConfig(path, cfgOverrides...)  // shellconfig loading + deep merge
    │   skillsMgr = discoverSkills(cfg)                   // workspace-level skill discovery  
    │   dbConn, dbSvc     = openDBConnection(dataDir)      // Open / reconnect database per-workspace 
    │   appInstance   = app.New(ctx, dbConn, configStore, skillsMgr)  // Wires up session/message/coordinator/lsp/mcp/skills everything
    │   
    │   if err != nil: s.mu.Lock(); s.pending--; s.mu.Unlock(); return error ...  // Cleanup on init failure  
    │    
    └── Back to critical section for final registration 
   
    s.mu.Lock()
    defer s.pending--     // Decrement pending counter (used by idle-shutdown logic)
                          
    w = workspace.New(...)   // Construct Workspace struct: path, resolvedPath 
                                  
    s.workspaces.Put(w.ID(), w)  // Put it in global map keyed by ID  
    s.pathIndex[w.resolvedPath] = w.ID()  // Index the path for deduplication  
    
    shutdownFn = func() { /* Triggers Shutdown when last workspace removed or server is done hosting live sessions */ }       
   
    if err := w.start(); err != nil:
        s.workspaces.Remove(w.ID())
        s.pathIndex[w.resolvedPath] = "" // Clear index on error for cleanup 
    │    
    return w, nil  
```

### Phase 2: Client Attachment（客户端连接阶段）

Client (VS Code / Neovim plugin) attaches via SSE HTTP endpoint to workspaces: `GET /v1/workspaces/{id}/events`.

```go
func (w *Workspace) AttachSSE(clientID string, stream http.ResponseWriter) error {   // Register client connection with workspace  
    w.clientsMu.Lock() 
    defer clientsMu.Unlock()
   
    state := &clientState {streams: +1}     // streams counts number of SSE event streams the client has open — one at a time normally but can have more simultaneously during rapid reconnection. 

    // Check that the client isn't expired or disabled before attaching to this workspace 
    if s.retired[clientID]: return ErrClientRetired ...  // Client already notified exit; no new creation / attachment allowed anymore 
    
    w.clients[clientID] = state   // Add or update existing entry  
}  

// The hold timer mechanism prevents a workspace from being torn down while a newly created client is still attaching:
func (c *clientState) startHoldTimer(grace time.Duration, release func()) {
     c.holdTimer = time.AfterFunc(grace, func()  // grace is createGrace (30s), detachGrace (10s); if the timer fires before stream count reaches > 0, workspace teardown begins. 
         w.teardown(c)  
      )
}
```

### Phase 3: Active Operation（运行阶段）

In this phase a user's agent calls through HTTP are routed to `backend.Workspace.AgentRequest()` → internally invokes the app instance methods that dispatch agent runs or update sessions or handle permissions requests.

This includes: 
- `POST /v1/workspaces/{id}/agent` — Start new agent run
- `GET /v1/workspaces/{id}/sessions/{sid}/messages` — Get messages for display in client TUI UI.
- `OPTIONS ...askFor...` — Ask permission on behalf of workspace to allow certain tool calls

During this phase the workspaces' internal AppState is kept alive: no teardown during any active session or connection by any attached clients until grace periods expire (see next section). 

### Phase 4: Grace Periods and Teardown（宽限期与终止阶段）

#### Grace Periods Overview

Crush uses three staggered grace timers to manage graceful shutdown. This is what makes Crush server behave like a "lively" daemon rather than simply instantly killing processes when the last client detaches. 

| Timer name | Duration | Trigger |
|-------------|---------|----------|  
| `DefaultCreateGrace` = 30s | Created workspace but not yet attached an SSE stream | Prevent torn-down during rapid connection attempts from same user or multiple VS Code windows opening at same time 
| `DefaultDetachGrace` = 10s | Client's last SSE stream dropped without explicit release (e.g. plugin crashed / laptop closed lid unexpectedly) | Allows reconnection window before destroying the workspace entirely  
| `DefaultIdleShutdownDelay` = 60s  | Last workspace released or client disconnected for all workspaces | Graceful period for server to shut itself down once nothing else needs it  

```go  
func (s *Backend) maybeSchedShutdown(): {
    if s.closing || !s.workspaces.Empty() || s.pending > 0: return // Don't run during creating or when still active workspaces pending
}
// Schedules an idle shutdown after lingerDelay seconds; but will be cancelled by any new CreateWorkspace call in that window. 
s.shutdownTimer = time.AfterFunc(s.lingerDelay, func ( ) {
    s.mu.Lock()  
       if !s.closing:   // still nothing left to do so we actually go ahead and shut down  
           ...trigger full stop sequence  
           for w := range all workspaces: w.Shutdown(ctx)  // tear each one down 
           s.h.close(); ...// Shut HTTP listener too 
```

#### Teardown Sequence（Workspace-level）

When a client detaches or workspace times out (holdTimer/detachGrace fires):

```go
func (w *workspace) teardown(...) {
    if !w.clientsMu.TryLock() // Already locked somewhere else; skip teardown attempt for now. Only retry later once we get the lock again next cycle 
        return  
    defer clientsMu.Unlock () 
   
    // Remove this specific client ID from our list of attached ones first. No more events will be pushed to that one anymore after removing it! 
    state, ok := w.clients[clientID]; if !ok: return // Wasn't registered at all - nothing to do  
    delete(w.clients, clientID)  // Remove before doing actual cleanup work 
    
    w.activeSessions[clientID] = ""   // Reset active session ID for that client (no longer viewing any sessions on UI anymore too — effectively "detached")
      
    if w.streamsRemaining() == 0 && !state.released: // If no remaining streams AND not explicitly released earlier by the one disconnecting 
      startHoldTimer(gracePeriod)    /* Detach grace period: timer set to destroy workspace after detachGrace seconds unless reattached in time */  
      // During this window (detachGrace = ten seconds):
      //   - Reconnection from same user is still possible without destroying workspace / losing session messages etc
      //   - On timeout → calls `w.teardown()` again recursively! 
      return  // Don't destroy yet. Let the timer decide what happens next  
      
    if w.streamsRemaining() == 0 && state.released: /* If explicitly released before disconnect happened, we DO go ahead and destroy now immediately because it means there's definitely no intention of reconnecting */
       s.teardownWorkspace(w)  // Proceed straight to full teardown  
}  

func (w *workspace) teardown() {
    w.runMu.Lock()          // Prevent new agent dispatch during shutdown 
      defer runMu.Unlock() 
   
    if w.closing: return; /* Prevent re-entrant calls */  
    w.closing = true;      // Mark this workspace as "already closing so no one else attempts tear down again on same object!"  
      
    w.cancel()   // Cancel all goroutine contexts derived from its context ctx 
  
    /* WaitGroup mechanism ensures that all running agent turns (which are started before shutdown begins) complete gracefully rather than being killed mid-flight */
    if runWG != 0: 
         go func(){
             runWG.Wait()       // Block until every active turn terminates on its own terms instead of abruptly killing everything at once
             w.cleanup( )       /* Only after all turns are done do we continue cleanup work like closing DB / HTTP listeners etc */
         }()  
      
    if shutdownFn != nil:  /* Notify backend that it should also trigger server graceful shut down sequence too (see s.schedShutdown above) */    
}

```

#### Server Shutdown Sequence

When final workspace released and idleTimer fires, the whole process starts tearing itself down progressively across all its component subsystems: 

| Step | Action |
|------|--------|
| 1 | `h.Shutdown(ctx)` → graceful HTTP server shutdown. Stops accepting new connections and waits for in-flight HTTP requests to finish naturally. |
| 2 | For each active workspace W: call W.Shutdown( ) to tear down all resources (agents stop, DB releases). 
| 3 | db.Close() on every connection pool released back 
| 4. MCP client shutdown 
  

```go
// Server's main listener loop stops accepting new connections
func (s*Server) Shutdown(ctx context.Context){
    s.h.Shutdown(ctx) /* Stop listening and wait out current requests finishing gracefully */ 
    
    /* For each workspace that was created */  
      for w := range all workspaces:
         w.teardown()   /* See detailed teardown steps above — waits out inflight agent calls too! */ 
         
     // Release database connections so they can be returned or garbage collected by Go runtime  
     db.Release(cfg.Options.DataDirectory) 
  
    // Close any remaining MCP server connections and unblock goroutines waiting on init completion signals. This allows `mcp.WaitForInit()` inside client/server run paths to return instead of hanging forever if something went wrong before initializing successfully during startup
}

```

## 5.4 Workspace Path Deduplication Logic

The key mechanism preventing multiple workspaces for same directory: 

1. `resolvedPath = filepath.EvalSymlinks(filepath.Abs(Path))` → absolute path with symlinks resolved + cleaned up slashes etc. If evaluation failed then we use just `path.Clean()` as a fallback — never leaving unresolved links dangling inside our internal maps. 
2. In `CreateWorkspace()`: check if s.pathIndex[resolvedPath] already exists! If true, refuse (return error "workspace at same path already created"). **Note: This is strictly exact string-match** of resolved paths; two different real paths that happen to both point same location will conflict with each other due always resolving first.

This means you must open files from a subfolder inside your project if you want separate workspace sessions for each directory — they need actually-be-different absolute directories! 

## 5.5 Client Lifecycle in Server Mode

Client attachment and disconnection is managed through two mechanisms: **hold timers** (for preventing early teardown) and **retired lists** (permanently banning further creates):

```go
type clientState struct {
    streams          int                    // Count of currently open SSE event streams 
   holdTimer      *time.Timer                  // Active timer object if one has been set by server during grace periods above
    currentSessionID string               // Which session this particular attached session is currently viewing inside our application UI (empty = landing screen / no selection)
    released         bool                      // Whether client announced exit cleanly before disconnecting — used by release mechanism for skipping detach grace period since user explicitly said "I'm done now"  
}

// When a client announces they're leaving gracefully:    
func (s*Backend).RetireClient(clientID string):{   
   s.remired[clientID] = struct{}{}            // Always adding entries permanently to this set without ever removing later memory leaks but it's acceptable because only one UUID per connected process lifetime; cost tiny compared benefits of avoiding orphan workspace issues which would otherwise happen!
}

// When a new connection arrives after someone has retired and no more workspaces created anymore — we reject that instead (which means if user wants to reconnect they need restart server first). This prevents any possibility creating another workspace while still holding onto the same IDs etcetera which could cause unexpected behavior down stream lines like sending old sessions data incorrectly again!

func (s *Backend.CreateWorkspace(...) { 
   s.mu.Lock() defer mu.Unlock() 
   
      if _, isRetired:=s.retired[clientID];isRetired { return nil,ErrClientRetired} // Can create new workspace anymore because it was already marked earlier as retired before! 
       
     ...normal logic proceeds only once we verify this isn't someone who has been disqualified permanently already via retiree list check above!  
```

## 5.6 Backend Error Handling Conventions

Backend uses sentinel errors defined at the top level of `backend.go` for specific scenarios that clients should handle explicitly:

| Error | When Returned | Client Action |
|-------|--------------|---------------|
| `ErrWorkspaceNotFound` | Workspace ID doesn't exist in our map | Retry after creating one! |
| `ErrClientRetired` | Client ID already announced exit (or crashed) | Can't create anymore until restart server |  
| `ErrServerNotIdle`   | Server is shutting down right now  | Wait for re-attach once ready again before trying more requests! 
| `ErrPathRequired`    | Workspace path wasn't provided  | Re-send request with correct one set instead of empty string which causes nothing to happen silently otherwise too later on downstream calls within same function call context!
| `ErrInvalidPermissionAction` | Invalid action in permissions update/permission query (allowed values are allow deny grant etc)  — verify your syntax matches allowed constants first before retrying another request again please! 

## 5.7 Workspace vs Session Hierarchy

``` 
Workspace A (path=/Users/john/project-A, id=aaa, cfg=A_config.yaml)
    ├── Session A1 (title="Build API endpoint", messages = [...])
    ├── Session A2 (title ="Add unit tests", messages=[...])  
        └── Child session Z-0  (parent_id=A2, created by agent itself during run! It's nested underneath parent_session_A_2 above but with its own set of separate messages completely distinct from either A1 or the parent one!).

Workspace B (path=/Users/john/project-B id=bbb, cfg=B_config.yaml — could be completely different config file than workspace A even though both live under /Users/john/!)
    ├── Session B1 (title="Debug bug #42", messages = [...])  
```

Key point: **workspaces are fully isolated** from one another (different databases/configs/skills/agents/LSP instances), while sessions belong strictly inside one workspace only. No cross-work session sharing exists at all in current design! 
