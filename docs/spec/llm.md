# LLM — Types & Provider

## Core Types

```go
package llm

type Role string

const (
    RoleSystem    Role = "system"
    RoleUser      Role = "user"
    RoleAssistant Role = "assistant"
    RoleTool      Role = "tool"
)

type Message struct {
    Role       Role
    Content    string
    ToolCalls  []ToolCall
    ToolCallID string      // only for RoleTool
}

type ToolCall struct {
    ID    string
    Name  string
    Input string           // JSON string
}

type ToolDef struct {
    Name        string         `json:"name"`
    Description string         `json:"description"`
    InputSchema map[string]any `json:"input_schema"`
}
```

## ChatRequest & ChatChunk

```go
type ChatRequest struct {
    Messages     []Message
    Tools        []ToolDef
    SystemPrompt string
}

type ChatChunk struct {
    Type              ChatChunkType
    TextDelta         string
    ToolCallID        string
    ToolCallName      string
    ToolCallInputDelta string
    FinishReason      string   // "stop" | "tool_calls" | "length"
}

type ChatChunkType int

const (
    ChunkText          ChatChunkType = iota
    ChunkToolCallStart
    ChunkToolCallDelta
    ChunkDone
)
```

## Provider Interface

```go
type Provider interface {
    ChatStream(ctx context.Context, req *ChatRequest) (<-chan ChatChunk, <-chan error)
}
```

两个 channel：chunks 和 errors。Provider 在 goroutine 中写入，读完或出错后 close 两者。

调用方用 `select` 同时监听：

```go
chunks, errs := provider.ChatStream(ctx, req)
for {
    select {
    case chunk, ok := <-chunks:
        if !ok { return }
        // process chunk
    case err, ok := <-errs:
        if !ok { return }
        return err
    }
}
```

## Anthropic Adapter

文件：`internal/llm/anthropic.go`

### Request Mapping

| Internal | Anthropic API |
|----------|---------------|
| `RoleSystem` | `system` (top-level param) |
| `RoleUser` | `user` role in messages array |
| `RoleAssistant` | `assistant` role in messages array |
| `RoleTool` | `user` role with `tool_result` content block |

Tool definitions → `tools[]` top-level param.

### SSE Event → ChatChunk Mapping

| Anthropic SSE event | ChatChunk output |
|---------------------|------------------|
| `content_block_start` (type `text`) | Begin streaming text |
| `content_block_start` (type `tool_use`) | `ChunkToolCallStart` |
| `content_block_delta` (`text_delta`) | `ChunkText` |
| `content_block_delta` (`input_json_delta`) | `ChunkToolCallDelta` |
| `message_delta` (`stop_reason: "end_turn"`) | `ChunkDone{FinishReason: "stop"}` |
| `message_delta` (`stop_reason: "tool_use"`) | `ChunkDone{FinishReason: "tool_calls"}` |

### Error Handling

- HTTP 4xx/5xx → write to errs channel, close both channels
- SSE stream interrupted → write error to errs, close both
- Auth failure → `Config` validation should catch this early, but Provider still reports it

### Configuration

```go
type AnthropicProvider struct {
    apiKey  string
    model   string
    baseURL string   // default: https://api.anthropic.com/v1
    client  *http.Client
}
```
