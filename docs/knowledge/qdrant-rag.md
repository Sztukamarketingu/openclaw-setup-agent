# Qdrant — Semantic Memory and RAG

## What Qdrant does

Qdrant is a vector database for semantic search. You embed documents (convert text to vectors) and store them in Qdrant. When the agent needs context — it searches Qdrant for the most relevant chunks and includes them in its response.

**Use cases:**
- Agent answers questions using your product catalog, documentation, or FAQ
- Agent finds relevant past emails or offers when drafting new ones
- Agent retrieves pricing or policy information without needing it in every prompt
- "Find similar offers to this one" — semantic not keyword search

**When Qdrant is NOT needed:**
- The agent's SOUL.md and workspace cover the knowledge base (good for static, small knowledge)
- Knowledge fits in the context window and doesn't change often
- You're in POC phase — start without Qdrant, add it when the knowledge base grows

---

## Architecture

```
User question
    ↓
Agent receives message
    ↓
Agent searches Qdrant: "find relevant chunks for this question"
    ↓
Qdrant returns top-N most semantically relevant chunks
    ↓
Agent includes chunks as context + generates response

New document added:
n8n workflow → chunk text → embed (OpenAI/Anthropic) → upsert to Qdrant
```

---

## Install Qdrant on the same VPS

For POC: run Qdrant as a Docker container on the same Hostinger VPS.

```bash
docker run -d \
  --name qdrant \
  --restart always \
  -p 127.0.0.1:6333:6333 \
  -v /data/qdrant:/qdrant/storage \
  qdrant/qdrant
```

**Important:** bind to `127.0.0.1:6333` — do not expose Qdrant publicly. Access is only from the same VPS.

Verify it's running:
```bash
curl http://127.0.0.1:6333/healthz
```

Expected: `{"title":"qdrant - vector search engine","version":"..."}`

---

## Add Qdrant to Hostinger Environment

In Hostinger panel → VPS → **Environment**, add:

```
QDRANT_URL=http://127.0.0.1:6333
QDRANT_API_KEY=
```

Leave `QDRANT_API_KEY` empty for local Qdrant (no auth needed). Set a value if using Qdrant Cloud.

Click **Deploy**.

---

## Create a collection

Before using Qdrant, create a collection. The vector size depends on the embedding model:

| Embedding model | Vector size |
|----------------|-------------|
| OpenAI text-embedding-3-small | 1536 |
| OpenAI text-embedding-3-large | 3072 |
| Anthropic (not available directly) | — |
| Sentence transformers (local) | 384–768 |

Create collection via API:

```bash
curl -X PUT http://127.0.0.1:6333/collections/openclaw \
  -H 'Content-Type: application/json' \
  -d '{
    "vectors": {
      "size": 1536,
      "distance": "Cosine"
    }
  }'
```

---

## Configure OpenClaw to use Qdrant

Use `docs/templates/config-with-qdrant.json5` as the base. Send to agent:

```
Configure OpenClaw to use Qdrant for semantic search.
QDRANT_URL and QDRANT_API_KEY are set in the environment variables.
Collection name: openclaw
Use the qdrant_search tool and set up vectorStore in config.json.
```

---

## Indexing documents into Qdrant

OpenClaw reads from Qdrant. Indexing (writing to Qdrant) is typically done by n8n.

### n8n indexing workflow pattern

```
Trigger (new file, email, form, schedule)
    ↓
Read document content
    ↓
Chunk text (split into 500-1000 token segments with overlap)
    ↓
Embed each chunk (OpenAI Embeddings node or HTTP Request to embeddings API)
    ↓
Upsert to Qdrant (HTTP Request node: PUT /collections/openclaw/points)
```

### Qdrant upsert request

```json
PUT http://127.0.0.1:6333/collections/openclaw/points
{
  "points": [
    {
      "id": "unique-id-for-this-chunk",
      "vector": [0.1, 0.2, ...],
      "payload": {
        "text": "original chunk text",
        "source": "filename or URL",
        "date": "2026-01-15",
        "type": "offer | faq | policy | email"
      }
    }
  ]
}
```

---

## Querying Qdrant from OpenClaw

When the agent has the `qdrant_search` tool enabled, it searches automatically. You can also trigger a search directly:

```
Search knowledge base: find our pricing policy for enterprise clients
```

Manual Qdrant search (for testing):

```bash
# First embed the query (example with OpenAI)
# Then:
curl -X POST http://127.0.0.1:6333/collections/openclaw/points/search \
  -H 'Content-Type: application/json' \
  -d '{
    "vector": [0.1, 0.2, ...],
    "limit": 5,
    "with_payload": true
  }'
```

---

## Qdrant Cloud (production alternative)

For production or when you want managed Qdrant:

1. Create account: https://cloud.qdrant.io
2. Create a cluster — free tier available
3. Get the cluster URL and API key
4. Update Hostinger Environment:

```
QDRANT_URL=https://your-cluster.qdrant.io
QDRANT_API_KEY=your-qdrant-api-key
```

No other changes needed — the config references env vars.

---

## Memory split: SOUL.md vs Qdrant

| Memory type | Where | Use for |
|-------------|-------|---------|
| Short-term, operational | OpenClaw session / MEMORY.md | Current conversation context, user preferences |
| Static knowledge | SOUL.md + workspace files | Small, stable knowledge bases (< 50 pages) |
| Large, searchable, dynamic | Qdrant | Product catalogs, email archives, large documentation, frequently updated content |

Do not duplicate: one source of truth per knowledge type.

---

## Troubleshooting

### Agent doesn't use Qdrant context

1. Verify `vectorStore` config is correct and Qdrant is running
2. Check `qdrant_search` is in `tools.allow`
3. Verify the collection exists and has documents: `curl http://127.0.0.1:6333/collections/openclaw`
4. Test a direct search to confirm documents are indexed

### Qdrant container crashes on small VPS

Qdrant needs RAM proportional to the number of vectors. On a 2GB VPS:
- Limit collection size or use disk-based storage
- Set memory limit: add `--memory 512m` to the Docker run command
- Consider Qdrant Cloud if the VPS is too small

### Slow search responses

- Reduce `limit` in search queries (default 5 is fine, don't use 50)
- Use smaller embedding model (text-embedding-3-small vs large)
- Add an index on payload fields you filter by
