# OpenClaw — Model Configuration

## Two-provider strategy

Configure two providers for maximum coverage:

| Provider | Purpose | Models |
|----------|---------|--------|
| **Anthropic** (direct) | Primary agent models | Claude Sonnet, Opus, Haiku |
| **OpenRouter** | All other models + fallbacks | Gemini, DeepSeek, GPT, Llama, and hundreds more |

---

## Step 1: Set API keys in Hostinger Environment

In Hostinger panel → VPS → **Environment** section, add:

```
ANTHROPIC_API_KEY=sk-ant-YOUR_KEY
OPENROUTER_API_KEY=sk-or-YOUR_KEY
```

Click **Deploy** after adding. The container will see the new variables.

Never paste API keys directly to the agent in chat — chat messages are logged and can end up in memory.

Get keys:
- Anthropic: https://console.anthropic.com/settings/keys
- OpenRouter: https://openrouter.ai/keys (deposit $5 to start)

---

## Step 2: Clear default models

A new OpenClaw instance includes default models from various providers — but without API keys for those providers, they won't work. Clean up before adding your own:

Send to agent:
```
Show me the current model list in the catalog (agents.defaults.models).
Remove all models that are not from Anthropic — I'll only keep what I have API keys for.
```

---

## Step 3: Add Anthropic models

Send to agent:
```
Configure Anthropic models as the primary models:

1. Add to model catalog (agents.defaults.models):
   - anthropic/claude-sonnet-4-5 (alias: "Sonnet")
   - anthropic/claude-opus-4-6 (alias: "Opus")
   - anthropic/claude-haiku-4-5-20251001 (alias: "Haiku")

2. Set primary: anthropic/claude-sonnet-4-5

Save the configuration.
```

---

## Step 4: Add OpenRouter models

Send to agent:
```
Add OpenRouter models to the catalog (agents.defaults.models):

1. openrouter/google/gemini-2.5-pro (alias: "Gemini")
2. openrouter/deepseek/deepseek-chat-v3-0324 (alias: "DeepSeek")
3. openrouter/openai/gpt-4.1 (alias: "GPT")

Save the configuration.
```

Full model list: https://openrouter.ai/models

---

## Step 5: Set cross-provider fallbacks

Fallbacks must be from a **different provider** than the primary. If Anthropic goes down, a fallback to another Claude model won't help.

Send to agent:
```
Set cross-provider fallbacks for the primary model:
- fallback 1: openrouter/google/gemini-2.5-pro
- fallback 2: openrouter/deepseek/deepseek-chat-v3-0324

Save the configuration.
```

If Sonnet fails → auto-switch to Gemini → if that fails → DeepSeek. Returns to primary when available.

---

## Switching models in chat

Use the `/model` command at any time:

```
/model              → list available models
/model Sonnet       → switch to Claude Sonnet 4.5
/model Opus         → switch to Claude Opus 4.6 (most capable, higher cost)
/model Haiku        → switch to Haiku (fastest, cheapest)
/model Gemini       → switch to Gemini 2.5 Pro
/model DeepSeek     → switch to DeepSeek
```

---

## Adding a new model later

Send to agent:
```
Add model openrouter/meta-llama/llama-4-maverick with alias "Llama" to the model catalog.
```

---

## Model selection guide

| Task | Recommended model | Reason |
|------|------------------|--------|
| Daily Q&A, drafting, analysis | Sonnet | Best balance of quality and cost |
| Onboarding, complex reasoning, long documents | Opus | Highest capability, higher cost — use selectively |
| Simple tasks, fast responses, high volume | Haiku | Cheapest, fastest |
| Comparison / second opinion | Gemini or DeepSeek | Different model family, useful for cross-checking |

---

## Cost notes

- **Anthropic models** — billed directly by Anthropic
- **OpenRouter models** — billed through OpenRouter; each model has its own rate (check openrouter.ai/models)
- Use `/usage tokens` or `/usage cost` in chat to see token spend per session
