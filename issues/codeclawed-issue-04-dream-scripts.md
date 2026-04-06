# Issue 4: Dream System Scripts Not Ported

## Status
The five dream/reflection scripts are not in the codeclawed project at all. These are the nightly memory consolidation system — a core part of the agent lifecycle.

## Source Scripts (at `~/.openclaw/scripts/`)

### `dream-gate.sh`
Nightly orchestrator. Runs at ~3:00 AM. For each agent:
1. Calls `session-health-report.sh` to check if the agent had activity
2. Calls `extract-session-dialogue.py` to pull conversation turns
3. Calls `dream-preprocessor.py` to condense into dream prompts
4. Invokes the agent with the dream prompt via OpenClaw cron

### `reflection-gate.sh`
Weekly orchestrator. Runs Sundays at noon. Similar pattern to dream-gate but produces weekly reflection summaries instead of nightly dreams.

### `dream-preprocessor.py`
Python script. Takes raw session dialogue and condenses it into a structured dream prompt. Extracts key topics, emotional themes, unresolved questions.

### `extract-session-dialogue.py`
Python script. Reads OpenClaw session files and extracts user/assistant turns. **This is the script that needs the most adaptation** — it currently reads from `~/.openclaw/agents/<id>/sessions/` which doesn't exist in the new system.

### `session-health-report.sh`
Shell script. Checks session activity metrics to determine if an agent was active enough to warrant a dream cycle.

## What Needs to Happen

### 1. Copy and adapt `dream-gate.sh`
- Replace `$OPENCLAW cron run <ID>` calls with `agent-runner-cli` invocations
- Replace `$OC_HOME` with `$CODECLAWED_HOME`
- Update agent directory paths from `workspace-<id>` to `agents/<id>`
- Source `env.sh` from the new location

### 2. Copy and adapt `reflection-gate.sh`
Same pattern as dream-gate.

### 3. Port `extract-session-dialogue.py` (significant rewrite)
This is the biggest change. In OpenClaw, sessions live at structured paths managed by the daemon. In codeclawed:
- Interactive chat history is in `output/agents/<id>/chat-history.jsonl`
- Each line is a JSON object: `{id, agentId, role, content, timestamp, source}`
- Headless `claude -p` calls (from cron jobs) don't write to chat-history unless they go through the chat server's `/api/deliver` endpoint

The script needs to:
- Read `chat-history.jsonl` instead of OpenClaw session directories
- Parse the JSONL format
- Filter by date range (last 24h for dreams, last 7d for reflections)
- Output the same format the dream-preprocessor expects

### 4. Copy `dream-preprocessor.py` (minimal changes)
Likely only path updates. The input format may need adjustment based on what `extract-session-dialogue.py` outputs.

### 5. Copy and adapt `session-health-report.sh`
Update to check `chat-history.jsonl` for activity instead of OpenClaw session files.

### Target location
`scripts/` in the project root (source templates), and deployed to `output/scripts/` during generation.

### Files to read first
- `~/.openclaw/scripts/dream-gate.sh`
- `~/.openclaw/scripts/reflection-gate.sh`
- `~/.openclaw/scripts/dream-preprocessor.py`

Note: `extract-session-dialogue.py` — the original is at `~/.openclaw/scripts/` but the path may also be referenced in dream-gate.sh. Read dream-gate.sh first to understand how these scripts chain together.

### Dependencies
- The chat server must persist all agent interactions to `chat-history.jsonl` (already done)
- Cron job outputs should also be persisted (currently only if delivered via `/api/deliver`)
