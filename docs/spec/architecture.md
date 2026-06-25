# Architecture Overview

## Module Map

```mermaid
graph TB
    A["<b>cmd/radix</b><br/>Parse flags, load config, wire dependencies, start Agent"]
    B["<b>internal/agent</b><br/>ReAct loop, stream consumer, concurrent tool execution"]
    
    subgraph CoreComponents [Core Components]
        direction LR
        C["<b>llm</b><br/>Provider<br/>Types"]
        D["<b>tool</b><br/>Registry<br/>+ impl"]
        E["<b>memory</b><br/>Manager"]
    end
    
    F["<b>internal/config</b><br/>.env / env var loading"]

    A --- B
    B --- CoreComponents
    CoreComponents --- F
    
    style A fill:#f6f8fa,stroke:#d1d5da
    style B fill:#f6f8fa,stroke:#d1d5da
    style F fill:#f6f8fa,stroke:#d1d5da
    style C fill:#ffffff,stroke:#d1d5da
    style D fill:#ffffff,stroke:#d1d5da
    style E fill:#ffffff,stroke:#d1d5da
```

## Data Flow (One ReAct Turn)

```mermaid
graph TD
    U[User Input] --> A[agent.Run]
    A --> step1[memory.AddMessage: RoleUser, input]
    step1 --> step2[memory.BuildRequest: tools]
    step2 -->|returns ChatRequest| step3[llm.ChatStream: ctx, req]
    step3 -->|HTTP POST → SSE| step4[agent.consumeStream: chunks, errs]
    
    step4 -->|Text Delta| stdout[io.Writer]
    step4 -->|ToolCall Delta| acc[Accumulate as ToolCall array]
    
    acc --> step5{Check FinishReason}
    step5 -->|"stop"| done[return nil]
    step5 -->|"tool_calls"| step6[agent.executeConcurrently: toolCalls]
    
    step6 --> exec[tool.Execute: ctx, inputJSON]
    exec --> step7[memory.AddMessage: RoleTool, result]
    step7 -->|loop back to step 2| step2
```

## Dependency Graph

```mermaid
graph LR
    agent --> llm["llm.Provider (interface)"]
    agent --> tool["tool.Registry (concrete)"]
    agent --> memory["memory.Manager (concrete)"]
    agent --> io["io.Writer (Phase 3: Bubble Tea)"]
    
    cmd --> config["config.Load()"]
    cmd --> agentNew["agent.New(llm, tools, memory, os.Stdout)"]
```

Agent depends on three replaceable components (LLM Provider, Tools, Output target), the rest use concrete structs.

## Phase Mapping

| Phase | Modules Changed | Impact Scope |
|-------|-----------|---------|
| Phase 1 | All new | - |
| Phase 2 | `memory/` implements compression logic; `agent/` adds turn counter | local |
| Phase 3 | Add `tui/`; `cmd/` changes output implementation | agent code unchanged |
| Phase 4 | Add `mcp/`; `tool/` adds MCP adapter | injected via Tool interface |
