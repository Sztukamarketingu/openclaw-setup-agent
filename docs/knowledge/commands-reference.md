# OpenClaw Agent Commands — Reference

Commands are typed directly in chat with the agent (Telegram, dashboard — no difference).

---

## Status & Context

| Command | What it does |
|---------|-------------|
| `/status` | Session status, token usage, active model |
| `/context [list\|detail\|json]` | Context breakdown — file sizes, tool schemas, skills |
| `/whoami` (alias `/id`) | Your sender ID |
| `/usage off\|tokens\|full\|cost` | Footer with token/cost info after each response |

---

## Model & Thinking

| Command | What it does |
|---------|-------------|
| `/model` | List models + switch (use aliases like `/opus`, `/sonnet`) |
| `/model Sonnet` | Switch to Claude Sonnet 4.5 |
| `/model Opus` | Switch to Claude Opus 4.6 (most capable) |
| `/model Haiku` | Switch to Haiku (fastest, cheapest) |
| `/think` (alias `/thinking`, `/t`) | Set thinking level (extended reasoning) |
| `/reasoning on\|off\|stream` | Show/hide reasoning in responses |
| `/verbose on\|full\|off` | Debug mode — more info about agent operation |
| `/elevated on\|off\|ask\|full` | Execute without permission prompts (⚠️ use carefully) |

---

## Session

| Command | What it does |
|---------|-------------|
| `/reset` or `/new [model]` | Start new session (optionally with a different model) |
| `/compact [instructions]` | Compress old history — frees context window |
| `/stop` | Stop the current agent run |
| `/queue` | Message queue settings |

---

## Skills & Tools

| Command | What it does |
|---------|-------------|
| `/skill [input]` | Run a skill by name |
| `/bash` (alias `!`) | Host shell — requires `commands.bash: true` in config |

---

## Config (owner only)

Requires the corresponding flags in config to be enabled.

| Command | What it does | Requires |
|---------|-------------|---------|
| `/config show\|get\|set\|unset` | Edit `openclaw.json` from chat | `commands.config: true` |
| `/debug show\|set\|unset\|reset` | Runtime overrides (temporary changes) | `commands.debug: true` |
| `/restart` | Restart gateway | `commands.restart: true` |

---

## TTS

| Command | What it does |
|---------|-------------|
| `/tts off\|always\|inbound\|tagged\|status` | Control text-to-speech — agent reads responses aloud |

---

## Group chat

| Command | What it does |
|---------|-------------|
| `/activation mention\|always` | Activation mode — respond to @mention or to everything |
| `/allowlist` | Manage allowlist in a group |

---

## Multi-channel

| Command | What it does |
|---------|-------------|
| `/dock-telegram` | Route responses to Telegram |
| `/dock-discord` | Route responses to Discord |
| `/dock-slack` | Route responses to Slack |

---

## Most useful daily commands

```
/status             → check session health before a long task
/model Opus         → switch to Opus for complex reasoning or onboarding
/model Sonnet       → switch back to Sonnet for regular work
/compact            → clear context before starting a new topic
/usage cost         → see how much this session cost
```

---

## Quick reference for troubleshooting

```
/status             → confirm which model is active
/verbose on         → see what tools the agent is calling
/debug show         → see current runtime overrides
```
