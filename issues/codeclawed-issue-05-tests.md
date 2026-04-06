# Issue 5: No Tests

## Status
vitest is a dev dependency but zero test files exist. The `test/` directory doesn't exist.

## What Should Be Tested

### Priority 1: Converters (pure functions, easy to test)

**`src/generate/converters/identity.ts`**
- Merges SOUL.md + USER.md + IDENTITY.md into CLAUDE.md
- Test: output is under 200 lines
- Test: core sections (identity, soul, user context) are present
- Test: overflow goes to `.claude/rules/identity-overflow.md`
- Test: handles missing IDENTITY.md (agent with no identity file)

**`src/generate/converters/rules.ts`**
- Parses AGENTS.md sections into separate rule files
- Test: "Session Startup" → `workflow.md`
- Test: "Red Lines" → `safety.md`
- Test: section with no matching pattern → `domain.md`
- Test: session startup instructions rewritten (removes "Read SOUL.md")

**`src/generate/converters/tools.ts`**
- Maps OpenClaw tool deny lists to Claude Code
- Test: `browser` → `['WebFetch', 'WebSearch']`
- Test: `subagents` → `['Bash(claude*)']`
- Test: unmappable tool logged as warning

**`src/generate/converters/cron.ts`**
- Converts cron jobs to wrapper scripts + plists
- Test: wrapper script sources env.sh
- Test: wrapper script calls agent-runner-cli with correct flags
- Test: plist XML is valid
- Test: delivery section generated for discord-mode jobs

**`src/generate/converters/sentinel.ts`**
- Adapts sentinel scripts
- Test: `openclaw agent --message` replaced with agent-runner call
- Test: `$OC_HOME` replaced with `$CODECLAWED_HOME`
- Test: registry paths updated

### Priority 2: Utilities

**`src/util/cron-parser.ts`**
- Cron expression to launchd conversion
- Test: `0 3 * * *` (daily 3am) → `StartCalendarInterval {Hour: 3, Minute: 0}`
- Test: `*/5 * * * *` → `StartInterval 300`
- Test: `0 12 * * 0` (Sunday noon) → correct Weekday value
- Test: plist XML generation is well-formed

**`src/util/markdown.ts`** (if it exists)
- Section extraction, line counting

### Priority 3: Agent Runner

**`lib/agent-runner.ts`**
- Test `buildArgs()` produces correct CLI arguments
- Test: `--resume` flag included when `resumeSessionId` set
- Test: `--verbose` added when `outputFormat` is `stream-json`
- Test: `--dangerously-skip-permissions` when `skipPermissions` is true
- Note: actual claude invocation can't be unit-tested without mocking

### Priority 4: Analysis

**`src/analyze/cron.ts`**
- `filterMigratableJobs()` — test filtering logic
- `categorizeJobs()` — test categorization patterns

### Test fixtures
Create `test/fixtures/` with:
- Sample SOUL.md, USER.md, IDENTITY.md, AGENTS.md
- Sample jobs.json with a mix of enabled/disabled, cron/at jobs
- Sample registry.json
- Sample openclaw.json (minimal, 2-3 agents)

### Setup
```bash
mkdir -p test/fixtures
# vitest config is already picked up from package.json "test": "vitest"
```

### Files to create
- `test/converters/identity.test.ts`
- `test/converters/rules.test.ts`
- `test/converters/tools.test.ts`
- `test/converters/cron.test.ts`
- `test/converters/sentinel.test.ts`
- `test/util/cron-parser.test.ts`
- `test/analyze/cron.test.ts`
- `test/agent-runner.test.ts` (buildArgs only)
- `test/fixtures/*.md`, `test/fixtures/*.json`
