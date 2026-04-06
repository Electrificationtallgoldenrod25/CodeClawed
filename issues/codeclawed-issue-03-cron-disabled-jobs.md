# Issue 3: Cron Jobs Generate as 0 (All Disabled)

## Status
`clawcode generate` produces 0 cron job scripts because all 64 jobs in the OpenClaw deployment have `enabled: false`.

## Root Cause
In `src/generate/index.ts` line 51:
```typescript
if (!job.enabled) continue; // Only migrate enabled jobs
```

This skips every job. The jobs are disabled in OpenClaw (likely because the system is being migrated), but they still need to be migrated so they can be re-enabled in the new system.

## What Needs to Happen

### Option A: Add `--include-disabled` flag (recommended)
Add a CLI flag to `clawcode generate`:
```
clawcode generate --include-disabled    # migrate all recurring jobs regardless of enabled state
```

In `src/generate/index.ts`, pass through the flag and change the filter:
```typescript
if (!job.enabled && !options.includeDisabled) continue;
```

Generated plists should still be created with `Disabled: true` in the plist XML so they don't auto-start. The user can then selectively enable them via `clawcode install`.

### Option B: Always migrate, mark as disabled
Remove the `enabled` filter entirely. Always generate the wrapper scripts and message files. Set `Disabled: true` in the launchd plist. This is simpler and arguably more correct — the migration tool should capture everything, the install step decides what to activate.

### Files to modify
- `src/cli.ts` — add `--include-disabled` option to the `generate` command
- `src/generate/index.ts` — pass through the option and adjust the filter on line 51

### Verification
After fix, `clawcode generate --include-disabled` should report generating cron jobs. Check:
- `output/scripts/jobs/` has `.sh` wrapper scripts
- `output/scripts/jobs/messages/` has `.md` message files
- `output/scripts/jobs/` has `.plist` files
- `output/config.json` cronJobs array is populated

### Scale
There are 64 total jobs. After `filterMigratableJobs` (removes one-shots and non-cron), roughly 20-25 recurring cron jobs should be generated (dreams, reflections, research sweeps, market briefs, etc.).
