# 八、数据持久化与 Session 系统

## 8.1 SQLite + sqlc 代码生成体系

Crush 使用 SQLite 作为唯一的数据存储后端，通过 **sqlc** 从原始 SQL 自动生成 Go 代码：

```
data directory (默认 ~/.crash/)
├── state.db           — 主数据库文件：
│   ├── sessions       — 用户对话元数据（ID、标题、创建时间）
│   ├── messages       — session 内的所有消息内容及时间戳
│   └── files          — FileTracker 追踪的文件 metadata 
├── migrations/        — schema 版本迁移文件  
internal/db/sql/*.sql  → sqlc generate → internal/db/*.go (models + CRUD)
```

### 目录结构

```
internal/db/
├── sql/                 Raw SQL query files
│   ├── sessions.sql     → sessions.sql.go (Session CRUD)
│   ├── messages.sql    → messages.sql.go (Message CRUD)  
│   ├── files.sql       → files.sql.go (File CRUD)
│   └── stats.sql       → stats.sql.go (cost/usage statistics)
├── models.go            All model structs: Session, Message, File, Stats  
├── querier.go           sqlc Querier interface shared across packages  
├── connect.go / _ncruces.go  Database connection pool management with per-directory file locking 
├── datadirlock.go       Implements fcntl flock on data directory lockfile
```

### 连接池管理

多个进程/实例不能同时读写同一个 data directory 的 SQLite 数据库。通过 **fcntl flock** 机制防止损坏：

## 8.2 Session Service (`internal/session/service.go`)

| Method | Purpose |
|--------|---------|
| `Create(ctx, title)` | 创建新对话，返回 UUID  
| `Get(ctx, id)`       | 通过 ID 加载单个 session 
| `UpdateTitle(ctx, id, name)` | 修改会话标题（通常在 LLM 首次回复后自动设置）  
| `Delete(ctx, id)`    — 从数据库彻底移除该会话及其所有关联记录

### Session Model

```go
type Session struct {
    ID              uuid.UUID   // Unique identifier 
    ParentSessionID *uuid.UUID   // Nullable parent reference for sub-s spawned by agent itself during runtime automatically! 
    Title           string      
    CreatedAt       time.Time     
}
```

## 8.3 Message Service (`internal/message/service.go`)

Message stores individual entries within each session:

| Field | Type | Purpose |
|-------|------|---------|
| Role | message.Role | Assistant / User / Tool distinction |
| Parts | []Part | Multi-part messages include text + attachments/images/tool_results packed together |

## 8.4 FileTracker Service (`internal/filetracker/service.go`)

Tracks files modified during agent runs for audit/diff purposes:

```go
type Service struct { 
    files map[string]*FileRecord   // filepath → metadata
}

func (s *Service) Touch(ctx context.Context, filePath string, mode os.FileMode) error 
    → Records each file accessed by any tool operation (view/edit/write/download) during an agent turn. Useful for generating summary of changes made at end of run.
```

## 8.5 Stats Service (`internal/db/stats.sql.go`)

Tracks cost/usage statistics per session:

| Field | Type | Purpose |
|-------|------|---------|
| SessionID | string | Which session this usage belongs to  
| Model | string | Which LLM model was used    
| InputTokens | int64 | Tokens consumed in prompts      
| OutputTokens | int64 | Tokens produced by the model     
| TotalCost | float64 | Dollar/cost equivalent of usage
