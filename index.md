---
layout: default
---

## Stop paying Claude to understand the same code it already understood yesterday.

llmcache is a local MCP server that caches AI-generated understanding of your source code. Claude reads each file once and every session after that starts from cached understanding, saving you tokens and money.

---

## Why It Matters

Claude Code uses a stateless API. On every turn, the **entire conversation history** — including every file Claude has read — is re-sent as input tokens. A 2,000-token file read on turn 1 costs you 2,000 input tokens on every subsequent turn for the rest of the session.

llmcache replaces that 2,000-token file with a 50-token summary. You save 1,950 tokens **on every turn**, not just once.

On top of that, Claude generates output tokens to process each file — intermediate understanding, analysis, reasoning. With a cached summary, Claude skips that work entirely. Output tokens cost 5x more than input tokens.

```
Without llmcache:
  Turn 1: prompt + file.go (2,000 tokens)        → you pay for 2,000
  Turn 5: prompt + file.go still in history       → you pay for 2,000 again
  Turn 10: still carrying it                      → and again

With llmcache:
  Turn 1: prompt + summary (50 tokens)            → you pay for 50
  Turn 5: summary still in history                → you pay for 50
  Turn 10: still just 50 tokens                   → 97.5% smaller
```

---

## Prerequisites

- Claude Code
- GCP project with Vertex AI API enabled
- `gcloud auth application-default login`

---

## Installation

**1. Download the binary**

```bash
# macOS (Apple Silicon)
curl -sL https://github.com/sp98/llmcache-site/releases/latest/download/llmcache-darwin-arm64 -o llmcache
chmod +x llmcache
sudo mv llmcache /usr/local/bin/

# macOS (Intel)
curl -sL https://github.com/sp98/llmcache-site/releases/latest/download/llmcache-darwin-amd64 -o llmcache
chmod +x llmcache
sudo mv llmcache /usr/local/bin/

# Linux (amd64)
curl -sL https://github.com/sp98/llmcache-site/releases/latest/download/llmcache-linux-amd64 -o llmcache
chmod +x llmcache
sudo mv llmcache /usr/local/bin/
```

**2. Start the proxy**

```bash
export ANTHROPIC_VERTEX_PROJECT_ID=my-project
export CLOUD_ML_REGION=us-east5
llmcache serve &

# Point Claude Code at the proxy
export ANTHROPIC_VERTEX_BASE_URL="http://localhost:8080/v1"
```

Configure Claude Code in `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "llmcache": {
      "command": "/path/to/llmcache",
      "args": ["mcp"]
    }
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [
          {
            "type": "command",
            "command": "~/.llmcache/hooks/deny-read.sh"
          }
        ]
      }
    ]
  }
}
```

**3. Create the deny-read hook**

Save this as `~/.llmcache/hooks/deny-read.sh` and make it executable (`chmod +x`):

```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | grep -o '"file_path":"[^"]*"' | head -1 | cut -d'"' -f4)

if [ -z "$FILE_PATH" ]; then
  exit 0
fi

BASENAME="${FILE_PATH##*/}"
EXT="${FILE_PATH##*.}"
case "$BASENAME" in
  Makefile|makefile|GNUmakefile)
    echo '{"permissionDecision":"deny","permissionDecisionReason":"Use understand_file from the llmcache MCP server instead of Read for Makefiles."}'
    ;;
  *)
    case "$EXT" in
      go|py|js|ts|tsx|jsx|rs|java|c|cpp|cc|h|hpp|rb|sh|proto|sql|md)
        echo '{"permissionDecision":"deny","permissionDecisionReason":"Use understand_file or understand_package from the llmcache MCP server instead of Read for source files."}'
        ;;
    esac
    ;;
esac
```

This hook intercepts `Read` calls on source files and tells Claude to use `understand_file` instead. Claude can still use `Read` for config files, logs, and when it needs exact line numbers for editing.

**4. Add to `~/.claude/CLAUDE.md`:**

```
MANDATORY: ALWAYS use understand_file or understand_package from the
llmcache MCP server instead of Read to understand source files.
Only use Read when you need exact line numbers for editing.
```

---

## Commands

| Command | Description |
|---------|-------------|
| `llmcache serve` | Start the proxy server. Logs all requests, tracks token usage, forwards to Vertex AI. |
| `llmcache mcp` | Start the MCP server (spawned automatically by Claude Code via settings.json). |
| `llmcache index <repo>` | Pre-index a repository so the first session starts with a warm cache. Use `-w 10` for parallel workers, `--include "*.go"` to filter. |
| `llmcache stats` | Show usage stats — cache hits, input/output tokens saved, and dollar savings with per-model pricing. Use `--detail` for per-file breakdown. |
| `llmcache gc` | Remove stale summaries. Use `--max-age 720h` to set retention, `--dry-run` to preview. |

### MCP Tools (used by Claude Code automatically)

| Tool | Description |
|------|-------------|
| `understand_file` | Returns a cached summary of a source file. If the file has changed (git hash mismatch), generates a fresh summary. If uncommitted, tells Claude to read directly. |
| `understand_package` | Summarizes all source files in a directory — gives Claude a package-level overview in one call. |
| `get_file_detail` | Extracts a specific function or type from a file. Uses Go AST for Go files, pattern search for others. |
| `list_summaries` | Lists all cached summaries for a repository with hit counts and freshness. |

---

## Configuration

llmcache works with zero config beyond two required env vars. Optional overrides via `~/.llmcache/config.yaml`:

| Setting | Default | Description |
|---------|---------|-------------|
| `project` | — (required) | GCP project ID (`ANTHROPIC_VERTEX_PROJECT_ID`) |
| `region` | — (required) | GCP region (`CLOUD_ML_REGION`) |
| `listen` | `:8080` | Proxy listen address |
| `db` | `~/.llmcache/data.db` | SQLite database path |
| `summary_model` | `claude-haiku-4-5@20251001` | Model used for generating summaries |
