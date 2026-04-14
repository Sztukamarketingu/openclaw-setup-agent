# Memory Management

## The problem

By default, OpenClaw loses context in long sessions and doesn't remember previous conversations. Two settings fix this.

---

## Setting 1: Memory Flush before compaction

When a session gets too long, OpenClaw must compact it — compress history to fit the context window. Without Memory Flush, it does this blindly and may lose important context.

**With Memory Flush enabled:** before compacting, the agent automatically saves key information to memory files (`MEMORY.md` or `memory/*.md`). After compaction it still knows what mattered.

```json
{
  "compaction": {
    "memoryFlush": {
      "enabled": true
    }
  }
}
```

---

## Setting 2: Session Memory Search

By default, `memory_search` only searches memory files (MEMORY.md + memory/*.md). With Session Memory Search enabled, it also searches **transcripts of previous sessions** — ask "what did we discuss two weeks ago?" and it finds it even if you never saved it to MEMORY.md.

```json
{
  "memorySearch": {
    "experimental": {
      "sessionMemory": true,
      "sources": ["memory", "sessions"]
    }
  }
}
```

> ⚠️ Session Memory Search is experimental — may change in future versions.

---

## How to enable both

Send to agent:

```
Enable memory flush before compaction and session memory search in my config.
Set compaction.memoryFlush.enabled to true and set memorySearch.experimental.sessionMemory to true
with sources including both memory and sessions. Apply the config changes.
```

Verify it works:

```
Search memory and previous sessions: what did we last talk about?
```

If it returns results from previous conversations (not just MEMORY.md) — session search is working.

---

## Memory file locations

| Path | Purpose |
|------|---------|
| `/data/.openclaw/workspace/MEMORY.md` | Primary memory file — key facts, preferences, context |
| `/data/.openclaw/workspace/memory/` | Extended memory — separate files by topic |
| `/data/.openclaw/workspace/memory/archive.md` | Archived old entries |

---

## Good memory habits

**`/compact` before a new task**
Clears context from the previous task. Without this the agent mixes information from different topics and gets confused.

**"Remember this" after an important task**
After finishing a task, write:
```
Save the key decisions from this task to memory and tell me what you saved.
```
The agent writes to MEMORY.md and confirms what it kept.

**`/status` to check token usage**
Shows how many tokens you've used in the session and how many remain. Check before longer tasks to know if you're approaching the limit.

---

## Automated memory hygiene (cron jobs)

Set up dedicated cron jobs for memory management — separate from heartbeat, so they don't interfere with other processes.

Send to agent:

```
Set up separate cron jobs for memory management (do not modify heartbeat):

1. CRON: every 6 hours — "Memory Review"
   - Review recent entries in memory (MEMORY.md + memory/*.md)
   - Check for pending tasks or action items
   - If session is long — run /compact

2. CRON: every Monday at 9:00 — "Weekly Memory Cleanup"
   - Review full MEMORY.md
   - Archive outdated entries (move to memory/archive.md)
   - Remove duplicates and consolidate repeated information
   - Note key decisions from the previous week

3. CRON: 1st day of month at 9:00 — "Monthly Memory Audit"
   - Full audit — review all files in memory/
   - Check if saved information is still accurate
   - Remove what is no longer relevant
```

---

## Onboarding — teach the agent who you are

After deployment, do a one-time onboarding session so the agent knows your context, preferences, and needs. It saves everything to memory and uses it in every subsequent conversation.

**Use Opus for onboarding** — it builds the richest user profile:

```
/model Opus
```

Then:

```
Run a thorough onboarding process with me according to your guidelines.
```

The agent asks a series of questions — answer specifically. The more you share, the better it works.

Verify what it retained:
```
Summarize briefly what you know about me and my tasks.
```

If anything is missing or wrong — correct it. Then clear context:
```
/compact
```

> **Why Opus?** Opus understands nuance better, asks sharper questions, and creates a richer user profile. It's a one-time cost (~$1-2 for the session) — then you work daily on cheaper Sonnet, which uses what Opus saved to memory.
