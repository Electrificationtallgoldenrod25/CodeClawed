# Issue 1: `clawcode install` Command Is a Stub

## Status
The CLI `install` subcommand prints "not yet implemented" and exits.

## What Exists
- `LaunchdScheduler` class at `lib/scheduler/launchd.ts` — fully implemented with `install`, `uninstall`, `list`, `status`, `enable`, `disable` methods
- `Scheduler` interface at `lib/scheduler/types.ts`
- Cron converter at `src/generate/converters/cron.ts` already generates `.plist` files alongside wrapper scripts
- Generated plists land in `output/scripts/jobs/<slug>.plist` (though currently 0 are generated — see Issue 3)

## What Needs to Happen

### Wire the CLI command (`src/cli.ts`, lines 103-114)
Replace the stub with:

1. Read `output/config.json` to get the list of cron jobs
2. Optionally filter by `--jobs` categories (dreams, reflections, sentinels, market, content, research)
3. For each job:
   - Use `LaunchdScheduler.install()` which writes the plist to `~/Library/LaunchAgents/` and calls `launchctl load`
   - OR copy the already-generated `.plist` files from `output/scripts/jobs/` to `~/Library/LaunchAgents/` and load them
4. If `--dry-run`, list what would be installed without writing

The second approach (copy generated plists) is simpler since the plists already exist from the generate step. The `LaunchdScheduler` is more useful for runtime management (enable/disable individual jobs after install).

### Suggested implementation
```
clawcode install --output ./output                    # install all
clawcode install --output ./output --jobs dreams      # install only dream jobs
clawcode install --output ./output --dry-run          # preview
clawcode install --output ./output --uninstall        # remove all codeclawed plists
```

### Files to modify
- `src/cli.ts` — replace stub action (lines 110-114)

### Files to read
- `lib/scheduler/launchd.ts` — the scheduler that does the actual work
- `src/constants.ts` — `LAUNCHD_LABEL_PREFIX` used for plist naming

### Dependencies
- Issue 3 (cron jobs generating as 0) must be resolved first, otherwise there's nothing to install
