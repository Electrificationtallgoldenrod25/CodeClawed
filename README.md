# CodeClawed

A migration tool for moving [OpenClaw](https://github.com/openclaw) multi-agent deployments to [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## Status: Work in Progress

This project is **not complete**. The core migration pipeline (analyze, generate, verify, chat) works end-to-end, but several components are stubs or partially implemented. See [Known Gaps](#known-gaps) below.

We are actively developing this tool against a live 16-agent OpenClaw deployment. The prompt and architecture are evolving as we resolve remaining issues.

## What This Is

CodeClawed is two things:

1. **A CLI tool** that reads an OpenClaw installation (`~/.openclaw/`) and produces a fully functional Claude Code project — one directory per agent, with identity files, rules, memory, cron job wrappers, and a web chat frontend.

2. **A build prompt** (`codeclawed-migration-prompt.md`) that contains everything needed to reproduce the tool from scratch. Hand it to Claude Code and it will build the migration tool for your deployment.

## How to Use the Prompt

The prompt is designed to be given to Claude Code (or any capable LLM) as a complete specification. It covers:

- OpenClaw's file formats and what each file does
- How each file maps to Claude Code's conventions
- The agent-runner abstraction (the single function that wraps `claude -p`)
- Cron-to-launchd schedule conversion
- The chat server with session resumption
- Memory system (files + optional pgvector)
- Build order and dependencies

### For your own deployment

1. Read `codeclawed-migration-prompt.md` to understand the architecture
2. Point Claude Code at the prompt and your `~/.openclaw/` directory
3. Adjust for your deployment's specifics (number of agents, which cron categories you use, tool restrictions)
4. Run `analyze` → `generate` → `verify` → `chat`

### To contribute

The prompt is the source of truth for the architecture. If you're fixing a bug or adding a feature, update the prompt document alongside the code so they stay in sync.

## Quick Start (if the tool is already built)

```bash
# Install dependencies
npm install

# Analyze your OpenClaw deployment
npx tsx src/cli.ts analyze --oc-home ~/.openclaw

# Generate the Claude Code project
npx tsx src/cli.ts generate --oc-home ~/.openclaw --output ./output

# Verify the output
npx tsx src/cli.ts verify --output ./output

# Start the chat frontend
npx tsx src/cli.ts chat --output ./output
# Open http://localhost:3456
```

## How It Works

### The core idea

OpenClaw is a daemon. Claude Code is a stateless CLI. CodeClawed bridges the gap:

- **Identity**: OpenClaw's `SOUL.md` + `USER.md` + `IDENTITY.md` merge into a single `CLAUDE.md` (max 200 lines). `AGENTS.md` shards into `.claude/rules/*.md` files by section topic.
- **Agent invocation**: A single `agent-runner.ts` function wraps `claude -p`. Everything in the system — cron jobs, the chat server, scripts — calls through this one function. To change how agents are invoked, you change one file.
- **Conversation continuity**: Claude Code supports `--resume <session-id>`. The chat server captures session IDs and resumes conversations automatically, giving agents full context across messages.
- **Scheduling**: Cron expressions convert to macOS launchd plists (abstracted behind a `Scheduler` interface for future Linux/systemd support).
- **Memory**: Agent memory files are human-readable markdown, copied as-is. An optional pgvector MCP server adds semantic search.

### What the folder IS the agent means

```
output/agents/forge/
├── CLAUDE.md              ← Claude Code auto-loads this as identity
├── .claude/
│   ├── rules/*.md         ← Auto-loaded behavioral rules
│   └── mcp_config.json    ← Tool extensions (memory search)
├── memory/                ← Agent's private memory files
└── shared-memory → ...    ← Cross-agent shared knowledge
```

When `claude -p` runs with its working directory set to `output/agents/forge/`, it becomes Forge. There's no agent registry, no instantiation code, no daemon. The directory defines the agent.

## Known Gaps

All previously identified gaps have been covered. THIS DOES NOT MEAN THIS IS READY FOR IMPLMENTATION: Additional gaps are being idenfied and handled.

## Requirements

- Node.js >= 20
- Claude Code CLI (`claude`) installed and on PATH
- macOS (for launchd scheduler; Linux support is architected but not implemented)
- PostgreSQL + pgvector (optional, only needed for database-backed memory search)

## License

TBD
