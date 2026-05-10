# Multi-Agent Architecture

## When to use multiple agents

Start with a single agent. A well-written SOUL.md handles most business cases. Add agents only when you hit a clear structural limit.

### Add a second agent when:

| Signal | Why it matters |
|--------|----------------|
| SOUL.md exceeds 15 000 tokens | The model loses earlier instructions mid-conversation; splitting into smaller focused files is cheaper and more reliable |
| You need different models per role | Specialist tasks (writing, analysis) warrant Opus; daily intake and routing is fine on Haiku — 5–10x cost difference |
| You need different permission scopes | An email-sending agent should always require human approval; an intake agent should not. Separate agents enforce this structurally, not just via prompting |
| External triggers flood the main agent | If n8n sends 30 hooks per day and 25 are routine, an intake agent filters noise before it reaches the orchestrator |
| A role clearly maps to one specialist area | Database lookups, CRM updates, outbound email — each has a well-defined input/output contract that a dedicated agent can own |

### Do NOT add agents when:

- Your SOUL.md is under 10 000 tokens and works fine
- The "second agent" would only do one trivial action
- You are thinking of agents as tabs in a browser — each with a different topic

---

## The four agent archetypes

### Orchestrator (`main`)

The decision-maker. Receives context from other agents, reasons about what to do next, and delegates execution back to specialists.

**Critical rule:** The orchestrator makes zero direct API calls. No Gmail. No CRM. No database. It only:
1. Reads context packages sent by other agents
2. Makes decisions
3. Delegates with explicit instructions
4. Waits for reports before acting again
5. Writes to the human-facing channel (Telegram, Discord)

If your `main` agent is calling APIs directly, it will grow past 15k tokens within weeks and become unreliable.

### Intake agent

The gatekeeper. All external triggers (n8n webhooks, Telegram messages, form submissions, scheduled crons) land here first.

Intake decides: is this simple enough to handle alone, or does it need the orchestrator? For simple cases (status update, acknowledged reply), intake handles it and reports. For complex decisions, intake packages all available context — history, CRM records, relevant documents — and delivers it to `main` in a single structured payload.

This protects the orchestrator from noise and ensures `main` always has full context when it wakes.

### Specialist agent

Has deep knowledge and tools in one domain. Receives explicit task instructions from `main`, executes, and reports back.

Examples:
- `artist` — Airtable price lists, availability lookup, browser fallback for unlisted artists
- `catalog` — product database, inventory, pricing rules
- `docs` — document generation, template filling, PDF output (good candidate for Opus)

Specialists should have narrow tool access — only what their domain requires.

### Execution agent

A specialist for side-effectful operations that require human approval or careful audit trails.

The `email` agent is the canonical example: it can draft and send emails, but it never sends without explicit human confirmation. By isolating this in a separate agent, you get a structural guarantee — not just a SOUL.md instruction that might be forgotten.

---

## Designing from business process

Map your actual business flow before choosing agents. A booking agency example:

```
External world:
  New inquiry email           → n8n hook → intake
  Client fills web form       → n8n hook → intake
  Morning scheduled briefing  → cron hook → intake

Intake processes:
  Simple follow-up            → handle locally, report to main
  New inquiry                 → enrich with history, CRM, knowledge base → send to main

Main decides:
  Artist available? Check.    → delegate to artist
  Draft reply to client?      → delegate to email, wait for approval
  Update CRM status?          → delegate to pipeline

Specialists execute:
  artist  → Airtable lookup, availability, pricing
  email   → Gmail send/read, draft queue, approval gating
  pipeline → ClickUp tasks, CRM updates, shared state (topic_mappings)
```

Draw your own process map before writing any SOUL.md. If a step appears in multiple agents, you have either a duplicate or a candidate for a shared state file.

---

## SOUL.md size and agent splitting

A single-agent SOUL.md that grows past ~15 000 tokens becomes unreliable. The model processes the system prompt linearly, and instructions in the first half tend to fade when the context window fills.

**How to decide what goes where:**

| Content type | Keep in SOUL.md | Move to skill |
|---|---|---|
| Identity, tone, absolute rules | Always | Never |
| Business domain knowledge (2–3 paragraphs) | Yes | No |
| Step-by-step procedures (>10 steps) | No | Yes |
| API call patterns for recurring tasks | No | Yes |
| Delegation decision table | Yes (short) | No |

When splitting a monolithic SOUL.md into multiple agents, ask: "Which decisions require full business context (→ main) and which can be answered by a specialist with only domain knowledge (→ specialist)?"

---

## Inter-agent communication

### Hook format

One agent wakes another by sending an HTTP POST to the local hooks proxy:

```bash
python3 - << 'EOF'
import subprocess, json
payload = {
    "agentId": "target-agent-id",
    "message": json.dumps({
        "relay_message": "SHORT_DESCRIPTION_OF_REQUEST",
        "data": {
            "key": "value"
        }
    })
}
subprocess.run([
    "curl", "-s", "-X", "POST", "http://127.0.0.1:18789/hooks/agent",
    "-H", "Authorization: Bearer ${OPENCLAW_HOOKS_TOKEN}",
    "-H", "Content-Type: application/json",
    "-d", json.dumps(payload)
], capture_output=True)
print("hook sent")
EOF
```

Use Python + `subprocess` instead of raw `curl` with JSON arguments. Long JSON strings passed as shell arguments break on special characters — stdin via Python is reliable.

### The relay_message pattern

When a specialist reports back to main, it uses the same hook format. The `relay_message` field acts as the subject line — a short string the orchestrator can read in the session log to understand what arrived. The `data` field carries the structured payload.

```json
{
  "relay_message": "ARTIST_LOOKUP_COMPLETE",
  "data": {
    "artist": "Name",
    "available": true,
    "price_client": 15000,
    "price_agency": 12000,
    "manager_email": "manager@example.com",
    "note": "Peak season surcharge applies"
  }
}
```

### Async nature

Agent-to-agent hooks are fire-and-forget from the sender's perspective. The response does not come back as an HTTP reply — it arrives as a new session message when the target agent finishes and sends its own hook back to the caller. Design your SOUL.md instructions around this: always "wait for report before proceeding."

---

## Config: defining multiple agents

```json5
agents: {
  list: [
    {
      id: "main",
      name: "Main Orchestrator",
      model: {
        primary: "anthropic/claude-sonnet-4-6",
      },
      workspace: "/data/.openclaw/workspace",
    },
    {
      id: "intake",
      name: "Intake Agent",
      model: {
        primary: "anthropic/claude-haiku-4-5",   // Cheaper — high volume, simple decisions
      },
      workspace: "/data/.openclaw/workspace-intake",
    },
    {
      id: "specialist",
      name: "Domain Specialist",
      model: {
        primary: "anthropic/claude-haiku-4-5",
      },
      workspace: "/data/.openclaw/workspace-specialist",
      tools: {
        allow: ["read_file", "exec"],   // Restricted to domain tools only
      },
    },
    {
      id: "email",
      name: "Email Agent",
      model: {
        primary: "anthropic/claude-haiku-4-5",
      },
      workspace: "/data/.openclaw/workspace-email",
    },
  ],
},
```

### Naming and ID conventions

- `id` — short, lowercase, no spaces (`main`, `intake`, `artist`, `email`, `pipeline`). This is what you reference in hooks and configs.
- `workspace` — always `/data/.openclaw/workspace-{id}`. The `main` agent uses `/data/.openclaw/workspace` (no suffix) by convention.
- `model` — Orchestrators and writers use Sonnet or Opus. Intake and specialists use Haiku. Pipeline/CRM agents use the cheapest model that can follow structured instructions (Haiku 4.5 or DeepSeek v3 via OpenRouter).

---

## Shared state between agents

Agents often need to share a lookup table — for example, a mapping from email thread ID to Telegram topic ID. Do not let multiple agents write this from memory.

Pattern: one agent owns the file as single source of truth. Others delegate reads and writes to that agent.

```
topic_mappings.json   →  owned and written by pipeline agent only
                      →  other agents send "WRITE topic_mappings" delegation to pipeline
                      →  other agents can request "READ topic_mappings for thread_id=X" from pipeline
```

This prevents race conditions and keeps the audit trail in one place.

---

## Step-by-step: adding a new agent

### 1. Decide what the agent owns

Write one sentence: "This agent does X and only X." If you cannot write that sentence, you are not ready to create the agent.

### 2. Add it to config

Add a new entry in `agents.list` with a unique `id` and `workspace`. Choose the model based on role (see naming conventions above).

### 3. Create the workspace directory on VPS

```bash
ssh user@vps "mkdir -p /data/.openclaw/workspace-AGENTID && chown -R node:node /data/.openclaw/"
```

### 4. Write the SOUL.md

Keep it under 8 000 tokens. Include:
- Role (2–3 sentences)
- The hook-reporting pattern (show the exact Python snippet for this agent's ID)
- Delegation table (what to delegate and to whom)
- Skills section (exec cat pattern + trigger table)
- Absolute rules (what this agent must never do)

### 5. Upload SOUL.md

```bash
scp output/SOUL-AGENTID.md user@vps:/data/.openclaw/workspace-AGENTID/SOUL.md
chown node:node /data/.openclaw/workspace-AGENTID/SOUL.md
```

### 6. Restart the OpenClaw container

```bash
docker compose -f /data/docker-compose.yml up -d
```

After recreating the container, restart `hooks-proxy` as well (see gotchas below).

### 7. Verify the new agent is listed

```bash
docker exec $(docker ps -q) openclaw agents list
```

### 8. Send a test hook

```bash
curl -s -X POST http://127.0.0.1:18789/hooks/agent \
  -H "Authorization: Bearer ${OPENCLAW_HOOKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"agentId":"your-new-agent","message":"PING"}'
```

Check the session file to confirm the agent received it and responded.

---

## Common mistakes

### hooks-proxy orphan after container recreate

`hooks-proxy` (socat) uses `network_mode: "service:openclaw"` in docker-compose. When the `openclaw` container is recreated, the proxy loses its network attachment and silently stops forwarding. After any `docker compose up -d`, always restart `hooks-proxy`:

```bash
docker compose -f /data/docker-compose.yml restart hooks-proxy
```

If hooks stop arriving and everything else looks fine, this is the first thing to check.

### Invalid field in agent config causes silent reload failure

Config fields that do not exist in the OpenClaw schema are silently ignored during hot-reload. If a new config option appears to have no effect after `config.json5` is updated, validate the field name against the official schema. Common trap: `channels.telegram.agentId` is not a recognized field in older releases — set routing in `hooks.allowedAgentIds` instead.

### Billing cache after top-up

If your Anthropic account runs out of credits and you top up, the container may cache the "insufficient credits" error and continue refusing requests. Restart the container after topping up.

### PENDING_PIPELINE placeholder in generated output

When a specialist agent embeds a delegation command in its response as a text placeholder (e.g., `PENDING_PIPELINE: update task X`) without executing the hook, the pipeline agent never receives the instruction. Skills and SOUL.md must include runnable bash code for every delegation — not prose descriptions of what to do.

### n8n API: `active` field is read-only in PUT

When updating a workflow via the n8n API, the `active` field cannot be included in the PUT body — it returns a 400 error. Use the dedicated activate/deactivate endpoints instead.

### Agent does not pick up new skills after session start

Skills installed on disk are loaded when the agent reads `exec cat /data/.openclaw/workspace-AGENTID/skills/SKILL_NAME/SKILL.md`. If the skill was installed mid-session, the agent does not automatically discover it. Either restart the session or explicitly tell the agent to exec the skill file.

---

## Business profiles — which agent set fits your business

### Booking agency / talent management

- `main` (Sonnet) — orchestrator
- `intake` (Haiku) — triage: email webhooks, form submissions, DMs
- `artist` or `catalog` (Haiku) — price list, availability, manager contacts
- `email` (Haiku) — outbound correspondence, always requires approval
- `pipeline` (Haiku / DeepSeek v3) — CRM updates, task management, shared state

### E-commerce / online store

- `main` (Sonnet) — orchestrator
- `intake` (Haiku) — incoming orders, customer inquiries
- `catalog` (Haiku) — product data, prices, availability
- `support` (Haiku) — customer replies, FAQ resolution
- `ops` (Haiku) — order status, warehouse, fulfillment tracking

### Marketing agency / freelancer

- `main` (Sonnet) — orchestrator
- `intake` (Haiku) — lead triage, form submissions
- `writer` (Opus) — copy, proposals, campaign briefs (expensive but justified)
- `crm` (Haiku) — HubSpot / Pipedrive updates

### Consulting / coaching

- `main` (Sonnet) — orchestrator
- `intake` (Haiku) — calendar requests, inbound inquiries
- `scheduler` (Haiku) — calendar checks, booking confirmation
- `docs` (Opus) — proposals, session notes, deliverable reports

### Single person, low volume

**Do not use multi-agent.** One well-configured `main` agent with good SOUL.md is faster to build, cheaper to run, and easier to debug. Multi-agent overhead pays off only when you have volume, permission boundaries, or token limits that a single agent cannot handle.
