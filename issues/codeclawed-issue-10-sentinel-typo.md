# Issue 10: Typo in Sentinel Converter Regex

## Status
`src/generate/converters/sentinel.ts` has `ocpenclaw` instead of `openclaw` in two regex patterns, causing them to never match.

## The Bug

### `adaptRunSentinelScript()` — line 29
```typescript
adapted = adapted.replace(
  /\bocpenclaw\s+agent\b/g,    // ← "ocpenclaw" — typo
  `node "${agentRunnerCliPath}" --skip-permissions`
);
```

Should be:
```typescript
  /\bopenclaw\s+agent\b/g,
```

### `adaptDispatchScript()` — line 61
```typescript
adapted = adapted.replace(
  /\bocpenclaw\s+agent\b/g,    // ← same typo
  `node "${agentRunnerCliPath}" --skip-permissions`
);
```

Should be:
```typescript
  /\bopenclaw\s+agent\b/g,
```

## Impact
These regexes are the fallback patterns for replacing bare `openclaw agent` calls (without the `${}` variable syntax). The `${OPENCLAW}` variable-style replacements on adjacent lines work correctly. If any sentinel scripts use the bare command `openclaw agent` (without the variable), those lines won't be adapted.

## Fix
Two single-character changes in `src/generate/converters/sentinel.ts`: lines 29 and 61, change `ocpenclaw` to `openclaw`.

## Files to modify
- `src/generate/converters/sentinel.ts` — lines 29 and 61
