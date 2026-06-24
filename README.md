# Radix

A Go AI CLI agent built from scratch. Not a Claude Code replacement — a learning project to understand AI agent internals.

## Docs

- [PRD](docs/PRD.md) — what and why
- [Architecture](docs/spec/architecture.md) — module map and data flow
- [LLM](docs/spec/llm.md) — types, Provider interface, Anthropic adapter
- [Tool](docs/spec/tool.md) — Tool interface, Registry, built-in tools
- [Memory](docs/spec/memory.md) — message history, token budget
- [Agent](docs/spec/agent.md) — ReAct loop, stream consumer, concurrent execution
- [Config](docs/spec/config.md) — configuration loading
- [Phase Notes](docs/spec/phase-notes.md) — what each phase changes

## Quick Start

```bash
cp .env.example .env
# edit .env with your ANTHROPIC_API_KEY
go run ./cmd/radix
```

## Architecture

Inspired by paicli's module structure, rewritten in Go with goroutine-based concurrency.
