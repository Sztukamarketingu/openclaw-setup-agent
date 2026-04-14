# OpenClaw Configuration Reference

Official docs: https://docs.openclaw.ai
Model providers: https://docs.openclaw.ai/concepts/model-providers.md
Multi-agent: https://docs.openclaw.ai/concepts/multi-agent.md
Webhooks: https://docs.openclaw.ai/automation/webhook.md
VPS setup: https://docs.openclaw.ai/vps.md

---

## Config file location

`~/.openclaw/config.json5`

OpenClaw uses JSON5 format (supports comments, trailing commas, unquoted keys).

---

## Minimal config structure

```json5
{
  // Model provider
  models: {
    providers: {
      // Built-in: anthropic, openai, openrouter
      // Custom: see "Custom provider" section below
    },
  },

  // Agent defaults
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-5",
      },
    },
  },

  // Channels
  channels: {
    telegram: {
      enabled: true,
      token: "${TELEGRAM_BOT_TOKEN}",
    },
  },

  // Gateway security
  gateway: {
    auth: {
      token: "${OPENCLAW_GATEWAY_TOKEN}",
    },
  },

  // Webhooks (n8n integration)
  hooks: {
    enabled: false,
    token: "${OPENCLAW_HOOKS_TOKEN}",
  },
}
```

---

## Model providers

### Anthropic (direct)

```json5
agents: {
  defaults: {
    model: {
      primary: "anthropic/claude-sonnet-4-5",
    },
  },
},
```

Requires `ANTHROPIC_API_KEY` set as environment variable.

### OpenAI (direct)

```json5
agents: {
  defaults: {
    model: {
      primary: "openai/gpt-4o",
    },
  },
},
```

Requires `OPENAI_API_KEY` set as environment variable.

### OpenRouter

```json5
agents: {
  defaults: {
    model: {
      primary: "openrouter/anthropic/claude-3.5-sonnet",
    },
  },
},
```

Requires `OPENROUTER_API_KEY`.

### Custom provider (Nexos, LM Studio, vLLM, LiteLLM)

```json5
models: {
  providers: {
    nexos: {
      baseUrl: "${NEXOS_BASE_URL}",
      apiKey: "${NEXOS_API_KEY}",
      api: "openai-completions",
      models: [
        { id: "claude-sonnet-4-5", name: "Claude Sonnet via Nexos" },
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

See `docs/knowledge/nexos-integration.md` for details and troubleshooting.

---

## Channels

### Telegram

```json5
channels: {
  telegram: {
    enabled: true,
    token: "${TELEGRAM_BOT_TOKEN}",
    // Restrict who can message the bot:
    dmPolicy: "allowlist",           // "open" | "allowlist" | "closed"
    allowlist: ["@yourusername"],    // Telegram usernames (with @)
  },
},
```

### Multiple channels

```json5
channels: {
  telegram: {
    enabled: true,
    token: "${TELEGRAM_BOT_TOKEN}",
    agentId: "main",                 // Route to specific agent
  },
  discord: {
    enabled: false,                  // Enable when ready
    token: "${DISCORD_BOT_TOKEN}",
  },
},
```

---

## Multi-agent routing

For multiple agents with different roles:

```json5
agents: {
  list: [
    {
      id: "main",
      name: "Main Assistant",
      model: { primary: "anthropic/claude-sonnet-4-5" },
    },
    {
      id: "offers",
      name: "Offers Specialist",
      model: { primary: "anthropic/claude-opus-4-6" },
    },
  ],
},
channels: {
  telegram: {
    enabled: true,
    token: "${TELEGRAM_BOT_TOKEN}",
    bindings: [
      { agentId: "main" },             // Default agent
    ],
  },
},
```

---

## Hooks (webhook) configuration

Required if n8n or other systems will trigger the agent:

```json5
hooks: {
  enabled: true,
  token: "${OPENCLAW_HOOKS_TOKEN}",   // NOT the same as gateway token
  path: "/hooks",
  defaultSessionKey: "hook:n8n:ingress",
  allowRequestSessionKey: false,
  allowedSessionKeyPrefixes: ["hook:"],
  allowedAgentIds: ["main"],          // Which agents can receive hooks
},
```

Endpoint: `POST http://127.0.0.1:18789/hooks/agent`
Header: `Authorization: Bearer {OPENCLAW_HOOKS_TOKEN}`

---

## Gateway configuration

```json5
gateway: {
  // Listen address — keep on loopback unless using reverse proxy with auth
  listen: "127.0.0.1:18789",         // Default
  auth: {
    token: "${OPENCLAW_GATEWAY_TOKEN}",
  },
},
```

For remote UI access: use SSH tunnel or Tailscale. Do not expose port 18789 publicly.

---

## VPS performance tuning

Add to `/etc/openclaw.env` or systemd unit for better performance on small VPS:

```bash
NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
OPENCLAW_NO_RESPAWN=1
```

Create cache directory: `mkdir -p /var/tmp/openclaw-compile-cache`

Systemd unit override (`systemctl edit openclaw`):

```ini
[Service]
Environment=OPENCLAW_NO_RESPAWN=1
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Restart=always
RestartSec=2
TimeoutStartSec=90
```

---

## SOUL.md — agent identity file

Location: `~/.openclaw/workspace/SOUL.md`

This markdown file defines the agent's personality, role, and operating rules. OpenClaw reads it at agent startup. It is the primary way to customize how the agent behaves without modifying config.json5.

See `docs/templates/soul-template.md` for the standard structure.

---

## Useful CLI commands

```bash
# Check Gateway status
openclaw gateway status

# Restart Gateway after config change
openclaw gateway restart

# Check all channel statuses
openclaw channels status --probe

# View recent logs
openclaw logs --tail 50

# List agents
openclaw agents list

# Add a new agent
openclaw agents add <id>
```
