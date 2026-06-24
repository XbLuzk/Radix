# Architecture Overview

## Module Map

```
┌──────────────────────────────────────┐
│              cmd/radix                │
│   Parse flags, load config, wire     │
│   dependencies, start Agent          │
├──────────────────────────────────────┤
│           internal/agent              │
│   ReAct loop, stream consumer,       │
│   concurrent tool execution          │
├─────────┬──────────┬────────────────┤
│  llm    │  tool    │  memory        │
│ Provider│ Registry │  Manager       │
│ Types   │ + impl   │                │
├─────────┴──────────┴────────────────┤
│           internal/config            │
│   .env / env var loading             │
└──────────────────────────────────────┘
```

## Data Flow（一轮 ReAct）

```
User Input
    │
    ▼
agent.Run()
    │
    ├─► memory.AddMessage(RoleUser, input)
    │
    ├─► memory.BuildRequest(tools)
    │       └─► ChatRequest{Messages, Tools, SystemPrompt}
    │
    ├─► llm.ChatStream(ctx, req)
    │       └─► HTTP POST → Anthropic SSE → ChatChunk channel
    │
    ├─► agent.consumeStream(chunks, errs)
    │       ├─ 文本增量 → io.Writer (Phase 1: stdout)
    │       └─ ToolCall 增量 → 累积为 ToolCall[]
    │
    ├─► 判断 FinishReason:
    │       "stop"       → return nil  (done)
    │       "tool_calls"  → continue
    │
    └─► agent.executeConcurrently(toolCalls)
            ├─ tool.Execute(ctx, inputJSON)
            └─ memory.AddMessage(RoleTool, result)
                    └─► loop back to step 2
```

## Dependency Graph

```
agent → llm.Provider (interface)
agent → tool.Registry (concrete)
agent → memory.Manager (concrete)
agent → io.Writer        (Phase 3: replaced with Bubble Tea channel)

cmd → config.Load()
cmd → agent.New(llm, tools, memory, os.Stdout)
```

Agent 依赖三个可替换的东西（LLM Provider、工具集、输出目标），其余用 concrete struct。

## Phase Mapping

| Phase | 改哪些模块 | 影响范围 |
|-------|-----------|---------|
| Phase 1 | 全部新建 | - |
| Phase 2 | `memory/` 实现压缩逻辑；`agent/` 加 turn counter | local |
| Phase 3 | 新增 `tui/`；`cmd/` 换 output 实现 | agent 代码不改 |
| Phase 4 | 新增 `mcp/`；`tool/` 加 MCP 适配器 | 通过 Tool interface 注入 |
