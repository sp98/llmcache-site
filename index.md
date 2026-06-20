---
layout: default
---

## Claude Code re-reads your codebase every session. llmcache makes it stop.

llmcache is a local MCP server that caches AI-generated understanding of your source code. Claude reads each file once and every session after that starts from cached understanding, saving you tokens and money.

---

## How It Works

**First read** — Claude calls `understand_file`. llmcache reads the file, sends it to Haiku for a compact summary, and caches it by git hash.

**Every read after** — llmcache serves the cached summary instantly. The raw file never enters the conversation. Cache invalidates automatically when the file changes (git hash mismatch).

**What you save** — smaller tool results mean fewer input tokens per turn AND fewer output tokens (Claude skips generating its own understanding). Both savings compound across every subsequent API call in the conversation.

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

## Real Numbers

From daily development on a large Go project (~1,500 source files):

| Metric | Value |
|--------|-------|
| Input tokens saved per day | ~662,000 |
| Summaries cached | 1,500+ |
| Cache hit rate | Structural (every repeat read is a hit) |

### Projected Monthly Savings

| Scenario | Solo (Sonnet) | Solo (Opus) | Team of 10 (Sonnet) | Team of 10 (Opus) |
|----------|--------------|------------|--------------------|--------------------|
| Conservative | ~$22/mo | ~$109/mo | ~$215/mo | ~$1,085/mo |
| Expected | ~$44/mo | ~$218/mo | ~$430/mo | ~$2,170/mo |

These numbers are **conservative** — they only count the direct input token delta per cache hit. Actual savings are higher due to output token savings and compounding across conversation turns.

---

## Why It Saves More Than You Think

Claude Code uses a stateless API. On every turn, the **entire conversation history** — including every file Claude has read — is re-sent as input tokens. A 2,000-token file read on turn 1 costs you 2,000 input tokens on every subsequent turn for the rest of the session.

llmcache replaces that 2,000-token file with a 50-token summary. You save 1,950 tokens **on every turn**, not just once.

On top of that, Claude generates output tokens to process each file — intermediate understanding, analysis, reasoning. With a cached summary, Claude skips that work entirely. Output tokens cost 5x more than input tokens.

| What's saved | Tracked in stats? | Rate (Opus) |
|--------------|-------------------|-------------|
| Input tokens (direct) | Yes | $5/MTok |
| Output tokens (skipped understanding) | Yes | $25/MTok |
| Input tokens (compounding across turns) | No — actual savings are higher | $5/MTok |

---

## Features

- **Git-aware invalidation** — cache keys are git hashes. Change a file, the old summary is automatically stale. No manual cache clearing.
- **Per-model pricing** — stats show savings based on the actual model you're using (Haiku, Sonnet, or Opus).
- **Pre-indexing** — run `llmcache index` to cache your entire repo before you start coding. First session is instantly fast.
- **Zero config** — set two environment variables and go. Everything else has sensible defaults.
- **Stats dashboard** — `llmcache stats` shows hits, tokens saved, and dollar savings with per-model pricing.
- **Docker support** — single container, mounts a volume for the SQLite database.

---

## Quick Start

**1. Build and start the proxy**

```bash
go build -o llmcache .
export ANTHROPIC_VERTEX_PROJECT_ID=my-project
export CLOUD_ML_REGION=us-east5
./llmcache serve &
```

**2. Configure Claude Code** (in `~/.claude/settings.json`)

```json
{
  "mcpServers": {
    "llmcache": {
      "command": "/path/to/llmcache",
      "args": ["mcp"]
    }
  }
}
```

**3. Start coding** — Claude automatically uses cached summaries via `understand_file` and `understand_package`.

Check your savings:

```bash
llmcache stats
```

```
 MCP Summaries  (8 cached)
╭───────────┬──────┬────────────────────┬─────────────────────┬─────────╮
│           │ Hits │ Input Tokens Saved │ Output Tokens Saved │ Saved * │
├───────────┼──────┼────────────────────┼─────────────────────┼─────────┤
│ today     │ 6    │ 30,306             │ 5,756               │ $0.31   │
│ this week │ 6    │ 30,306             │ 5,756               │ $0.31   │
│ all time  │ 6    │ 30,306             │ 5,756               │ $0.31   │
╰───────────┴──────┴────────────────────┴─────────────────────┴─────────╯
  * Savings based on claude-opus-4-6 pricing — $5.00/MTok input, $25.00/MTok output
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│ YOU: ask Claude a question about your code              │
├─────────────────────────────────────────────────────────┤
│ CLAUDE CODE: decides what to read                       │
│   ├── understand_file    → llmcache MCP (cached)        │
│   ├── understand_package → llmcache MCP (cached)        │
│   └── sends prompt to API with compact summaries        │
├─────────────────────────────────────────────────────────┤
│ llmcache MCP SERVER:                                    │
│   ├── git hash match? → return cached summary           │
│   ├── new/changed?    → summarize with Haiku → cache    │
│   └── uncommitted?    → tell Claude to read directly    │
├─────────────────────────────────────────────────────────┤
│ llmcache PROXY:                                         │
│   ├── logs all requests (tokens, model, timing)         │
│   └── forwards to Vertex AI                             │
└─────────────────────────────────────────────────────────┘
```

---

## Requirements

- Go 1.26+
- Claude Code with Vertex AI access
- GCP project with Vertex AI API enabled
- `gcloud auth application-default login`
