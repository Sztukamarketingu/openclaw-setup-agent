# OpenClaw Setup — Progress & Action Plan

> This file is created by the agent after the discovery conversation and updated after every completed step.
> When you return to a new session, the agent reads this file first and continues from where you left off.
> Do not edit manually unless you want to change the plan.

---

## Business profile

| Field | Value |
|-------|-------|
| **Business** | _[2-3 sentence description]_ |
| **Agent audience** | _[customers / internal team / both]_ |
| **Agent name** | _[name]_ |
| **Tone** | _[professional / friendly / concise / formal]_ |
| **Primary channel** | _[Telegram / Discord / none]_ |
| **Model provider** | _[Anthropic / OpenAI / Nexos / OpenRouter]_ |
| **n8n integration** | _[yes / no]_ |
| **Browser automation** | _[yes — which systems / no]_ |
| **Setup path** | _[A: Minimal POC / B: POC + Nexos / C: POC + n8n automation]_ |

---

## Current phase

> **[PHASE NAME]** — [one line describing what's happening now]

Phases: `Discovery` → `Config Design` → `Deployment` → `Verification` → `Post-Setup` → `Complete`

---

## Action plan

### Phase 1: Discovery
- [ ] Business type and use case confirmed
- [ ] Agent audience defined
- [ ] Top capabilities agreed
- [ ] Knowledge sources listed
- [ ] Restrictions and escalation rules defined
- [ ] Agent name and tone chosen
- [ ] Channel selected (Telegram / Discord)
- [ ] Model provider selected
- [ ] n8n integration: yes / no
- [ ] Browser automation: yes / no
- [ ] Summary confirmed by user

### Phase 2: Configuration design
- [ ] `output/SOUL.md` generated and approved
- [ ] `output/config.json5` generated and approved
- [ ] `output/env-vars.txt` generated and approved

### Phase 3: Deployment
- [ ] VPS identified via Hostinger API
- [ ] Environment variables set on VPS
- [ ] `config.json5` uploaded to VPS
- [ ] `SOUL.md` uploaded to VPS
- [ ] OpenClaw Gateway restarted
- [ ] Gateway status: running

### Phase 4: Verification
- [ ] Telegram bot responds to test message
- [ ] Agent introduces itself with correct name and tone
- [ ] No API key errors in logs
- [ ] Hooks endpoint responds (if n8n enabled)

### Phase 5: Post-setup
- [ ] Agent onboarding completed (run on Opus)
- [ ] Models configured (Anthropic + OpenRouter)
- [ ] Web search enabled (Perplexity via OpenRouter)
- [ ] Memory Flush enabled
- [ ] Session Memory Search enabled
- [ ] Memory hygiene cron jobs set
- [ ] Auto-update cron job set
- [ ] OS unattended-upgrades enabled
- [ ] GitHub backup configured

### Phase 6: Extensions (optional, later)
- [ ] n8n workflows connected
- [ ] Browser automation configured
- [ ] Discord channel added
- [ ] Qdrant RAG set up
- [ ] Second agent added
- [ ] Cost optimization audit done

---

## Generated files

| File | Status | Notes |
|------|--------|-------|
| `output/SOUL.md` | _pending / generated / deployed_ | |
| `output/config.json5` | _pending / generated / deployed_ | |
| `output/env-vars.txt` | _pending / generated / deployed_ | |

---

## Deployment log

| Date | Step | Result |
|------|------|--------|
| | | |

---

## Known issues / blockers

_None yet._

---

## Next session: start here

> _[Agent writes a specific instruction here after each session, e.g.:]_
> "Resume from Phase 3, Step: upload config.json5 to VPS. The VPS IP is X.X.X.X, environment variables were already set. Run: scp output/config.json5 ..."

---

## Notes

_[Any decisions, trade-offs, or context worth remembering between sessions.]_
