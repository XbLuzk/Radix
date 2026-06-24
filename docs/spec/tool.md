# Tool — Interface & Registry

## Interface

```go
package tool

type Tool interface {
    Name() string
    Def() llm.ToolDef
    Execute(ctx context.Context, inputJSON string) (string, error)
    IsDestructive() bool
}
```

- `Def()` 返回工具的 JSON Schema 定义，直接透传给 LLM
- `Execute(inputJSON)` 接收 LLM 传的 JSON string，工具自己 Unmarshal
- `IsDestructive()` Phase 1 预留给后续 Phase 的用户确认机制

## Registry

```go
type Registry struct {
    tools map[string]Tool
}

func NewRegistry() *Registry
func (r *Registry) Register(t Tool)
func (r *Registry) Get(name string) (Tool, bool)
func (r *Registry) List() []llm.ToolDef
```

Concrete struct，不需要 interface。只有一种实现。

## Phase 1 Tools

| Tool | File | IsDestructive | Notes |
|------|------|---------------|-------|
| `read_file` | `read_file.go` | false | 支持 `path` + `offset`/`limit` |
| `write_file` | `write_file.go` | true | `path` + `content`，覆盖写入 |
| `execute_command` | `execute_command.go` | true | `command` + `cwd`，Phase 1 直接执行 |

### read_file

```json
{
  "name": "read_file",
  "description": "Read a file from the local filesystem. You can access any file directly by using this tool.",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": { "type": "string", "description": "Absolute path to the file" },
      "offset": { "type": "integer" },
      "limit": { "type": "integer" }
    },
    "required": ["path"]
  }
}
```

### write_file

```json
{
  "name": "write_file",
  "description": "Write content to a file. Overwrites existing files.",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": { "type": "string" },
      "content": { "type": "string" }
    },
    "required": ["path", "content"]
  }
}
```

### execute_command

```json
{
  "name": "execute_command",
  "description": "Execute a shell command.",
  "input_schema": {
    "type": "object",
    "properties": {
      "command": { "type": "string" },
      "cwd": { "type": "string" }
    },
    "required": ["command"]
  }
}
```

Implementation notes:
- `read_file`: read file bytes → return as string. Handle "file not found" gracefully.
- `write_file`: create parent dirs if needed (`os.MkdirAll`).
- `execute_command`: use `os/exec`, capture stdout+stderr, 30s timeout.
