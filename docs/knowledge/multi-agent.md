# Multi-Agent Setup

## When to use multiple agents

Start with one agent (single-agent POC). Add a second agent only when there's a clear reason:

| Situation | Solution |
|-----------|---------|
| Different audience needs very different tone or knowledge | Separate agents |
| One agent for customers, one for internal team | Separate agents |
| Complex tasks need a specialized "deep work" agent (Opus) while daily chat uses Sonnet | Separate agents with different models |
| n8n triggers a specialized "drafting" agent | Separate agent with restricted tools |
| Different channel should reach a different agent | Route channels to agent IDs |

**Do not** create multiple agents just because you have multiple topics. One well-configured agent with good SOUL.md handles most cases.

---

## Config: defining multiple agents

```json5
agents: {
  list: [
    {
      id: "main",
      name: "Main Assistant",
      model: {
        primary: "anthropic/claude-sonnet-4-5",
      },
      workspace: "/data/.openclaw/workspace",
    },
    {
      id: "drafts",
      name: "Document Drafter",
      model: {
        primary: "anthropic/claude-opus-4-6",   // Heavier model for quality drafts
      },
      workspace: "/data/.openclaw/workspace-drafts",
      tools: {
        allow: ["read_file", "write_file"],   // Restricted tool set
      },
    },
  ],
},
```

Each agent gets:
- Its own `id` (used for routing and hooks)
- Its own `workspace` directory (SOUL.md, memory, files)
- Optionally its own model and tool restrictions

---

## Routing channels to agents

Different Telegram bots → different agents:

```json5
channels: {
  telegram: {
    enabled: true,
    botToken: "${TELEGRAM_BOT_TOKEN}",
    dmPolicy: "allowlist",
    allowFrom: [123456789],
    agentId: "main",             // This channel → main agent
    groups: { "*": { enabled: false } },
  },
  // Second bot for internal team:
  telegram_internal: {
    enabled: true,
    botToken: "${TELEGRAM_INTERNAL_BOT_TOKEN}",
    dmPolicy: "allowlist",
    allowFrom: [123456789, 987654321],
    agentId: "drafts",           // This channel → drafts agent
    groups: { "*": { enabled: false } },
  },
},
```

---

## Routing n8n hooks to specific agents

```json5
hooks: {
  enabled: true,
  token: "${OPENCLAW_HOOKS_TOKEN}",
  allowedAgentIds: ["main", "drafts"],   // Both agents can receive hooks
},
```

In the n8n HTTP Request body, specify the target agent:

```json
{
  "message": "Draft this offer...",
  "agentId": "drafts",
  "wakeMode": "now",
  "deliver": false
}
```

---

## Separate SOUL.md per agent

Each agent should have its own SOUL.md in its workspace:

```
/data/.openclaw/workspace/SOUL.md           ← for "main" agent
/data/.openclaw/workspace-drafts/SOUL.md    ← for "drafts" agent
```

The drafts agent SOUL.md should be focused: "You are a document drafting specialist. Your only job is to produce high-quality written outputs given structured input. You do not answer general questions."

---

## CLI: managing agents

```bash
# List configured agents
docker exec $(docker ps -q) openclaw agents list

# Add a new agent
docker exec $(docker ps -q) openclaw agents add drafts

# Check agent status
docker exec $(docker ps -q) openclaw agents status
```

---

## Cost consideration

Each agent with a different model costs separately. Opus is ~5x more expensive than Sonnet. Only use Opus for agents where quality justifies the cost (document drafting, complex analysis) — not for general chat.

```
main agent (Sonnet)  → daily conversations, quick answers
drafts agent (Opus)  → n8n-triggered document drafts only
```

The `main` agent handles 95% of interactions at Sonnet cost. The `drafts` agent is called selectively by n8n for tasks that justify Opus.

---

## Security: least privilege per agent

Give each agent only the tools it needs:

```json5
// Main agent — can search, read files, use browser
{ id: "main", tools: { allow: ["read_file", "search_memory", "web_search"] } }

// Drafts agent — only reads input, writes output files
{ id: "drafts", tools: { allow: ["read_file", "write_file"] } }

// Support agent — no file access, no browser
{ id: "support", tools: { allow: ["search_memory"] } }
```

Never use `tools.allow: ["*"]` for n8n-triggered agents — they receive external input and should have the narrowest possible tool set.
