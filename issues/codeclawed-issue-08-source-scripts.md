# Issue 8: Source Script Templates Missing From Project

## Status
The `scripts/` directory in the project root is empty. `env.sh` and `wrapper.sh` are described in the plan as source files to ship with the project, but they only exist as generated output — `env.sh` is created inline in `src/generate/index.ts` (the `generateEnvSh()` function), and `wrapper.sh` was mentioned in the plan but never created.

## What This Means
- There are no reusable script templates in the source tree
- `env.sh` is hardcoded in a TypeScript function rather than being a template
- `wrapper.sh` (the generic agent invocation wrapper from the plan) doesn't exist anywhere
- When other users run `clawcode generate`, the env.sh is generated from code, which is fine — but having the templates as standalone files makes them easier to review and customize

## What Needs to Happen

### 1. Create `scripts/env.sh` as a Handlebars template
Move the env.sh generation out of the inline function in `generate/index.ts` and into a template file. The template would take `outputRoot`, `chatPort`, etc. as variables.

### 2. Create `scripts/wrapper.sh`
The plan described this as a generic wrapper that:
- Sources `env.sh`
- Parses `--agent`, `--model`, `--message-file`, `--timeout`, `--thinking`, `--deliver-to` flags
- Calls `agent-runner-cli.js`
- Optionally delivers output to the chat server

This is essentially what each generated cron wrapper does, but generalized. Useful for ad-hoc agent invocations from the terminal.

### 3. Update `src/generate/index.ts`
Instead of the inline `generateEnvSh()` function, read from the template and render it.

### Files to modify
- `src/generate/index.ts` — replace `generateEnvSh()` with template rendering

### Files to create
- `scripts/env.sh.hbs` (or just `scripts/env.sh` as a template with sed-style placeholders)
- `scripts/wrapper.sh`

### Notes
This is a low-priority cleanup issue. The system works without it. The main benefit is maintainability — having scripts as files rather than string literals in TypeScript is easier to review and edit.
