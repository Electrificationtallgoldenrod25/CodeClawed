# Issue 6: Memory MCP Server Has No Vector Search

## Status
The pgvector `embedding` column exists in the schema but embeddings are never computed. The MCP server's `memory_search` tool uses full-text search only (`ts_rank` / `plainto_tsquery`). The sync daemon (`lib/memory/sync.ts`) inserts content but leaves the `embedding` column null.

## What Exists
- `lib/memory/schema.sql` — table with `embedding vector(1536)` column, IVFFlat index
- `lib/memory/mcp-server.ts` — MCP server with `memory_search`, `memory_store`, `memory_list` tools; search is text-only
- `lib/memory/sync.ts` — chokidar watcher that syncs `.md` files to the database without embeddings

## What Needs to Happen

### 1. Choose an embedding provider
Options:
- **Anthropic Voyager** — not yet available as a standalone embedding API
- **OpenAI `text-embedding-3-small`** — 1536 dimensions (matches schema), reliable, costs ~$0.02/1M tokens. Requires `OPENAI_API_KEY`.
- **Local model via Ollama** — e.g., `nomic-embed-text` (768 dims) or `mxbai-embed-large` (1024 dims). Free, no API key, but requires Ollama running. Schema dimension would need adjustment.
- **Voyage AI** — high quality embeddings, `voyage-3` is 1024 dims

Recommendation: Support OpenAI as default (most users will have a key), with Ollama as an optional local alternative. Make the provider configurable.

### 2. Add embedding function
Create `lib/memory/embed.ts`:
```typescript
interface EmbeddingProvider {
  embed(text: string): Promise<number[]>;
  dimensions: number;
}
```

Implement for OpenAI (HTTP call to `/v1/embeddings`) and optionally Ollama (`/api/embeddings`).

### 3. Update sync daemon (`lib/memory/sync.ts`)
In `syncFile()`, after reading content:
1. Call the embedding provider
2. Include the embedding vector in the INSERT/UPSERT

### 4. Update MCP server (`lib/memory/mcp-server.ts`)
Change `memory_search` to use hybrid search when embeddings are available:
```sql
SELECT *, 
  (0.7 * (1 - (embedding <=> $1::vector))) + 
  (0.3 * ts_rank(to_tsvector('english', content), plainto_tsquery('english', $2)))
  AS score
FROM memory_entries
WHERE agent_id = $3
ORDER BY score DESC
LIMIT $4
```

The weights (0.7 vector, 0.3 text) match what OpenClaw used in its hybrid search config.

Fall back to text-only search if the embedding column is null (provider not configured).

### 5. Update `memory_store` tool
When content is stored via the MCP tool (not just file sync), also compute and store the embedding.

### 6. Schema adjustment (if using non-1536 models)
If supporting Ollama models with different dimensions, the schema needs to be configurable or use a larger dimension with padding. Alternatively, just document that the dimension must match the provider.

### Files to modify
- `lib/memory/sync.ts` — add embedding on sync
- `lib/memory/mcp-server.ts` — hybrid search query

### Files to create
- `lib/memory/embed.ts` — embedding provider abstraction

### Configuration
Add to `output/config.json`:
```json
"memory": {
  "databaseUrl": "postgresql://localhost:5432/codeclawed",
  "embedding": {
    "provider": "openai",
    "model": "text-embedding-3-small",
    "dimensions": 1536
  }
}
```

Environment: `OPENAI_API_KEY` or `OLLAMA_HOST`.

### Notes
- The OpenClaw config shows hybrid search with `vectorWeight: 0.7`, `textWeight: 0.3`, MMR enabled with `lambda: 0.7`, and temporal decay with `halfLifeDays: 30`. These should be carried forward.
- Batch embedding (embed multiple chunks in one API call) would be more efficient for the full sync. The OpenAI API supports batch input.
