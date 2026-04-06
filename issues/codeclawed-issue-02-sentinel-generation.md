# Issue 2: Sentinel Generation Not Wired Into Pipeline

## Status
The sentinel converter functions exist but are never called during `clawcode generate`.

## What Exists
- `src/generate/converters/sentinel.ts` — three functions:
  - `adaptRunSentinelScript()` — rewrites `run_sentinel.sh` to use agent-runner
  - `adaptDispatchScript()` — rewrites dispatch scripts (youtube, freshrss)
  - `adaptRegistry()` — updates registry.json paths
- Analysis phase (`src/analyze/sentinel.ts`) successfully parses the sentinel registry
- OpenClaw sentinel files at `~/.openclaw/workspace/skills/sentinel/`:
  - `SKILL.md`, `registry.json`, `run_sentinel.sh`
  - `check_youtube_feeds.sh`, `check_freshrss_feeds.sh`
  - `dispatch_youtube_feeds.sh`, `dispatch_freshrss_feeds.sh`
  - `state/` directory with state files
- `AnalysisResult.sentinels` contains the parsed registry with 2 sentinels: `youtube-feeds` (*/5 min) and `freshrss-feeds` (*/15 min)

## What Needs to Happen

### 1. Add sentinel generation to `src/generate/index.ts`
After the cron job generation step (around line 70), add:

- Read the original sentinel scripts from the OpenClaw workspace
- Call `adaptRunSentinelScript()` on `run_sentinel.sh`
- Call `adaptDispatchScript()` on each dispatch script
- Call `adaptRegistry()` to update paths
- Copy check scripts unchanged (they're already platform-agnostic)
- Write all adapted files to `output/scripts/sentinel/`
- Copy state directory structure

### 2. Generate launchd plists for sentinels
Each sentinel has a cron schedule (e.g., `*/5 * * * *`). Generate a plist that runs the adapted `run_sentinel.sh` at the sentinel's interval.

### 3. Add sentinel scripts to the GenerationResult
The `sentinelScripts` field in `GenerationResult` is currently returned as `new Map()`. Populate it.

### Files to modify
- `src/generate/index.ts` — add sentinel generation step
- Potentially `src/generate/converters/sentinel.ts` — fix the typo (see Issue 10) and add a higher-level orchestration function

### Files to read
- `~/.openclaw/workspace/skills/sentinel/` — all source scripts
- `src/generate/converters/sentinel.ts` — the existing converter functions
- `src/analyze/sentinel.ts` — how the registry is parsed

### Notes
- The sentinel pattern is zero-token: check scripts run on a cron schedule, and only if the check fails (exit code 1) does the LLM get invoked. This is important to preserve — the adapted scripts must maintain this gate.
- The check scripts should be copied verbatim. Only `run_sentinel.sh` and the dispatch scripts need adaptation.
