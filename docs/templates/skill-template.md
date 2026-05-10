---
# REQUIRED: kebab-case name matching the directory name under skills/
name: your-skill-name

# REQUIRED: one sentence starting with "Use when" — the agent reads this to select the right skill
# Be specific about the trigger condition, not just the topic
# Example: "Use when intake reports source: email and intent: new_inquiry (no existing task_id for this thread)"
description: Use when [specific trigger condition here]
---

## Input fields expected

<!-- 
  Document what fields this skill expects from the incoming hook or context.
  This is the contract — whoever calls this skill must provide these fields.
  
  Example:
  - `task_id` — existing task ID in project management tool
  - `sender_email` — email address of the person who sent the inquiry
  - `subject` — email subject line
  - `message_id` — Gmail message ID for fetching full body
  - `topic_id` — optional; if present, this is a follow-up, not a new inquiry
-->

| Field | Required | Description |
|-------|----------|-------------|
| `field_name` | yes / no | What this field contains |
| `field_name_2` | yes / no | What this field contains |

---

## Trigger

<!--
  Describe in plain language when this skill applies.
  The agent reads this section after loading the file to confirm it picked the right skill.
  
  Example:
  "Use this skill when the hook payload contains source: "email" and intent: "new_inquiry",
  and pipeline confirmed there is no existing task for this thread_id."
-->

[Describe the exact condition under which this skill applies]

---

## Step 1: Validate input

<!--
  Always start with a quick sanity check on required fields.
  If something required is missing, stop and report the error — do not proceed with incomplete data.
-->

```bash
# Check that required fields are present
# Replace FIELD_VALUE with the actual value from the hook payload
if [ -z "FIELD_VALUE" ]; then
  echo "ERROR: required field missing"
  exit 1
fi
echo "input validated: FIELD_VALUE"
```

---

## Step 2: Fetch or look up data

<!--
  Do the necessary data retrieval: API calls, database lookups, file reads.
  One bash block per distinct operation. Print results so the session log is readable.
  
  Examples:
  - Read a record from Airtable
  - Get email body from Gmail
  - Query Qdrant for similar past events
  - Look up a lead in the CRM
-->

```bash
# Example: fetch a record from an external API
RESULT=$(curl -s -X GET "https://api.example.com/records/RECORD_ID" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json")

# Extract the field you need
VALUE=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['field'])")
echo "fetched value=$VALUE"
```

---

## Step 3: Delegate to another agent (if needed)

<!--
  If this skill needs work done by a specialist agent (e.g., artist lookup, CRM update),
  delegate here and wait for the report before continuing.
  
  Use Python + subprocess for all JSON payloads — never raw curl with -d for complex JSON.
  The relay_message is the subject line; data carries the structured payload.
-->

```bash
python3 - << 'EOF'
import subprocess, json

payload = {
    "agentId": "target-agent-id",   # Replace with the actual agent ID
    "message": json.dumps({
        "relay_message": "TASK_DESCRIPTION",
        "data": {
            "key1": "VALUE_FROM_STEP_2",
            "key2": "OTHER_VALUE"
        }
    })
}

result = subprocess.run([
    "curl", "-s", "-X", "POST", "http://127.0.0.1:18789/hooks/agent",
    "-H", "Authorization: Bearer ${OPENCLAW_HOOKS_TOKEN}",
    "-H", "Content-Type: application/json",
    "-d", json.dumps(payload)
], capture_output=True, text=True)

print("delegated to target-agent-id:", result.stdout)
EOF
```

<!--
  After sending the delegation, the skill should WAIT for the specialist to report back
  before proceeding to the next step. Note this explicitly:
-->

Wait for `target-agent-id` to report back before continuing to Step 4.

---

## Step 4: Execute the main action

<!--
  The core operation of this skill: write a message, update a record, create a task, send a notification.
  Keep each action in its own bash block with clear output.
  
  Example: write a Telegram message to a forum topic
-->

```bash
# Example: send a message to a Telegram forum topic
curl -sS "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage" \
  -H "Content-Type: application/json" \
  -d "{
    \"chat_id\": -1001234567890,
    \"message_thread_id\": TOPIC_ID,
    \"text\": \"Your message here\",
    \"parse_mode\": \"Markdown\"
  }"
```

---

## Step 5: Update shared state (if needed)

<!--
  If this skill modifies shared state (topic_mappings, a task status, a database record),
  do it here. Delegate to the pipeline/state-owning agent — never write shared state directly
  from two different agents.
-->

```bash
# Example: update task status via pipeline agent
python3 - << 'EOF'
import subprocess, json

payload = {
    "agentId": "pipeline",
    "message": json.dumps({
        "relay_message": "UPDATE_TASK_STATUS",
        "data": {
            "task_id": "TASK_ID",
            "status": "in_progress",
            "note": "YYYY-MM-DD: Skill completed"
        }
    })
}

subprocess.run([
    "curl", "-s", "-X", "POST", "http://127.0.0.1:18789/hooks/agent",
    "-H", "Authorization: Bearer ${OPENCLAW_HOOKS_TOKEN}",
    "-H", "Content-Type: application/json",
    "-d", json.dumps(payload)
], capture_output=True)
print("state updated")
EOF
```

---

## Report to main

<!--
  REQUIRED — every skill must end with a report back to the orchestrator (or to whoever called this skill).
  Without this, the orchestrator does not know the task finished and the workflow stalls.
  
  relay_message: short descriptor of what happened (used in session log scanning)
  data: structured result that the calling agent needs to continue its work
  
  Include all values that the orchestrator needs for the next decision:
  - IDs of created records
  - Results of lookups (available: true/false, price, contact info)
  - Status of actions taken
  - Any errors encountered
-->

```bash
python3 - << 'EOF'
import subprocess, json

payload = {
    "agentId": "main",   # or the ID of the agent that called this skill
    "message": json.dumps({
        "relay_message": "SKILL_NAME_COMPLETE",
        "data": {
            "status": "ok",                    # ok | error | partial
            "task_id": "TASK_ID",
            "result_field": "VALUE",           # key results
            "error": None                      # or error description if status != ok
        }
    })
}

subprocess.run([
    "curl", "-s", "-X", "POST", "http://127.0.0.1:18789/hooks/agent",
    "-H", "Authorization: Bearer ${OPENCLAW_HOOKS_TOKEN}",
    "-H", "Content-Type: application/json",
    "-d", json.dumps(payload)
], capture_output=True)
print("reported to main")
EOF
```

---

<!--
  INSTALLATION CHECKLIST (remove this block before deploying)
  
  1. Save this file as SKILL.md inside:
       /data/.openclaw/workspace-AGENTID/skills/YOUR-SKILL-NAME/SKILL.md
  
  2. Fix ownership:
       chown -R node:node /data/.openclaw/workspace-AGENTID/skills/
  
  3. Add trigger to the agent's SOUL.md skills table:
       | [trigger description] | `your-skill-name` |
  
  4. Test with a manual hook:
       curl -s -X POST http://127.0.0.1:18789/hooks/agent \
         -H "Authorization: Bearer ${OPENCLAW_HOOKS_TOKEN}" \
         -H "Content-Type: application/json" \
         -d '{"agentId":"AGENTID","message":"{\"relay_message\":\"TEST\",\"data\":{...}}"}'
  
  5. Check session log:
       ls -lt /data/.openclaw/workspace-AGENTID/sessions/ | head -3
-->
