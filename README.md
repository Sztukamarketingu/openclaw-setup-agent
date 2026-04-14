# OpenClaw Setup Agent

A Claude Code agent that guides you through configuring OpenClaw on a Hostinger VPS — from business discovery to live deployment.

**What this agent does:**
1. Asks about your business and communication needs
2. Generates a custom OpenClaw configuration (agent personality, model, channels)
3. Deploys the configuration to your VPS via Hostinger API and SSH
4. Verifies the setup end-to-end

**Assumes:** OpenClaw is already installed and network-secured on your VPS.

---

## Quick start

```bash
git clone https://github.com/Sztukamarketingu/openclaw-setup-agent
cd openclaw-setup-agent
cp .env.example .env
# Fill in .env with your credentials (see below)
claude
```

The agent starts automatically and guides you through the rest.

---

## Required credentials

Open `.env` and fill in the **REQUIRED** fields before starting:

| Variable | Where to get it |
|----------|-----------------|
| `HOSTINGER_API_KEY` | Hostinger panel → API tokens |
| `VPS_HOSTNAME` | Your VPS IP address |
| `VPS_USERNAME` | SSH username (usually `root` or `ubuntu`) |
| `VPS_SSH_KEY_PATH` | Path to your SSH private key |
| `ANTHROPIC_API_KEY` | console.anthropic.com (or use OpenAI/Nexos/OpenRouter) |
| `TELEGRAM_BOT_TOKEN` | Create a bot via [@BotFather](https://t.me/botfather) |
| `OPENCLAW_HOOKS_TOKEN` | Generate a random 32+ character string |

---

## What gets generated

The agent creates these files in `output/` (gitignored):

| File | Description |
|------|-------------|
| `output/SOUL.md` | Agent identity — name, personality, capabilities, restrictions |
| `output/config.json5` | OpenClaw configuration — model, channels, hooks, security |
| `output/env-vars.txt` | Environment variables to set on the VPS |

You review and approve each file before anything is deployed.

---

## Optional integrations

Add these to `.env` if you want them:

- **n8n** — connect n8n workflows to trigger OpenClaw agents
- **Nexos gateway** — route models through Nexos for cost control and budgets
- **OpenRouter** — access multiple model providers through one key
- **Qdrant** — semantic memory and RAG (advanced, later phase)

---

## Project structure

```
openclaw-setup-agent/
├── CLAUDE.md                        # Agent instructions (Claude Code reads this)
├── README.md                        # This file
├── .env.example                     # Credential template
├── .gitignore
├── output/                          # Generated files (gitignored)
├── docs/
│   ├── knowledge/
│   │   ├── openclaw-config.md       # Config file reference
│   │   ├── hostinger-vps.md         # Hostinger API reference
│   │   ├── telegram-setup.md        # Telegram channel setup
│   │   ├── n8n-integration.md       # n8n webhook contract
│   │   ├── nexos-integration.md     # Nexos gateway setup
│   │   └── security.md              # Security best practices
│   ├── profiles/
│   │   ├── agency.md                # Agency / talent management
│   │   ├── ecommerce.md             # Online store
│   │   ├── support.md               # Customer support
│   │   └── saas.md                  # SaaS / tech business
│   └── templates/
│       ├── soul-template.md         # SOUL.md template
│       ├── config-minimal.json5     # Minimal config
│       ├── config-with-telegram.json5
│       └── config-with-n8n.json5
```

---

## Requirements

- [Claude Code](https://claude.ai/code) installed (`npm install -g @anthropic-ai/claude-code`)
- A Hostinger VPS with OpenClaw already installed
- SSH key access to the VPS
- A Telegram bot token
- An API key for at least one model provider

---

## Security

- `.env` is gitignored — never commit it
- All API keys are set as environment variables on the VPS, never in config files
- The agent enforces token separation between dashboard access and webhook tokens
- OpenClaw UI stays on loopback; remote access via SSH tunnel or Tailscale

---

## OpenClaw resources

- Official docs: https://docs.openclaw.ai
- LLM/tool index: https://docs.openclaw.ai/llms.txt
- GitHub: https://github.com/openclaw/openclaw
