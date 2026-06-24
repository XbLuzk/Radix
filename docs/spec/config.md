# Config — Configuration Loading

## Struct

```go
package config

type Config struct {
    APIKey     string   // ANTHROPIC_API_KEY
    Model      string   // default: "claude-sonnet-4-6"
    MaxTokens  int      // default: 4096
    TokenLimit int      // default: 100000
}
```

## Load Order

1. 环境变量（`os.Getenv`）
2. 当前目录 `.env` 文件（`godotenv`）

环境变量优先级高于 `.env`。

## Usage

```go
// cmd/radix/main.go
cfg, err := config.Load()
if err != nil {
    log.Fatal(err)
}

provider := llm.NewAnthropic(cfg.APIKey, cfg.Model)
memory := memory.NewManager(systemPrompt, cfg.TokenLimit)
// ...
```

## .env Example

```
ANTHROPIC_API_KEY=sk-ant-xxx
# optional overrides:
# RADIX_MODEL=claude-sonnet-4-6
# RADIX_MAX_TOKENS=4096
# RADIX_TOKEN_LIMIT=100000
```
