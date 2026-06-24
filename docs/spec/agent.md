# Agent — ReAct Loop

## Struct & Constructor

```go
package agent

type Agent struct {
    llm       llm.Provider
    tools     *tool.Registry
    memory    *memory.Manager
    output    io.Writer
    maxTurns  int
}

func New(llm llm.Provider, tools *tool.Registry, memory *memory.Manager, output io.Writer) *Agent {
    return &Agent{
        llm:      llm,
        tools:    tools,
        memory:   memory,
        output:   output,
        maxTurns: 25,
    }
}
```

## Main Loop

```go
func (a *Agent) Run(ctx context.Context, userInput string) error {
    a.memory.AddMessage(llm.Message{Role: llm.RoleUser, Content: userInput})

    for turn := 0; turn < a.maxTurns; turn++ {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }

        req := a.memory.BuildRequest(a.tools.List())
        chunks, errs := a.llm.ChatStream(ctx, req)

        assistantMsg, isToolCall, err := a.consumeStream(chunks, errs)
        if err != nil {
            return fmt.Errorf("llm: %w", err)
        }
        a.memory.AddMessage(assistantMsg)

        if !isToolCall {
            return nil
        }

        results := a.executeConcurrently(ctx, assistantMsg.ToolCalls)
        for _, tr := range results {
            a.memory.AddMessage(tr)
        }
    }
    return ErrMaxTurns
}
```

## Stream Consumer

```go
func (a *Agent) consumeStream(chunks <-chan llm.ChatChunk, errs <-chan error) (llm.Message, bool, error) {
    var (
        textBuf    strings.Builder
        toolCalls  []llm.ToolCall
        currentTC  *llm.ToolCall
    )

    for {
        select {
        case chunk, ok := <-chunks:
            if !ok {
                // channel closed, stream done
                msg := llm.Message{Role: llm.RoleAssistant, Content: textBuf.String(), ToolCalls: toolCalls}
                isToolCall := len(toolCalls) > 0
                return msg, isToolCall, nil
            }
            switch chunk.Type {
            case llm.ChunkText:
                textBuf.WriteString(chunk.TextDelta)
                fmt.Fprint(a.output, chunk.TextDelta)

            case llm.ChunkToolCallStart:
                if currentTC != nil {
                    toolCalls = append(toolCalls, *currentTC)
                }
                currentTC = &llm.ToolCall{
                    ID:   chunk.ToolCallID,
                    Name: chunk.ToolCallName,
                }

            case llm.ChunkToolCallDelta:
                if currentTC != nil {
                    currentTC.Input += chunk.ToolCallInputDelta
                }

            case llm.ChunkDone:
                if currentTC != nil {
                    toolCalls = append(toolCalls, *currentTC)
                }
                msg := llm.Message{Role: llm.RoleAssistant, Content: textBuf.String(), ToolCalls: toolCalls}
                isToolCall := chunk.FinishReason == "tool_calls"
                return msg, isToolCall, nil
            }

        case err, ok := <-errs:
            if ok && err != nil {
                return llm.Message{}, false, err
            }
        }
    }
}
```

## Concurrent Tool Execution

```go
func (a *Agent) executeConcurrently(ctx context.Context, calls []llm.ToolCall) []llm.Message {
    results := make([]llm.Message, len(calls))
    var wg sync.WaitGroup

    for i, call := range calls {
        wg.Add(1)
        go func(idx int, tc llm.ToolCall) {
            defer wg.Done()

            t, ok := a.tools.Get(tc.Name)
            if !ok {
                results[idx] = llm.Message{
                    Role: llm.RoleTool, ToolCallID: tc.ID,
                    Content: fmt.Sprintf("tool not found: %s", tc.Name),
                }
                return
            }

            // Per-tool 30s timeout, independent from sibling tools
            toolCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
            defer cancel()

            result, err := t.Execute(toolCtx, tc.Input)
            if err != nil {
                results[idx] = llm.Message{
                    Role: llm.RoleTool, ToolCallID: tc.ID,
                    Content: fmt.Sprintf("error: %v", err),
                }
                return
            }
            results[idx] = llm.Message{
                Role: llm.RoleTool, ToolCallID: tc.ID, Content: result,
            }
        }(i, call)
    }
    wg.Wait()
    return results
}
```

## Why Per-Tool Timeout ≠ Shared Context

共享 ctx：一个工具超时 → ctx 取消 → 其他 goroutine 被 kill → 结果不完整。

Per-tool `context.WithTimeout(context.Background(), 30s)`：每个工具独立计时，失败一个不影响其他。执行结果全部回灌 LLM，LLM 看到某个工具报 timeout 会自行处理。

## Error Handling

| 错误来源 | 处理方式 |
|----------|---------|
| `ChatStream` 中途出错 | `consumeStream` 返回 error，`Run` 终止 |
| `Tool.Execute` 返回 error | 格式化为 `"error: <msg>"` 装进 `RoleTool` 消息，LLM 自行判断 |
| 工具未找到 | `"tool not found: <name>"`，同上 |
| 超过 25 轮 | 返回 `ErrMaxTurns`，CLI 打印提示 |
| `ctx.Done()` | 用户 Ctrl+C，立即退出 |
