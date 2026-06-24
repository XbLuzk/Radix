# Radix — Go AI CLI Agent

## 1. 这是什么

Radix 是一个用 Go 从零手写的 AI CLI Agent。它不是要替代 Claude Code，而是一个**学习型项目**——通过亲手实现 ReAct loop、上下文管理、工具调用链路，深入理解 AI agent 的工程内核。

目标用户：我自己（以及面试官）。

## 2. 为什么做

- 现有 agent（Claude Code、Codex、Cursor）都是黑盒。吃透它们的最快方式是自己造一个
- paicli（Java）证明了 agent 架构可以很清晰，但 Java 太重。用 Go 重做一遍，更轻、更快、更好分发
- 外企 AI 应用岗面试，能聊清楚"我从零实现过 agent loop"比调过 LangChain API 有说服力得多

## 3. 不做的事

- ❌ 不搞容器化沙盒——太重，实用价值存疑
- ❌ 不搞状态树/时空回溯——概念好但 LLM 非确定性导致无法真正实现
- ❌ 不以"创新点"自居——这不是范式突破，是工程练习

## 4. 设计原则

1. **Go 原生并发**：并行 tool call 用 goroutine + channel，不引入消息队列
2. **接口隔离**：LLM Provider、Tool、Memory 全部接口化，可替换
3. **单二进制分发**：`go build` 一个文件，不依赖运行时
4. **先跑通再优化**：Phase 1 只做最简 ReAct loop，不提前设计

---

## 5. 架构

```
┌─────────────────────────────────┐
│            TUI (Bubble Tea)                                                     │
├─────────────────────────────────┤
│          Agent (ReAct Loop)                                                  │
│   Think → Act → Observe → Loop                                   │
├──────────┬──────────┬───────────┤
│  LLM                  │  Tool                   │  Memory               │
│ Provider              │ Registry             │  Manager               │
├──────────┴──────────┴───────────┤
│  Anthropic / OpenAI / DeepSeek                                        │
└─────────────────────────────────┘
```

核心循环：

```
1. 用户输入 → 拼装 messages（system prompt + 历史 + 新消息）
2. 调 LLM → 解析响应
3. 如果是 tool_call → 执行工具 → 结果回灌 messages → 回到步骤 2
4. 如果是 text → 流式输出给用户 → 等待下一轮输入
```

## 6. 技术选型

| 层 | 选择 | 理由 |
|----|------|------|
| 语言 | Go | 编译为单二进制，goroutine 天然适合并行 tool call |
| TUI | [Bubble Tea](https://github.com/charmbracelet/bubbletea) | Go 生态最成熟的 TUI 框架 |
| LLM SDK | 自写 HTTP client | Anthropic/OpenAI 的 API 足够简单，不需要 SDK |
| MCP | 自写 MCP client | 协议本身简单，stdio/SSE 两种 transport |

## 7. 分期计划

### Phase 1: 最小 ReAct（2 周）

**验收标准**：在终端里输入一句话，agent 能自主调用工具完成任务。

- [ ] Go REPL 框架，读用户输入
- [ ] Anthropic API client（messages endpoint, streaming, tool use）
- [ ] Tool 接口 + 3 个内置工具（`read_file` / `write_file` / `execute_command`）
- [ ] ReAct loop：接收 tool_call → 执行 → 回灌结果 → 继续
- [ ] 流式输出 text response 到终端
- [ ] 博客 #1：Why Go for an AI agent?

### Phase 2: 上下文工程（2 周）

**验收标准**：长对话不爆 context window，自动压缩历史。

- [ ] Message 历史管理（system / user / assistant / tool_result）
- [ ] Token 计数（tiktoken-go 或近似算法）
- [ ] 自动压缩：接近 budget 时对早期消息做摘要
- [ ] `/compact` 手动触发压缩
- [ ] 博客 #2：How I manage context window in an AI agent

### Phase 3: TUI 体验（2 周）

**验收标准**：比裸 REPL 好看，有流式渲染和 diff 高亮。

- [ ] Bubble Tea 集成，替换裸 REPL
- [ ] 流式 thinking 展示（折叠 reasoning）
- [ ] Diff 高亮渲染
- [ ] 博客 #3：Building a TUI for AI agent interaction

### Phase 4: 生态接入（选做，2 周）

- [ ] MCP client（stdio transport）
- [ ] 通过 JSON 配置加载第三方 MCP server
- [ ] 博客 #4：MCP from scratch — how the protocol actually works

---

## 8. 与 paicli 的关系

paicli 是 Java 实现的成熟 CLI agent（23 期迭代），Radix 在架构上参考了它的模块划分（Agent / ToolRegistry / MemoryManager / LLM Provider），但：

- Radix 用 Go 重写，不涉及任何 paicli 代码拷贝
- 接口设计和并发模型因语言不同而有本质差异
- README 中注明 "Architecture inspired by paicli"

---

## 9. 博客计划

每完成一个 Phase，一篇 1500-2000 字的英文技术博客，发 GitHub Pages 个人博客 + 同步到 dev.to：

1. **Why I built an AI agent in Go** — 语言选择、并发模型、单二进制
2. **Context window management from scratch** — token 计数、压缩策略、取舍
3. **TUI design for AI agents** — 流式渲染、diff 展示、用户体验
4. **MCP protocol internals** — stdio transport、工具发现、错误处理

博客比代码更重要——面试官大概率不会 clone 你的 repo 跑，但会搜你的名字看博客。
