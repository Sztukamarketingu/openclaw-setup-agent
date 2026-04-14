# Nexos Gateway Integration

## What Nexos provides

Nexos AI Gateway (https://nexos.ai/ai-gateway/) is a single API endpoint that routes to multiple LLM providers. Key features:
- Cost tracking per team, project, or model
- Budget limits and alerts
- Request logging and observability
- Model routing and fallbacks
- OpenAI-compatible API (works as a drop-in replacement)

---

## OpenClaw integration pattern

OpenClaw does not natively list Nexos as a provider — integration uses the `openai-completions` custom provider pattern.

### Required information from Nexos

Before configuring, collect from your Nexos console:
1. **Base URL** — typically `https://api.nexos.ai/v1` or similar (confirm in Nexos dashboard)
2. **API key** — your Nexos account API key
3. **Model IDs** — exact identifiers as Nexos exposes them (e.g. `claude-sonnet-4-5`, `gpt-4o`)

**Important:** Model IDs in Nexos may differ from provider-native IDs. Get the exact list from Nexos before configuring.

### Test Nexos endpoint before configuring OpenClaw

```bash
curl -s -X POST "${NEXOS_BASE_URL}/chat/completions" \
  -H "Authorization: Bearer ${NEXOS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "YOUR_NEXOS_MODEL_ID",
    "messages": [{"role": "user", "content": "Say hello"}]
  }'
```

If this returns a valid completion response, the same `baseUrl` and `apiKey` are ready for OpenClaw.

---

## config.json5 — Nexos provider block

```json5
models: {
  providers: {
    nexos: {
      baseUrl: "${NEXOS_BASE_URL}",
      apiKey: "${NEXOS_API_KEY}",
      api: "openai-completions",
      // List models exactly as Nexos exposes them
      models: [
        { id: "claude-sonnet-4-5", name: "Claude Sonnet via Nexos" },
        { id: "gpt-4o", name: "GPT-4o via Nexos" },
      ],
    },
  },
},
agents: {
  defaults: {
    model: {
      primary: "nexos/claude-sonnet-4-5",
    },
  },
},
```

The format is `"nexos/<model-id>"` where `model-id` matches exactly what is in `models[].id`.

---

## Cost and budget management

Budget control lives in **Nexos**, not in OpenClaw. Choose one source of truth for cost reporting:
- Use Nexos dashboard for budget tracking and alerts
- Do not duplicate cost tracking in OpenClaw's usage tracking for the same calls

If you want both: use Nexos for budget enforcement (hard limits), use OpenClaw usage tracking for per-agent observability.

---

## Troubleshooting

### 400 Bad Request from Nexos

Possible causes:
- Model ID does not match what Nexos exposes — check `/v1/models` if Nexos supports it
- Base URL has wrong format (trailing slash, missing `/v1`)
- Request contains fields Nexos does not support

Fix: test with raw `curl` first (see above). Compare working curl body with what OpenClaw sends.

### "Developer role not supported" or similar

Some providers behind Nexos may not support all message roles. If OpenClaw sends a `developer` role message and Nexos does not support it:

```json5
models: {
  providers: {
    nexos: {
      // ... other fields ...
      compat: {
        supportsDeveloperRole: false,
      },
    },
  },
},
```

Check OpenClaw model-providers docs for current `compat` options.

### Context window or token limit errors

Set explicit limits in the provider config:

```json5
models: {
  providers: {
    nexos: {
      // ... other fields ...
      models: [
        {
          id: "claude-sonnet-4-5",
          name: "Claude Sonnet via Nexos",
          contextWindow: 200000,
          maxTokens: 8096,
        },
      ],
    },
  },
},
```

### API key rotation

Nexos keys can be rotated in the Nexos console. After rotation:
1. Update `NEXOS_API_KEY` in `/etc/openclaw.env` on the VPS
2. Restart OpenClaw: `openclaw gateway restart`
3. No change to `config.json5` needed (it references the env var)

---

## Alternative: LiteLLM as proxy

If Nexos does not provide a fully OpenAI-compatible API, add LiteLLM as a local proxy between OpenClaw and Nexos:

```
OpenClaw → LiteLLM (localhost:4000) → Nexos
```

OpenClaw config would then point to `http://127.0.0.1:4000` as `baseUrl` with LiteLLM's key.

This adds complexity — only use it if direct Nexos integration has compatibility issues.
