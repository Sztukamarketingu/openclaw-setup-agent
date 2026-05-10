# Skills

## What is a skill

A skill is a file (`SKILL.md`) that contains step-by-step executable instructions for a specific, recurring task. An agent loads a skill on demand by running `exec cat` on the file path and then following the instructions inside.

Skills are not memories. They are not static facts the agent knows. They are procedural playbooks — runnable, specific, and testable.

### Skills vs. SOUL.md content

| Put it in SOUL.md | Make it a skill |
|---|---|
| Identity, tone, absolute rules | Step-by-step procedure (>15 lines) |
| Short delegation decision table | Task that involves multiple API calls |
| 2–3 sentences of domain background | Task with a clear trigger condition |
| Agent-to-agent hook pattern | Task that other agents might reuse |

Rule of thumb: if you find yourself writing a procedure with more than 15 lines in SOUL.md, extract it as a skill. Your SOUL.md stays lean; your skills stay specific.

---

## Skill anatomy

A skill file has two parts: a YAML frontmatter block, and a markdown body with numbered steps.

```markdown
---
name: skill-name-kebab-case
description: One sentence — what this skill does and when to use it. The agent reads this to decide which skill fits the current situation.
---

## Trigger

[When should the agent use this skill? Describe the input condition.]

## Step 1: [Step name]

[Exact bash command or delegation instruction. No vague prose.]

```bash
curl -s ...
```

## Step 2: [Step name]

...

## Report to main

[The closing step — always send a structured result back to the orchestrator.]

```bash
python3 - << 'EOF'
import subprocess, json
payload = {"agentId": "main", "message": json.dumps({
    "relay_message": "SKILL_RESULT",
    "data": { ... }
})}
subprocess.run(["curl", "-s", "-X", "POST", "http://127.0.0.1:18789/hooks/agent",
    "-H", "Authorization: Bearer ${OPENCLAW_HOOKS_TOKEN}",
    "-H", "Content-Type: application/json",
    "-d", json.dumps(payload)], capture_output=True)
print("reported to main")
EOF
```
```

### Frontmatter fields

- `name` — kebab-case identifier. Matches the directory name under `skills/`.
- `description` — one sentence. The agent compares this against the current situation to decide which skill to load. Write it as "Use when [condition]" — that phrasing maps directly to how agents scan the skill table in SOUL.md.

---

## How the agent selects and loads a skill

### Step 1: Skill table in SOUL.md

Each agent's SOUL.md contains a skill table section:

```markdown
## Skills — load when relevant

exec cat /data/.openclaw/workspace-AGENTID/skills/SKILL_NAME/SKILL.md

| Trigger | Skill |
|---------|-------|
| Hook contains source: "email" and intent: "new_inquiry" | `intake-new-email` |
| Message contains "MORNING_BRIEFING" | `intake-morning-briefing` |
| DM from user — simple approval ("ok", "wyślij") | `intake-dm-handler` |
```

The agent reads the table, matches the trigger description to the current context, then runs `exec cat` on the matching skill file.

### Step 2: exec cat loads the instructions

```bash
exec cat /data/.openclaw/workspace-AGENTID/skills/intake-new-email/SKILL.md
```

The agent reads the file contents and treats them as the active instructions for the current task.

### Step 3: Agent follows the skill steps

The skill replaces general reasoning for the duration of the task. If step 3 says "run this curl command," the agent runs it. If step 4 says "delegate to pipeline," the agent sends the hook.

---

## Directory structure

```
/data/.openclaw/
  workspace/                    ← main agent
    SOUL.md
    skills/
      decision-new-inquiry/
        SKILL.md
      decision-offer-builder/
        SKILL.md

  workspace-intake/             ← intake agent
    SOUL.md
    skills/
      intake-new-email/
        SKILL.md
      intake-marquiz/
        SKILL.md
      intake-dm-handler/
        SKILL.md

  workspace-email/              ← email agent
    SOUL.md
    skills/
      email-client-response/
        SKILL.md
      email-artist-inquiry/
        SKILL.md
```

Each agent has its own `skills/` directory under its workspace. Skills are not shared across agents — if two agents need similar logic, copy and adapt (the specialization is usually worth it).

---

## Installing a skill on VPS

```bash
# Create skill directory
mkdir -p /data/.openclaw/workspace-AGENTID/skills/SKILL_NAME

# Copy skill file
scp output/skill-SKILL_NAME.md user@vps:/data/.openclaw/workspace-AGENTID/skills/SKILL_NAME/SKILL.md

# Fix ownership (OpenClaw runs as node user inside Docker)
chown -R node:node /data/.openclaw/workspace-AGENTID/skills/
```

Then add the skill to the agent's SOUL.md trigger table. No container restart needed — the agent reads the file on next `exec cat`.

**Important:** If you install a skill mid-session, the agent does not automatically discover it. Either start a new session (`/new`) or explicitly tell the agent to exec the skill file path.

---

## Writing good skill steps

### Use concrete bash commands, not descriptions

Bad:
```
## Step 2: Fetch email body
Retrieve the full email content from Gmail using the message ID.
```

Good:
```
## Step 2: Fetch email body

```bash
gog -a user@company.com gmail messages get {message_id} --include-body
```
```

### Use Python + subprocess for JSON payloads, not curl with -d arguments

Bad (breaks on quotes and special characters):
```bash
curl -s -X POST http://127.0.0.1:18789/hooks/agent \
  -H "Content-Type: application/json" \
  -d '{"agentId":"main","message":"{\"relay_message\":\"done\",\"data\":{\"key\":\"some long value with 'quotes'\"}}"}'
```

Good:
```bash
python3 - << 'EOF'
import subprocess, json
payload = {
    "agentId": "main",
    "message": json.dumps({
        "relay_message": "done",
        "data": {"key": "some long value with 'quotes'"}
    })
}
subprocess.run([
    "curl", "-s", "-X", "POST", "http://127.0.0.1:18789/hooks/agent",
    "-H", "Authorization: Bearer ${OPENCLAW_HOOKS_TOKEN}",
    "-H", "Content-Type: application/json",
    "-d", json.dumps(payload)
], capture_output=True)
print("sent")
EOF
```

The Python heredoc approach handles any string content without shell escaping issues.

### Always end with a report to main (or to the calling agent)

Every skill must have a final step that sends a structured result back to whoever called it. Without this, the orchestrator does not know the task finished and may re-trigger or stall.

### Make each step independently verifiable

Add a print or echo at the end of bash blocks so the session log shows what happened:

```bash
TOPIC_ID=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['result']['message_thread_id'])")
echo "topic_id=$TOPIC_ID"
```

### Use variables from the incoming hook payload

Reference the structured data from the hook in step instructions using `{variable_name}` notation. Document the expected input fields at the top of the skill file.

---

## Testing a skill

### Send a test hook from the VPS

```bash
curl -s -X POST http://127.0.0.1:18789/hooks/agent \
  -H "Authorization: Bearer ${OPENCLAW_HOOKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "intake",
    "message": "{\"relay_message\": \"TEST\", \"data\": {\"source\": \"email\", \"sender_email\": \"test@example.com\", \"subject\": \"Test inquiry\", \"message_id\": \"msg123\"}}"
  }'
```

### Read the session file

OpenClaw writes session logs to the agent's workspace. Check the latest session:

```bash
ls -lt /data/.openclaw/workspace-intake/sessions/ | head -5
cat /data/.openclaw/workspace-intake/sessions/LATEST_SESSION_FILE.json | python3 -m json.tool | tail -100
```

Look for the `exec cat` call (confirms skill was loaded) and the final curl to the hooks endpoint (confirms report was sent).

### Common test scenarios

1. Happy path — all required fields present in the payload
2. Missing optional field — skill should handle gracefully (e.g., "artist: BRAK")
3. Downstream agent unavailable — skill should report error to main, not silently fail

---

## Real example: intake-new-email

This skill handles a new client inquiry arriving via email through an n8n webhook. It illustrates the full pattern: structured input, multiple steps, delegation to other agents, and a final report.

**Trigger condition:** Hook payload contains `source: "email"` and `intent: "new_inquiry"` (no existing task for this thread).

**Steps:**

1. Check `topic_mappings` via pipeline — is there an existing task for this thread ID?
2. Fetch full email body from Gmail (the n8n hook may carry only a truncated preview)
3. Search Qdrant for past correspondence with this sender or this artist
4. Search CRM for an existing lead from this sender
5. If no existing task: create a Telegram forum topic for this inquiry + create a task in the project management tool
6. Write a "What we know" summary to the Telegram topic
7. Send a structured context package to `main` — sender, subject, extracted event details, history, CRM match, Telegram topic ID, task ID

`main` receives this package and starts the decision skill (`decision-new-inquiry`) with full context already assembled. It never has to do the lookup work itself.

This separation is the core value of the intake pattern: by the time the orchestrator sees a new inquiry, all the assembly work is done.

---

## Common mistakes

### Quoting fails when passing long text as curl argument

When a skill needs to pass multi-line text or text with special characters to a bash command, shell quoting becomes unreliable. Use Python stdin or heredocs instead of building long `-d '...'` strings.

### Skill ends without reporting back

The most common mistake. The agent completes all the steps, the work is done, but `main` never hears about it. Without a final hook back to the orchestrator, the task stalls. Every skill must end with a `Report to main` (or `Report to [calling agent]`) step.

### Skill trigger is too broad

A skill named "handle-email" with description "Use when an email arrives" will be selected for every email — including follow-ups, system notifications, and bounces — when you only wanted it for new client inquiries. Make trigger conditions specific: "Use when hook contains `source: email` AND `intent: new_inquiry` AND no existing task_id."

### Skill duplicates what is already in SOUL.md

If a 3-step procedure is already in SOUL.md and works fine, do not extract it as a skill just to have a skill. Skills add value when they are long, specific, and triggered by clear conditions. Short procedures belong in SOUL.md.

### Installing without fixing ownership

If you `scp` a skill file as `root` but the OpenClaw container runs as `node`, the agent gets a permission error on `exec cat` and the skill silently fails (or the agent hallucinates a response instead of reading the file). Always run `chown -R node:node /data/.openclaw/` after any file upload.
