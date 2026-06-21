# llmcache

**Stop paying Claude to understand the same code it already understood yesterday.**

llmcache is an MCP server that caches AI-generated understanding of your source code. Claude reads each file once and every session after that starts from cached understanding, saving you tokens and money.

## Why It Matters

Claude Code uses a stateless API. On every turn, the **entire conversation history**, including every file Claude has read, is re-sent as input tokens. A 1,000-token file read on turn 1 costs you 1,000 input tokens on every subsequent turn for the rest of the session.

llmcache replaces that 1,000-token file with a 50-token summary. You save 950 tokens **on every turn**, not just once.

On top of that, Claude generates output tokens to process each file. With a cached summary, Claude skips that work entirely. Output tokens cost 5x more than input tokens.

```
Without llmcache:
  Turn 1: prompt + file.go (1,000 tokens)        > you pay for 1,000
  Turn 5: prompt + file.go still in history       > you pay for 1,000 again
  Turn 10: still carrying it                      > and again

With llmcache:
  Turn 1: prompt + summary (50 tokens)            > you pay for 50
  Turn 5: summary still in history                > you pay for 50
  Turn 10: still just 50 tokens                   > 95% smaller
```

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (only supported client)
- One of:
  - Anthropic API key (`ANTHROPIC_API_KEY`)
  - GCP project with Vertex AI API enabled + `gcloud auth application-default login`

## Install

Download the binary for your platform:

```bash
# macOS (Apple Silicon)
curl -L -o llmcache https://github.com/sp98/llmcache-site/releases/latest/download/llmcache-darwin-arm64

# macOS (Intel)
curl -L -o llmcache https://github.com/sp98/llmcache-site/releases/latest/download/llmcache-darwin-amd64

# Linux (x86_64)
curl -L -o llmcache https://github.com/sp98/llmcache-site/releases/latest/download/llmcache-linux-amd64
```

```bash
chmod +x llmcache
sudo mv llmcache /usr/local/bin/
```

## How It Works

1. Claude Code calls `understand_file` (via MCP) instead of `Read`
2. llmcache checks if a cached summary exists and is still fresh (git hash match)
3. Cache hit: returns the compact summary (typically 90%+ smaller than the full file)
4. Cache miss: calls Claude Haiku to generate a summary, caches it, returns it

Summaries auto-invalidate when the file's git hash changes.

## Solo Setup (Free)

Local cache stored in SQLite. No server, no license key needed.

**1. Add the MCP server**

```bash
claude mcp add --transport stdio llmcache -- llmcache mcp
```

For Vertex AI users, pass the region:

```bash
claude mcp add \
  --env CLOUD_ML_REGION=us-east5 \
  --transport stdio llmcache -- llmcache mcp
```

**2. Add the Read hook**

This hook tells Claude to use `understand_file` instead of reading source files directly. Add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [
          {
            "type": "command",
            "command": "F=$(cat | grep -o '\"file_path\":\"[^\"]*\"' | head -1 | cut -d'\"' -f4); B=${F##*/}; E=${F##*.}; case \"$B\" in Makefile|makefile|GNUmakefile) echo '{\"permissionDecision\":\"deny\",\"permissionDecisionReason\":\"Use understand_file from llmcache instead.\"}';; *) case \"$E\" in go|py|js|ts|tsx|jsx|rs|java|c|cpp|cc|h|hpp|rb|sh|proto|sql|md) echo '{\"permissionDecision\":\"deny\",\"permissionDecisionReason\":\"Use understand_file or understand_package from llmcache instead.\"}';; esac;; esac"
          }
        ]
      }
    ]
  }
}
```

**3. Add the CLAUDE.md instruction**

Append to `~/.claude/CLAUDE.md`:

```
MANDATORY: ALWAYS use understand_file or understand_package from the
llmcache MCP server instead of Read to understand source files.
Only use Read when you need exact line numbers for editing.
```

**4. Verify**

Restart Claude Code, use it on a project, then check savings:

```bash
llmcache stats
llmcache stats --detail    # per-file breakdown
llmcache stats --monthly   # month-by-month
```

### Configuration

Optional settings via env vars or `~/.llmcache/config.yaml`:

| Env Var | Default | Description |
|---------|---------|-------------|
| `LLMCACHE_DB` | `~/.llmcache/data.db` | SQLite database path |
| `LLMCACHE_LOG_DIR` | `~/.llmcache/` | Log directory |
| `LLMCACHE_SUMMARY_MODEL` | `claude-haiku-4-5@20251001` | Model for generating summaries |

## Team Setup (Paid)

Shared cache backed by Postgres so multiple developers share summaries across the team. Requires a license key.

**Why Team mode?** In Solo mode, each developer builds their own cache from scratch. When developer A understands a file, developer B still pays full tokens to understand the same file. With Team mode, summaries are shared: one developer pays the generation cost, everyone else gets instant cache hits. The more developers on the team, the faster the cache warms up and the greater the savings. On a typical team, the shared cache reaches 90%+ hit rate within days.

**Pre-indexing repositories.** Admins can pre-index repositories so the cache is warm before developers start working:

```bash
llmcache index /path/to/repo -w 10
```

Developers get instant cache hits from their first session, with zero cold-start cost.

### Server setup (admin)

The team admin runs the llmcache team server connected to a Postgres database:

```bash
llmcache team --listen :8090 --db-url "postgres://user:pass@host:5432/llmcache?sslmode=disable"
```

Required env vars on the server:

| Env Var | Description |
|---------|-------------|
| `LLMCACHE_LICENSE_KEY` | License key (provided on purchase) |
| `LLMCACHE_TEAM_API_KEY` | API key for authenticating clients |

The server creates its tables on first start. Admin dashboard is available at `http://<server>:8090/admin/`.

### Developer setup

Each developer installs the llmcache binary (see Install above), then connects to the team server.

**1. Connect to the team server**

Your team admin will provide the server URL, API key, and license key.

```bash
claude mcp add \
  --env LLMCACHE_TEAM_SERVER_URL=<server-url> \
  --env LLMCACHE_TEAM_API_KEY=<api-key> \
  --env LLMCACHE_LICENSE_KEY=<license-key> \
  --env CLOUD_ML_REGION=us-east5 \
  --transport stdio llmcache -- llmcache mcp
```

**2. Add the Read hook and CLAUDE.md instruction**

Same as Solo Setup steps 2 and 3 above.

**3. Verify**

Restart Claude Code after setup. Verify with `llmcache stats`. Should show **Team Cache Savings**.

## CLI Commands

| Command | Description |
|---------|-------------|
| `llmcache mcp` | Start the MCP server (spawned automatically by Claude Code). |
| `llmcache stats` | Show cache hits, input/output tokens saved. Use `--detail` for per-file breakdown, `--monthly` for trends. |
| `llmcache index <repo>` | Pre-index a repository so the first session starts with a warm cache. Use `-w 10` for parallel workers, `--include "*.go"` to filter. |
| `llmcache gc` | Remove stale summaries. Use `--max-age 720h` to set retention, `--dry-run` to preview. |

### MCP Tools (used by Claude Code automatically)

| Tool | Description |
|------|-------------|
| `understand_file` | Returns a cached summary of a source file. If the file has changed (git hash mismatch), generates a fresh summary. If uncommitted, tells Claude to read directly. |
| `understand_package` | Summarizes all source files in a directory, giving Claude a package-level overview in one call. |
| `get_file_detail` | Extracts a specific function or type from a file. Uses Go AST for Go files, pattern search for others. |
| `list_summaries` | Lists all cached summaries for a repository with hit counts and freshness. |

## Bugs & Feedback

Found a bug or have a feature request? [Open an issue on GitHub](https://github.com/sp98/llmcache-site/issues).
