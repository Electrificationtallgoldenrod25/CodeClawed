# OpenClaw to Claude Code Migration — Build Prompt

You are building a TypeScript CLI tool that migrates an OpenClaw multi-agent deployment to run natively on Claude Code. The tool analyzes the existing OpenClaw installation, converts its file formats and configuration, and produces a fully functional Claude Code project with a web chat frontend.

This document contains everything you need to build the tool from scratch. Read it fully before starting.

---

## Background

### OpenClaw
OpenClaw is a Node.js daemon (port 18789) that manages multiple AI agents. Each agent has a workspace directory containing markdown identity and behavior files. OpenClaw handles conversation sessions, cron-scheduled tasks, model routing, tool restrictions, and channel bindings (typically Discord). Configuration lives in a central `openclaw.json`.

### Claude Code
Claude Code is Anthropic's stateless CLI. The headless mode (`claude -p`) takes a prompt, runs to completion, and exits. There is no daemon. Identity is defined by a `CLAUDE.md` file (max 200 lines, auto-discovered from the working directory) plus `.claude/rules/*.md` files for additional context. MCP servers provide tool extensions. Scheduling must use OS primitives (launchd on macOS).

### The migration challenge
OpenClaw is stateful and daemon-driven. Claude Code is stateless and invocation-driven. The migration tool bridges this gap by:
1. Converting OpenClaw's identity files into Claude Code's format
2. Abstracting all agent invocations through a single runner function
3. Converting cron schedules to OS-level schedulers
4. Building a web chat frontend to replace Discord channel bindings
5. Setting up a memory system (files + optional pgvector) for persistence across stateless calls

---

## Architecture

### Project structure

```
<project>/
├── src/                    # Migration CLI
│   ├── cli.ts              # Entry point (commander): analyze, generate, verify, install, chat
│   ├── types.ts            # All shared TypeScript interfaces
│   ├── constants.ts        # Tool mappings, model mappings, limits
│   ├── analyze/            # Phase 1: Parse OpenClaw deployment
│   │   ├── index.ts        # Orchestrator
│   │   ├── workspace.ts    # Scan workspace directories, read .md files
│   │   └── cron.ts         # Parse cron/jobs.json, filter, categorize
│   ├── generate/           # Phase 2: Produce Claude Code project
│   │   ├── index.ts        # Orchestrator: iterate agents, generate shared infra
│   │   ├── agent-project.ts  # Generate one agent's full directory
│   │   └── converters/     # Per-concern transformation modules
│   │       ├── identity.ts # SOUL.md + USER.md + IDENTITY.md → CLAUDE.md
│   │       ├── rules.ts    # AGENTS.md → .claude/rules/*.md
│   │       ├── tools.ts    # Tool deny list → --disallowed-tools mapping
│   │       ├── cron.ts     # Cron jobs → wrapper scripts + launchd plists
│   │       └── memory.ts   # Copy memory files, generate SQL seed
│   ├── verify/             # Phase 3: Validate output
│   │   └── index.ts
│   └── util/
│       ├── markdown.ts     # Section parser, line counter, heading cleaner
│       ├── cron-parser.ts  # Cron expr → launchd StartCalendarInterval/StartInterval
│       └── openclaw-paths.ts  # Resolve OC_HOME, workspace paths
├── lib/                    # Runtime libraries (also deployed to output)
│   ├── agent-runner.ts     # THE single abstraction wrapping `claude -p`
│   ├── agent-runner-cli.ts # Shell-callable CLI wrapper for agent-runner
│   ├── scheduler/
│   │   ├── types.ts        # Platform-agnostic Scheduler interface
│   │   └── launchd.ts      # macOS launchd implementation
│   └── memory/
│       ├── schema.sql      # pgvector table definition
│       ├── mcp-server.ts   # MCP server exposing memory_search, memory_store, memory_list
│       └── sync.ts         # File watcher syncing .md memory files to database
├── chat/                   # Web frontend replacing Discord
│   ├── server.ts           # Express + WebSocket, session resumption
│   └── public/
│       └── index.html      # Single-page dark-themed chat UI
├── package.json
└── tsconfig.json
```

### Generated output structure (per migration run)

```
output/
├── agents/
│   └── <agent-id>/
│       ├── CLAUDE.md                 # Merged identity (<200 lines)
│       ├── .claude/
│       │   ├── rules/
│       │   │   ├── workflow.md       # Session startup, responsibilities
│       │   │   ├── memory.md         # Memory management rules
│       │   │   ├── style.md          # Communication rules
│       │   │   ├── safety.md         # Boundaries, red lines
│       │   │   ├── domain.md         # Domain-specific rules (if any)
│       │   │   ├── tool-restrictions.md  # Migration notes for unmapped tools
│       │   │   └── identity-overflow.md  # Overflow if CLAUDE.md > 200 lines
│       │   └── mcp_config.json       # Points to memory MCP server
│       ├── memory/                   # Copied from OpenClaw workspace
│       └── shared-memory → ../../shared-memory
├── shared-memory/                    # Cross-agent knowledge base
├── lib/memory/seed.sql               # Database seed from memory files
├── scripts/
│   ├── env.sh                        # Environment config
│   └── jobs/                         # Per-cron-job wrappers
│       ├── <slug>.sh
│       ├── <slug>.plist
│       └── messages/<slug>.md
├── logs/
├── sessions.json                     # Active Claude Code session IDs per agent
└── config.json                       # Master config for the deployment
```

---

## OpenClaw Source Files

The migration tool reads from a standard OpenClaw installation (default `~/.openclaw/`):

### `openclaw.json` — Master configuration
```typescript
interface OpenClawConfig {
  agents: {
    defaults: {
      model: { primary: string };      // e.g. "anthropic/claude-sonnet-4-6"
      workspace: string;               // default workspace path
      memorySearch: MemorySearchConfig;
    };
    list: AgentDefinition[];           // array of agent configs
  };
  bindings: ChannelBinding[];          // Discord channel → agent mappings
}

interface AgentDefinition {
  id: string;                          // e.g. "main", "forge", "health"
  workspace: string;                   // workspace directory name
  model?: { primary: string };         // override model
  subagents?: { model: string };       // sub-agent model
  tools?: { deny?: string[] };         // denied tool names
}
```

### Workspace directories (`workspace/`, `workspace-<agent>/`)
Each agent has a workspace. The default workspace applies to agents that don't specify their own. Per-agent workspaces override or extend the default.

Standard files in each workspace:
| File | Purpose | Migration target |
|------|---------|-----------------|
| `SOUL.md` | Core personality, boundaries, epistemic stance | Merged into `CLAUDE.md` |
| `USER.md` | User context (who the user is, their needs) | Merged into `CLAUDE.md` |
| `IDENTITY.md` | Agent-specific name, role, emoji | Merged into `CLAUDE.md` |
| `AGENTS.md` | Operating manual: startup, memory rules, red lines, communication | Sharded into `.claude/rules/*.md` |
| `TOOLS.md` | Local tool/environment notes | Copied to `.claude/rules/tools-reference.md` |
| `MEMORY.md` | Curated long-term memory index | Copied to `memory/MEMORY.md` |
| `HEARTBEAT.md` | Proactive task list (if any) | Converted to scheduling rules |
| `memory/` | Daily memory files, observations | Copied to `memory/` |

### `cron/jobs.json` — Scheduled tasks
```typescript
interface CronJob {
  id: string;
  agentId: string;
  name: string;
  enabled: boolean;
  schedule: { kind: 'cron'; expr: string; tz: string } | { kind: 'at'; at: string };
  payload: { message: string; model?: string; thinking?: string; timeoutSeconds?: number };
  delivery: { mode: string; channel?: string; to?: string };
}
```

### `shared-memory/` — Cross-agent knowledge
Directory of `from-<agent>.md` files. Each agent can read these. Deployed as a shared directory with symlinks into each agent's workspace.

---

## Core Conversion Logic

### 1. Identity conversion (SOUL.md + USER.md + IDENTITY.md → CLAUDE.md)

Parse `IDENTITY.md` for structured fields (name, role, emoji) using the pattern:
```
- **Name**: Forge
- **Role**: Agent Builder & Infrastructure Admin
- **Emoji**: ...
```

Parse `SOUL.md` into sections by heading. Extract and order:
1. Preamble (text before first heading — often the most important identity statement)
2. Core truths / identity section
3. Boundaries / constraints
4. Epistemic standards
5. Vibe / personality
6. Any remaining sections

Parse `USER.md` into sections. Include all.

Assemble into a single markdown document. If over 200 lines, move the detailed user context sections to `.claude/rules/identity-overflow.md` and leave a pointer in CLAUDE.md.

### 2. Rules conversion (AGENTS.md → .claude/rules/*.md)

Parse AGENTS.md by headings. Route each section to a target file based on heading text:

| Heading pattern | Target file |
|----------------|-------------|
| Session Startup, Bootstrap, First Run | `workflow.md` |
| Memory, Remember | `memory.md` |
| Red Line, Boundary, Safety | `safety.md` |
| Communication, Group Chat, Platform, Discord | `style.md` |
| Heartbeat, Cron, Schedule, Proactive | `scheduling.md` |
| Tool, Skill, Building | `workflow.md` |
| Responsibility, Purpose, Role | `workflow.md` |
| (domain-specific, no match) | `domain.md` |

Rewrite session startup instructions: remove "Read SOUL.md" / "Read USER.md" (these are now in CLAUDE.md which is auto-discovered). Renumber remaining steps.

### 3. Tool deny list conversion

Map OpenClaw tool names to Claude Code `--disallowed-tools` values:

| OpenClaw tool | Claude Code equivalent |
|--------------|----------------------|
| `browser` | `WebFetch`, `WebSearch` |
| `subagents`, `sessions_spawn` | `Bash(claude*)` (prevents spawning sub-agents) |
| `cron` | No equivalent (cron is OS-level, not a Claude Code tool) |
| `sessions_send/list/history/yield` | No equivalent |
| `image_generate`, `image` | No equivalent |
| `process` | No equivalent |

Generate a `tool-restrictions.md` rule file listing unmapped denials as documentation.

### 4. Cron conversion

For each recurring cron job (schedule.kind === 'cron'):

**Wrapper script** (`scripts/jobs/<slug>.sh`):
```bash
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/../env.sh"

MESSAGE_FILE="${SCRIPT_DIR}/messages/<slug>.md"
OUTPUT=$(node "${CODECLAWED_HOME}/lib/agent-runner-cli.js" \
  --agent "<agentId>" \
  --model "<model>" \
  --message-file "$MESSAGE_FILE" \
  --working-directory "${CODECLAWED_HOME}/agents/<agentId>" \
  --timeout "<timeout>" \
  --skip-permissions)

echo "$OUTPUT"

# Optional: deliver to chat server
curl -s -X POST "http://localhost:${CHAT_PORT:-3456}/api/deliver" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg agent "<agentId>" --arg content "$OUTPUT" '{agentId: $agent, content: $content}')"
```

**Message file** (`scripts/jobs/messages/<slug>.md`): the raw payload.message from the cron job.

**Launchd plist**: Convert cron expression to launchd format:
- `*/N * * * *` → `<key>StartInterval</key><integer>N*60</integer>`
- Fixed times → `<key>StartCalendarInterval</key><dict>` with Minute, Hour, Day, Month, Weekday keys
- Ranges (e.g., `1-5` for weekdays) → expand to multiple CalendarInterval dicts
- Lists (e.g., `3,15`) → expand to multiple CalendarInterval dicts
- If the source job has `enabled: false`, set `<key>Disabled</key><true/>` in the plist

### 5. Memory conversion

- Copy each agent's `memory/` directory contents to `output/agents/<id>/memory/`
- Copy `MEMORY.md` into the memory directory
- Copy `shared-memory/` to `output/shared-memory/`
- Create symlinks from each agent's directory to shared-memory
- Generate `seed.sql` with INSERT statements for all memory file contents (for pgvector database)

---

## Agent Runner — The Critical Abstraction

**Every component in the system invokes agents through a single function.** This is the most important architectural decision.

```typescript
// lib/agent-runner.ts
interface AgentRunOptions {
  agentId: string;
  model: string;                    // "opus" | "sonnet" | "haiku"
  message: string;
  workingDirectory: string;         // agents/<id>/ — Claude Code reads CLAUDE.md from here
  systemPrompt?: string;            // --append-system-prompt
  disallowedTools?: string[];       // --disallowed-tools
  timeout?: number;
  skipPermissions?: boolean;        // --dangerously-skip-permissions
  outputFormat?: 'text' | 'stream-json';
  resumeSessionId?: string;         // --resume <session-id>
}
```

The function builds CLI arguments and spawns:
```
claude -p --model <model> [--resume <id>] [--output-format stream-json --verbose] [--dangerously-skip-permissions] [--append-system-prompt "..."]
```

With `cwd` set to the agent's directory. Claude Code auto-discovers CLAUDE.md from cwd. The message is written to stdin and stdin is closed.

Key details:
- `--output-format stream-json` **requires** `--verbose` — without it, the command silently fails
- Stream-json output format: each line is a JSON object. Text content is at `message.content[].text` for assistant messages, and `result` field for the final result
- The `session_id` appears in the first line (type: "system", subtype: "init")

Two exported functions:
- `runAgent(options)` — returns a Promise<AgentResult> with full stdout/stderr after completion
- `runAgentStreaming(options, onChunk)` — returns `{ process, result }` and calls onChunk for each stdout data event

Also provide `lib/agent-runner-cli.ts` — a CLI wrapper so shell scripts can call:
```bash
node agent-runner-cli.js --agent main --model opus --message "..." --skip-permissions
```

---

## Chat Server — Session Resumption

The chat server replaces Discord as the user-facing interface.

### Session management
Claude Code sessions persist across `claude -p` calls via `--resume <session-id>`. The chat server:

1. **First message to an agent**: Runs `claude -p` normally. Captures `session_id` from the stream-json init event. Stores it in `sessions.json` (agentId → sessionId).
2. **Subsequent messages**: Passes `--resume <session_id>`. Claude Code loads the full prior conversation — no need to inject history via system prompt.
3. **Resume failure**: If the session expired or errored, automatically falls back to a fresh session.
4. **"New Session" button**: Client sends `{type: "new_session", agentId}`. Server clears the stored session ID.

### WebSocket protocol
- Client sends: `{type: "message", agentId: "main", text: "hello"}`
- Server sends: `{type: "stream_start", agentId, resuming: bool}`
- Server sends: `{type: "stream_chunk", chunk: "<raw stream-json line>"}` (repeated)
- Server sends: `{type: "stream_end", agentId, exitCode, durationMs, sessionId}`
- Server sends: `{type: "error", error: "..."}`
- Cron deliveries: `{type: "delivery", message: ChatMessage}`

### Chat history
All exchanges persisted to `output/agents/<id>/chat-history.jsonl`:
```jsonl
{"id":"uuid","agentId":"main","role":"user","content":"...","timestamp":"ISO","source":"chat"}
{"id":"uuid","agentId":"main","role":"assistant","content":"...","timestamp":"ISO","source":"chat"}
```

The assistant content is extracted as plain text from the stream-json result (not raw stream-json).

### REST endpoints
- `GET /api/agents` — list agents from config.json
- `GET /api/history/:agentId` — last 50 messages from chat-history.jsonl
- `POST /api/deliver` — webhook for cron/script output delivery

### Frontend
Single-page HTML with inline CSS and JavaScript. Dark theme. Left sidebar with agent list (name + model badge). Main area with streaming message display. Enter to send, Shift+Enter for newline. Status bar shows session state.

---

## Scheduler Abstraction

Abstract interface so launchd (macOS) can be swapped for systemd (Linux) later:

```typescript
interface Scheduler {
  install(job: ScheduledJob): Promise<void>;
  uninstall(jobId: string): Promise<void>;
  list(): Promise<ScheduledJob[]>;
  status(jobId: string): Promise<JobStatus>;
  enable(jobId: string): Promise<void>;
  disable(jobId: string): Promise<void>;
}
```

macOS implementation writes plists to `~/Library/LaunchAgents/com.<tool-name>.<job-id>.plist` and uses `launchctl load/unload`.

---

## Memory System

### File-based (primary, always available)
Agent memory files at `output/agents/<id>/memory/*.md` are human-readable and the source of truth. Agents can read and write these directly using Claude Code's file tools.

### pgvector (optional, for semantic search)
MCP server exposes memory tools to agents:
- `memory_search(query, agent_id?, limit?)` — full-text search (vector search requires embedding provider)
- `memory_store(content, file_path?)` — insert/upsert
- `memory_list(agent_id?)` — list entries

Sync daemon watches memory files and upserts changes to the database.

Schema:
```sql
CREATE TABLE memory_entries (
  id SERIAL PRIMARY KEY,
  agent_id TEXT NOT NULL,
  file_path TEXT NOT NULL,
  content TEXT NOT NULL,
  embedding vector(1536),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  metadata JSONB DEFAULT '{}'
);
CREATE UNIQUE INDEX idx_memory_agent_file ON memory_entries(agent_id, file_path);
```

Each agent's `.claude/mcp_config.json` points to the MCP server with `AGENT_ID` scoped:
```json
{
  "mcpServers": {
    "memory": {
      "command": "node",
      "args": ["<output>/lib/memory/mcp-server.js"],
      "env": { "AGENT_ID": "<id>", "DATABASE_URL": "postgresql://localhost:5432/<dbname>" }
    }
  }
}
```

---

## Verification

The `verify` command checks the generated output:
- `config.json` exists and is valid JSON
- Each agent has `CLAUDE.md` under 200 lines
- Each agent has `.claude/mcp_config.json` (valid JSON)
- Each agent has at least one rule file in `.claude/rules/`
- Each cron wrapper script has a matching message file
- `scripts/env.sh` exists
- Working directories referenced in config.json exist
- Models are recognized Claude Code values

---

## CLI Commands

```bash
<tool> analyze [--oc-home ~/.openclaw] [--format text|json]
<tool> generate [--oc-home ~/.openclaw] [--output ./output] [--dry-run]
<tool> verify [--output ./output]
<tool> install [--output ./output] [--jobs <categories>] [--dry-run]
<tool> chat [--port 3456] [--output ./output]
```

---

## Dependencies

```json
{
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.12.1",
    "chalk": "^5.4.1",
    "chokidar": "^4.0.3",
    "commander": "^13.1.0",
    "cron-parser": "^5.0.6",
    "express": "^5.1.0",
    "handlebars": "^4.7.8",
    "pg": "^8.14.1",
    "pgvector": "^0.2.0",
    "ws": "^8.18.1",
    "zod": "^3.24.4"
  },
  "devDependencies": {
    "@types/express": "^5.0.2",
    "@types/node": "^22.15.2",
    "@types/pg": "^8.11.11",
    "@types/ws": "^8.18.1",
    "tsx": "^4.19.4",
    "typescript": "^5.8.3",
    "vitest": "^3.1.2"
  }
}
```

TypeScript config: ES2022, NodeNext, strict, outDir dist/.

---

## Build Order

| Step | What | Depends on |
|------|------|-----------|
| 1 | types.ts, constants.ts, util/* | — |
| 2 | analyze/* | Step 1 |
| 3 | agent-runner.ts + CLI wrapper | — |
| 4 | Identity converter | Step 2 |
| 5 | Rules converter | Step 2 |
| 6 | Tools converter | Step 1 |
| 7 | Memory converter | Step 2 |
| 8 | Cron converter | Step 1 |
| 9 | generate/index.ts + agent-project.ts | Steps 4-8 |
| 10 | verify/* | Step 9 |
| 11 | cli.ts | Steps 2, 9, 10 |
| 12 | scheduler/launchd.ts | — |
| 13 | memory/mcp-server.ts + sync.ts + schema.sql | — |
| 14 | chat/* | Step 3 |

Steps 3, 12, 13 can run in parallel with steps 4-11.

---

## Key Gotchas

1. **`--output-format stream-json` requires `--verbose`** in Claude Code CLI. Without it, the command fails silently to stderr.

2. **Claude Code's CLAUDE.md limit is 200 lines.** Exceeding this doesn't error but degrades performance. Always check and overflow to rules files.

3. **`claude -p` is stateless.** Each invocation starts fresh. Use `--resume <session-id>` for conversation continuity. The session ID comes from the stream-json init event.

4. **Tool deny names differ.** OpenClaw uses names like `browser`, `sessions_spawn`, `cron`. Claude Code uses `WebFetch`, `WebSearch`, `Bash(pattern)`. Many OpenClaw tools have no Claude Code equivalent.

5. **Cron expressions don't map 1:1 to launchd.** `*/N` patterns become `StartInterval` (seconds). Ranges and lists must be expanded into multiple `StartCalendarInterval` dicts. launchd has no native step-pattern support for non-minute fields.

6. **The working directory IS the agent.** When `claude -p` runs with `cwd` set to an agent's directory, it auto-discovers that agent's CLAUDE.md, rules, and MCP config. There's no agent registry or instantiation — the filesystem defines identity.

7. **All cron jobs in an existing deployment may be disabled.** The migration tool always migrates all recurring cron jobs regardless of their `enabled` state. Jobs that have `enabled: false` get `<key>Disabled</key><true/>` in their launchd plist, preventing them from auto-starting. They can be selectively enabled later via `clawcode install`.

---

## What This Prompt Does NOT Cover

This prompt describes the standard migration framework. The following are deployment-specific extensions that may or may not apply:

- **Sentinel systems** — Zero-token check scripts that only invoke the LLM on trigger. If your deployment uses sentinels, you'll need a sentinel converter that adapts the check/dispatch scripts and generates launchd plists for the check schedules.

- **Dream/reflection consolidation** — Nightly or weekly memory consolidation pipelines (extract-session-dialogue, dream-preprocessor, session-health-report). These read from chat history, condense conversations, and produce memory entries. If your deployment uses these, you'll need to port the gate scripts and adapt the dialogue extractor to read from chat-history.jsonl.

- **Vector search / embeddings** — The memory MCP server supports full-text search out of the box. Semantic vector search requires an embedding provider (OpenAI, Ollama, etc.) and a compute step during sync. The schema supports it but the embedding pipeline is a separate concern.

- **Database setup automation** — The schema.sql exists but creating the database, installing pgvector, and running the schema is manual. A `db init` command would automate this.

- **Tests** — The build order above doesn't include tests. A vitest suite covering converters (pure functions), utilities (cron parser, markdown parser), and agent-runner argument building is recommended.
