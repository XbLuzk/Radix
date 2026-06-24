# Phase Implementation Notes

每个 Phase 改哪些模块，不提前设计后续 Phase 的细节。

## Phase 1 — Minimal ReAct

| 模块 | 动作 | 文件 |
|------|------|------|
| `cmd/` | 新建，组装依赖 | `cmd/radix/main.go` |
| `config/` | 新建，加载 `.env` | `internal/config/config.go` |
| `llm/` | 新建 types + Provider interface + Anthropic 实现 | `internal/llm/provider.go`, `anthropic.go` |
| `tool/` | 新建 interface + Registry + 3 个工具 | `internal/tool/tool.go`, `read_file.go`, `write_file.go`, `execute_command.go` |
| `memory/` | 新建 Manager（无压缩） | `internal/memory/memory.go` |
| `agent/` | 新建 ReAct loop + stream consumer + 并发执行 | `internal/agent/agent.go` |

**验收**：终端输入 "Read the README and tell me what this project does"，Agent 自己调 `read_file` 读取 README 并回答。

## Phase 2 — Context Engineering

| 模块 | 动作 |
|------|------|
| `memory/` | 实现精确 token 计数（tiktoken-go）；`BuildRequest` 加入自动压缩逻辑 |
| `agent/` | 加入 turn counter + `/compact` 斜杠命令 |

不碰的文件：`llm/`、`tool/`、`config/`。

## Phase 3 — TUI

| 模块 | 动作 |
|------|------|
| `tui/` | **新建**，Bubble Tea 界面 |
| `cmd/` | 修改启动逻辑，替换 `os.Stdout` 为 TUI channel |

不碰的文件：`agent/`（`io.Writer` 接口不用改）、`llm/`、`tool/`、`memory/`。

## Phase 4 — MCP

| 模块 | 动作 |
|------|------|
| `mcp/` | **新建**，MCP stdio client |
| `tool/` | 新建 `mcp_adapter.go`，MCP tool → `tool.Tool` 接口 |
| `config/` | 新增 `LoadMCPConfig()`，读取 `mcp.json` |

不碰的文件：`agent/`、`llm/`、`memory/`。
