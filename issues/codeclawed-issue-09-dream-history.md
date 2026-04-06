# Issue 9: Dream System Can't Read Chat History

## Status
`extract-session-dialogue.py` (the OpenClaw script that extracts conversation turns for dream consolidation) reads from OpenClaw's session directory structure (`~/.openclaw/agents/<id>/sessions/`). This path doesn't exist in the codeclawed system. The dream system cannot run without this being rewritten.

## Relationship to Issue 4
Issue 4 covers porting all five dream scripts. This issue is specifically about the data format problem — how the dream system gets conversation data in the new architecture.

## The Data Flow

### OpenClaw (old)
```
OpenClaw daemon stores sessions → ~/.openclaw/agents/<id>/sessions/
extract-session-dialogue.py reads sessions → outputs dialogue
dream-preprocessor.py condenses dialogue → outputs dream prompt
dream-gate.sh invokes agent with dream prompt
```

### CodeClawed (new)
```
Chat server stores exchanges → output/agents/<id>/chat-history.jsonl
Cron job outputs → delivered via /api/deliver → also in chat-history.jsonl
                → or logged to output/logs/<slug>.log (NOT in chat history)
```

## The Problem

### chat-history.jsonl format
```json
{"id":"uuid","agentId":"main","role":"user","content":"...","timestamp":"2026-04-05T12:00:00Z","source":"chat"}
{"id":"uuid","agentId":"main","role":"assistant","content":"...","timestamp":"2026-04-05T12:00:05Z","source":"chat"}
```

### What's missing from chat-history.jsonl
- **Cron job outputs that aren't delivered to chat**: Wrapper scripts call `agent-runner-cli` directly. Output goes to `output/logs/<slug>.log`. Only if the delivery section POSTs to `/api/deliver` does the output end up in chat history. Many jobs may not have delivery configured.
- **Headless agent-runner calls**: Any direct call to `agent-runner-cli` from scripts or sentinels won't appear in chat history.

## What Needs to Happen

### 1. Ensure all agent interactions are logged
Options:
- **A**: Have `agent-runner-cli` always append to `chat-history.jsonl` (regardless of whether the chat server is involved). This makes the history file the single source of truth.
- **B**: Have the dream system read from BOTH `chat-history.jsonl` AND `output/logs/*.log`. More complex but doesn't change agent-runner.

Option A is cleaner. Add a `--log-to` flag to `agent-runner-cli`, or have it always append to the agent's `chat-history.jsonl` when the working directory is under `output/agents/`.

### 2. Rewrite `extract-session-dialogue.py`
- Read from `chat-history.jsonl` instead of OpenClaw session directories
- Parse JSONL format
- Filter by date range (accept `--since` and `--until` arguments)
- Output in the format `dream-preprocessor.py` expects
- Handle the `source` field: include `chat` and `cron` sources, potentially exclude `sentinel` (or make configurable)

### Files to modify
- `lib/agent-runner-cli.ts` — add chat-history logging (Option A)

### Files to create
- `scripts/extract-session-dialogue.py` — rewritten version

### Files to read first
- `~/.openclaw/scripts/extract-session-dialogue.py` — original to understand output format
- `~/.openclaw/scripts/dream-preprocessor.py` — to understand expected input format
- `~/.openclaw/scripts/dream-gate.sh` — to understand how these chain together

### Notes
- Claude Code also stores its own session data at `~/.claude/sessions/` but this is internal and may not be stable to read from. The chat-history.jsonl approach is more reliable.
- The `session_id` we now capture for resume (stored in `output/sessions.json`) could theoretically be used to read Claude's native session data, but that's fragile.
