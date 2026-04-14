# n8n ↔ OpenClaw Integration

## Overview

n8n and OpenClaw have distinct roles:

| Layer | Responsibility |
|-------|----------------|
| **n8n** | Deterministic workflows: schedules, SLA logic, integrations (CRM, email, databases), triggers |
| **OpenClaw** | Conversational AI: agent reasoning, drafting, channel delivery (Telegram), tool use |

n8n calls OpenClaw when it needs AI reasoning or content generation. OpenClaw does not replace n8n for workflows.

---

## OpenClaw webhook endpoints

| Method | Path | Use |
|--------|------|-----|
| `POST` | `{base}/hooks/agent` | Send a task to an agent (draft document, answer question, process request) |
| `POST` | `{base}/hooks/wake` | Send a system signal or notification (no LLM turn required) |

`base` = `http://127.0.0.1:18789` when n8n and OpenClaw run on the same VPS, or `https://openclaw.internal` behind a reverse proxy.

---

## Authentication

OpenClaw requires the hooks token in the **Authorization header**:

```
Authorization: Bearer {OPENCLAW_HOOKS_TOKEN}
```

**Do not use `?token=` query string** — OpenClaw returns 400 for query-string tokens.

In n8n HTTP Request node:
- Authentication: **Header Auth**
- Name: `Authorization`
- Value: `Bearer {{ $credentials.openclaw.token }}`

Store `OPENCLAW_HOOKS_TOKEN` in n8n Credentials, not in the workflow JSON.

---

## POST /hooks/agent — request fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message` | string | yes | Full context for the agent. Can be plain text or structured text containing JSON. |
| `name` | string | no | Label for this session, e.g. `"DraftOffer"` |
| `agentId` | string | no | Which agent to route to. Must be in `hooks.allowedAgentIds`. Default: `"main"` |
| `wakeMode` | string | no | `"now"` (immediate) or `"next-heartbeat"`. Default: `"now"` |
| `deliver` | boolean | no | `true`: send the agent's response to a channel. `false`: store in session only. |
| `channel` | string | no | Which channel to deliver to: `"telegram"`, `"last"`, etc. |
| `to` | string | no | Recipient on the channel (e.g. Telegram chat ID) |
| `model` | string | no | Override model for this request |
| `timeoutSeconds` | number | no | Max time to wait for agent completion |

### Response

- `200` — request accepted (async). The agent response is **not** in the HTTP response body.
- `401` — wrong or missing token
- `400` — malformed request (check token is not in query string)
- `413` — message payload too large
- `429` — too many failed auth attempts from this IP

---

## Example: trigger a document draft

```json
{
  "message": "Draft a proposal for the following client request:\n\nClient: Acme Corp\nRequest: 3-month consulting engagement, focus on SEO and content strategy\nBudget: $5,000-$8,000\nTimeline: Starting next month\n\nUse our standard proposal format. Language: English. Length: 400-600 words.",
  "name": "ProposalDraft",
  "agentId": "main",
  "wakeMode": "now",
  "deliver": false
}
```

Agent processes the request. Result appears in the OpenClaw session (accessible via Control UI).

---

## Example: draft + deliver to Telegram

```json
{
  "message": "Summarize this incoming support request and suggest a response:\n\n{customer_message}",
  "name": "SupportDraft",
  "agentId": "main",
  "wakeMode": "now",
  "deliver": true,
  "channel": "telegram",
  "to": "YOUR_TELEGRAM_CHAT_ID"
}
```

The agent's response is sent directly to Telegram.

---

## Example: POST /hooks/wake — notification signal

Use when you want to alert the agent without triggering a full LLM reasoning turn:

```json
{
  "text": "New high-priority email received from client@example.com. Subject: Urgent contract issue.",
  "mode": "now"
}
```

---

## n8n HTTP Request node setup

```
Method: POST
URL: http://{VPS_IP}:18789/hooks/agent
Headers:
  Authorization: Bearer {{ $credentials.openclaw_hooks.token }}
  Content-Type: application/json
Body (JSON):
  {
    "message": "{{ $json.your_field }}",
    "agentId": "main",
    "wakeMode": "now",
    "deliver": false
  }
```

---

## Getting the agent's output back into n8n

`POST /hooks/agent` returns 200 immediately (async). The actual agent response is **not** in the HTTP body.

**Three patterns for reading the result:**

**Pattern A — Agent writes to a system (recommended)**
Give the agent a tool that writes to ClickUp, Notion, Airtable, or your CRM. n8n polls or listens for the new record.

**Pattern B — Telegram delivery + n8n Telegram trigger**
Set `"deliver": true, "channel": "telegram"`. The human reviews in Telegram and approves. Then a separate n8n flow continues.

**Pattern C — Webhook callback**
Give the agent a tool that calls a designated n8n webhook URL with the result.

---

## OpenClaw config required for n8n integration

```json5
hooks: {
  enabled: true,
  token: "${OPENCLAW_HOOKS_TOKEN}",
  path: "/hooks",
  defaultSessionKey: "hook:n8n:ingress",
  allowRequestSessionKey: false,
  allowedSessionKeyPrefixes: ["hook:"],
  allowedAgentIds: ["main"],
},
```

---

## Security

- Run OpenClaw and n8n on the same VPS or in the same private network
- Use `http://127.0.0.1:18789` (loopback) when n8n and OpenClaw share the VPS
- If they are on separate servers: use Tailscale / private network, not public internet
- Keep `OPENCLAW_HOOKS_TOKEN` in n8n Credentials manager, not in workflow JSON
- Use a dedicated agent (e.g. `agentId: "offers"`) with restricted `tools.allow` for n8n-triggered tasks
