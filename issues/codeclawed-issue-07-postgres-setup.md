# Issue 7: No Postgres Setup Command

## Status
The schema file exists (`lib/memory/schema.sql`) with a comment saying `createdb codeclawed && psql codeclawed < schema.sql`, but there's no CLI command, setup script, or documentation to actually do this.

## What Exists
- `lib/memory/schema.sql` — CREATE EXTENSION vector, CREATE TABLE memory_entries, indexes
- `lib/memory/sync.ts` — assumes database exists, connects to `DATABASE_URL` or `postgresql://localhost:5432/codeclawed`
- `lib/memory/mcp-server.ts` — same assumption
- `output/config.json` — has `memory.databaseUrl` field

## What Needs to Happen

### Option A: Add a `clawcode db` subcommand (recommended)
```bash
clawcode db init                    # create database + run schema
clawcode db status                  # check connection + table exists
clawcode db seed --output ./output  # load generated seed.sql into database
clawcode db reset                   # drop and recreate (with confirmation)
```

### Implementation
In `src/cli.ts`, add a `db` command group:

**`db init`**:
1. Check if `psql` is available
2. Run `createdb codeclawed` (ignore error if exists)
3. Run `psql codeclawed < lib/memory/schema.sql`
4. Report success/failure

**`db status`**:
1. Connect to the database using pg Pool
2. Check if `memory_entries` table exists
3. Check if pgvector extension is installed
4. Report row count, connection info

**`db seed`**:
1. Read `output/lib/memory/seed.sql` (generated during `clawcode generate`)
2. Execute against the database
3. Report entries inserted

### Alternative: Auto-check on first use
The sync daemon and MCP server could check for the table on startup and run the schema if missing. Simpler UX but less transparent.

### Prerequisites for the user
- PostgreSQL installed and running (`brew install postgresql@16 && brew services start postgresql@16`)
- pgvector extension installed (`brew install pgvector` or from source)

### Files to modify
- `src/cli.ts` — add `db` subcommand group

### Files to read
- `lib/memory/schema.sql` — the schema to apply
- `lib/memory/sync.ts` — how the database is used
- `output/lib/memory/seed.sql` — generated seed data (agent memory file contents)

### Notes
- The `seed.sql` generated during `clawcode generate` contains INSERT statements for every agent's memory files. It does NOT include embeddings (those would need to be computed — see Issue 6).
- Consider adding a `DATABASE_URL` option to the CLI or reading it from `output/config.json`.
