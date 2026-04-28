# OpenClaw Setup Agent

A Claude Code agent that guides you through configuring OpenClaw on a Hostinger VPS — from business discovery to live deployment and beyond.

**What this agent does:**
1. Asks about your business and communication needs (step by step)
2. Generates a custom configuration — agent personality, model, channels, tools
3. Deploys to your VPS via Hostinger API and SSH
4. Verifies the setup end-to-end
5. Guides post-setup: models, memory, web search, backups, cost optimization

**Assumes:** OpenClaw is already installed and network-secured on your Hostinger VPS.

---

## Quick start

```bash
git clone https://github.com/Sztukamarketingu/openclaw-setup-agent
cd openclaw-setup-agent
claude "Start OpenClaw setup"
```

The agent starts, walks you through creating `.env`, asks about your business, and guides you through the rest.

**Returning to a previous session:**
```bash
cd openclaw-setup-agent
claude "Continue setup"
```

The agent reads `output/SETUP_PROGRESS.md` and resumes exactly where you left off.

---

## Required credentials

Open `.env` and fill in the **REQUIRED** fields before starting:

| Variable | Where to get it |
|----------|-----------------|
| `HOSTINGER_API_KEY` | Hostinger panel → API tokens |
| `VPS_HOSTNAME` | Your VPS IP address |
| `VPS_USERNAME` | SSH username (`root` for most Hostinger VPS) |
| `VPS_SSH_KEY_PATH` | Path to your SSH private key |
| `ANTHROPIC_API_KEY` | console.anthropic.com (or use OpenAI / Nexos / OpenRouter) |
| `TELEGRAM_BOT_TOKEN` | Create bot via [@BotFather](https://t.me/botfather) on Telegram |
| `OPENCLAW_GATEWAY_TOKEN` | Generate: `openssl rand -hex 32` |
| `OPENCLAW_HOOKS_TOKEN` | Generate: `openssl rand -hex 32` (must differ from gateway token) |

---

## What the agent generates

All files go to `output/` (gitignored). You review and approve before deployment.

| File | Description |
|------|-------------|
| `output/SOUL.md` | Agent identity — name, personality, capabilities, restrictions |
| `output/config.json5` | OpenClaw configuration — model, channels, hooks, security |
| `output/env-vars.txt` | Environment variables to set on the VPS |

---

## Optional integrations

Add to `.env` when ready:

| Integration | Variables | Purpose |
|-------------|-----------|---------|
| n8n | `N8N_BASE_URL`, `N8N_API_KEY` | Trigger agent from workflows |
| Nexos gateway | `NEXOS_API_KEY`, `NEXOS_BASE_URL` | Cost control, multi-model routing |
| OpenRouter | `OPENROUTER_API_KEY` | Access to Gemini, DeepSeek, GPT, and more |
| Qdrant | `QDRANT_URL`, `QDRANT_API_KEY` | Semantic memory and RAG |
| ElevenLabs TTS | `ELEVENLABS_API_KEY` + `voiceId` in config | Voice replies on Telegram (auto-synth from inbound voice notes) |
| OpenAI TTS | reuses `OPENAI_API_KEY` | Cheaper alternative to ElevenLabs (alloy/echo/fable/onyx/nova/shimmer) |
| Airtable | `AIRTABLE_API_KEY`, `AIRTABLE_BASE_ID` | Native CRUD on a base — list, search, create, update records via MCP |

---

## Specialist modes

Type these at any time during the session:

| Trigger phrase | What happens |
|---------------|-------------|
| "audit my config" / "check security" | Security review using Principle of Least Privilege |
| "optimize costs" / "make it cheaper" | Cost audit: VPS sizing, model routing, token efficiency |
| "automate a web panel" | Browser automation setup for systems without API |

---

## Project structure

```
openclaw-setup-agent/
│
├── CLAUDE.md                              # Agent brain — all behavior defined here
├── README.md
├── .env.example                           # Credential template
├── .gitignore
│
├── output/                                # Generated files — gitignored, user-specific
│   └── .gitkeep
│
├── docs/
│   │
│   ├── knowledge/                         # OpenClaw knowledge base
│   │   ├── openclaw-config.md             # Config file structure and field reference
│   │   ├── hostinger-vps.md               # Hostinger API, SSH access, env vars
│   │   ├── telegram-setup.md              # Telegram bot setup, allowlist, Hostinger env
│   │   ├── telegram-voice-setup.md        # Voice replies (TTS) — ElevenLabs/OpenAI, gotchas
│   │   ├── airtable-setup.md              # Airtable CRUD via MCP (mcporter + airtable-mcp-server)
│   │   ├── discord-setup.md               # Discord bot setup and channel routing
│   │   ├── n8n-integration.md             # n8n → OpenClaw webhook contract
│   │   ├── nexos-integration.md           # Nexos AI gateway setup
│   │   ├── tailscale-setup.md             # Secure remote access via Tailscale
│   │   ├── security.md                    # Tokens, SSH, secrets, firewall rules
│   │   ├── browser-automation.md          # CDP browser — automate panels without API
│   │   ├── multi-agent.md                 # Multiple agents with routing and tool isolation
│   │   ├── qdrant-rag.md                  # Qdrant vector DB — semantic memory and RAG
│   │   ├── models-configuration.md        # Anthropic + OpenRouter, aliases, fallbacks
│   │   ├── web-search.md                  # Perplexity search via OpenRouter
│   │   ├── memory-management.md           # Memory Flush, Session Search, onboarding
│   │   ├── auto-updates.md                # Docker cron update + OS unattended-upgrades
│   │   ├── github-backup.md               # Automated config backup to GitHub
│   │   ├── commands-reference.md          # Full agent command cheat sheet
│   │   └── docker-troubleshooting.md      # Container issues, logs, common errors
│   │
│   ├── profiles/                          # Business-type configuration guides
│   │   ├── agency.md                      # Talent agency, booking, events
│   │   ├── consulting.md                  # Consulting, freelance, advisory
│   │   ├── ecommerce.md                   # Online store, order handling
│   │   ├── real-estate.md                 # Property sales, rentals, lead qualification
│   │   ├── support.md                     # Customer support, help desk
│   │   └── saas.md                        # SaaS product, onboarding, tech support
│   │
│   ├── templates/                         # Ready-to-use config templates
│   │   ├── soul-template.md               # SOUL.md template with all sections
│   │   ├── config-minimal.json5           # Minimal config (no channels, no hooks)
│   │   ├── config-with-telegram.json5     # Telegram channel, allowlist, groups disabled
│   │   ├── config-with-n8n.json5          # Telegram + n8n hooks integration
│   │   ├── config-with-nexos.json5        # Nexos gateway as model provider
│   │   └── config-with-qdrant.json5       # Telegram + Qdrant RAG + n8n hooks
│   │
│   └── prompts/                           # Specialist mode prompts
│       ├── security-audit.md              # Principle of Least Privilege audit
│       ├── execution-rules.md             # Retry limits, timeouts, failure handling
│       └── cost-optimization.md           # VPS + LLM cost reduction
```

---

## After deployment — recommended sequence

The agent guides you through these after a successful first deployment:

1. **Agent onboarding** — run on Opus: `"Run a thorough onboarding process with me"`
2. **Add models** — Anthropic direct + OpenRouter for flexibility and fallbacks
3. **Enable web search** — Perplexity via OpenRouter (~$3/1000 queries)
4. **Enable memory** — Memory Flush + Session Memory Search
5. **Auto-updates** — Docker cron job + OS security patches
6. **GitHub backup** — dedicated agent account, private repo, daily cron

---

## Requirements

- [Claude Code](https://claude.ai/code) installed: `npm install -g @anthropic-ai/claude-code`
- A Hostinger VPS with OpenClaw already installed
- SSH key access to the VPS
- A Telegram bot token ([@BotFather](https://t.me/botfather))
- At least one model provider API key

---

## Security

- `.env` is gitignored — never commit it
- All API keys are set as environment variables on the VPS, never hardcoded in config files
- Dashboard token and webhook token are always separate values
- OpenClaw UI stays on loopback; remote access via SSH tunnel or Tailscale
- Telegram bot uses `dmPolicy: "allowlist"` with numeric user IDs
- `groups: { "*": { enabled: false } }` — bot disabled in all group chats

---

## OpenClaw resources

- Official docs: https://docs.openclaw.ai
- LLM/tool index: https://docs.openclaw.ai/llms.txt
- GitHub: https://github.com/openclaw/openclaw
- Hostinger VPS setup: https://www.hostinger.com/vps/docker/openclaw
