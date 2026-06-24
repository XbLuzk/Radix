# Memory — Message History & Context

## Manager

```go
package memory

type Manager struct {
    systemPrompt string
    messages     []llm.Message
    tokenLimit   int
}

func NewManager(systemPrompt string, tokenLimit int) *Manager

func (m *Manager) AddMessage(msg llm.Message)
func (m *Manager) Messages() []llm.Message
func (m *Manager) TokenCount() int
func (m *Manager) TokenLimit() int
func (m *Manager) SetSystemPrompt(p string)
func (m *Manager) BuildRequest(tools []llm.ToolDef) *llm.ChatRequest
```

## System Prompt

Phase 1 使用硬编码的 system prompt：

```
You are Radix, an AI coding assistant powered by Claude.
You have access to tools for reading/writing files and running shell commands.
Use tools when you need to interact with the file system.
Be concise. When writing code, match the style of the surrounding codebase.
```

`BuildRequest` 将 system prompt 放入 `ChatRequest.SystemPrompt` 字段。

## Token Counting

Phase 1 用近似算法：
- 英文：4 chars ≈ 1 token
- 中文/Unicode：1 char ≈ 1 token
- 具体实现：`len([]rune(content))`

Phase 2 替换为 tiktoken-go 精确计数。

## Compaction（Phase 2）

Phase 1 不实现压缩。`BuildRequest` 直接返回全部消息。

Phase 2 增加的逻辑：
1. `BuildRequest` 中检查 `TokenCount() > TokenLimit()`
2. 超出时，取前 N 条消息调用 LLM 做摘要（一次额外的非流式 API 调用）
3. 摘要结果替换原消息为：`RoleUser, Content: "Here is a summary of the earlier conversation:\n<summary>..."`
4. 保留最近 3 轮对话不做压缩

## Message 生命周期

```
User input  → AddMessage(RoleUser, "fix the bug")
Agent 输出   → AddMessage(RoleAssistant, "I'll read the file first", toolCalls=[...])
Tool 执行    → AddMessage(RoleTool, "<file content>", ToolCallID="tool_xxx")
Agent 继续   → AddMessage(RoleAssistant, "The issue is on line 42...")
... loop ...
User 再输入  → AddMessage(RoleUser, "also add a test")
```
