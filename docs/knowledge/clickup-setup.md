# ClickUp Setup — agent creates and updates tasks via REST API

How to give the OpenClaw agent access to ClickUp: task creation, status updates, custom fields, due dates, subtasks, and filtered listings. No MCP required — ClickUp REST API v2 is called directly via `curl` or `python3`.

Typical use cases: tracking a lead pipeline (new inquiry → offer sent → contract → archived), daily briefings from tasks with overdue due dates, creating subtasks for specialist follow-ups (e.g. "query sent to artist manager").

---

## Step 1 — Get the Personal API Token

User flow:

1. app.clickup.com → click your avatar (bottom left) → **Apps**
2. Under **API Token** — click **Generate** (or copy existing)
3. Copy the token (starts with `pk_`, shown once)

**Do not use OAuth** unless you are building a SaaS product. The personal API token is tied to the user account — it has full access to all workspaces the user belongs to. Good enough for an internal agent.

Validate before saving:

```bash
curl -s "https://api.clickup.com/api/v2/user" \
  -H "Authorization: ${CLICKUP_API_TOKEN}" | python3 -c '
import json, sys
u = json.load(sys.stdin)["user"]
print(f"OK — user: {u[\"username\"]} ({u[\"id\"]})")'
```

Expected: `OK — user: YourName (12345678)`

---

## Step 2 — Collect the IDs you need

The agent works with IDs, not names. Collect these before writing config or SOUL.md.

### Team ID (workspace)

```bash
curl -s "https://api.clickup.com/api/v2/team" \
  -H "Authorization: ${CLICKUP_API_TOKEN}" | python3 -c '
import json, sys
for t in json.load(sys.stdin)["teams"]:
    print(f"team_id={t[\"id\"]}  name={t[\"name\"]}")'
```

### List ID (where tasks live)

Navigate: Workspace → Space → Folder → List. The List is the container. Right-click the list name in the sidebar → **Copy link** → the URL ends in the list ID.

Or via API:

```bash
# Get spaces first
curl -s "https://api.clickup.com/api/v2/team/<TEAM_ID>/space" \
  -H "Authorization: ${CLICKUP_API_TOKEN}" | python3 -c '
import json,sys
for s in json.load(sys.stdin)["spaces"]:
    print(f"space_id={s[\"id\"]}  name={s[\"name\"]}")'

# Then lists in a folder, or directly in a space (folderless):
curl -s "https://api.clickup.com/api/v2/space/<SPACE_ID>/list?archived=false" \
  -H "Authorization: ${CLICKUP_API_TOKEN}" | python3 -c '
import json,sys
for l in json.load(sys.stdin)["lists"]:
    print(f"list_id={l[\"id\"]}  name={l[\"name\"]}  status_count={len(l.get(\"statuses\",[]))}")'
```

### User IDs (for assignees)

```bash
curl -s "https://api.clickup.com/api/v2/team/<TEAM_ID>/member" \
  -H "Authorization: ${CLICKUP_API_TOKEN}" | python3 -c '
import json,sys
for m in json.load(sys.stdin)["members"]:
    u = m["user"]
    print(f"user_id={u[\"id\"]}  name={u[\"username\"]}  email={u[\"email\"]}")'
```

Save these — you will hardcode them in SOUL.md.

### Status names (exact spelling required — see Gotcha #2)

```bash
curl -s "https://api.clickup.com/api/v2/list/<LIST_ID>" \
  -H "Authorization: ${CLICKUP_API_TOKEN}" | python3 -c '
import json,sys
for s in json.load(sys.stdin)["statuses"]:
    print(repr(s["status"]))'
```

Copy the output literally. Every status name you set via API must match this exactly.

### Custom Field IDs

```bash
curl -s "https://api.clickup.com/api/v2/list/<LIST_ID>/field" \
  -H "Authorization: ${CLICKUP_API_TOKEN}" | python3 -c '
import json,sys
for f in json.load(sys.stdin)["fields"]:
    print(f"field_id={f[\"id\"]}  name={f[\"name\"]}  type={f[\"type\"]}")'
```

For each custom field you want the agent to populate, save its `field_id`. Common fields in a pipeline setup: `telegram_topic_id`, `bitrix_lead_id`, `organizer_email`.

---

## Step 3 — Add the token to the VPS environment

```bash
ssh root@<VPS> "echo 'CLICKUP_API_TOKEN=<token>' >> /docker/<project>/.env"
```

Then recreate the container to load the new env (restart alone does NOT work):

```bash
cd /docker/<project> && docker compose up -d <service>
```

Verify:

```bash
docker exec <container> sh -c 'env | grep CLICKUP'
# expected: CLICKUP_API_TOKEN=pk_...
```

---

## Step 4 — Update SOUL.md with the ClickUp map

The agent needs to know which list it works in, which statuses exist, and which custom fields to populate. Without this it will ask the user for IDs or use the wrong status name (silent fail).

Append to `/data/.openclaw/workspace-<agent>/SOUL.md`:

```markdown
## ClickUp — task pipeline

You manage tasks in ClickUp via REST API (curl / python3 subprocess). Token is in env (`CLICKUP_API_TOKEN`), never ask the user for it.

**Key IDs:**
- List: `<LIST_ID>` — all pipeline tasks live here
- Team: `<TEAM_ID>`
- Assignees: Tomek=`<USER_ID_1>`, Joanna=`<USER_ID_2>`, Agent=`<USER_ID_3>`

**Statuses (use EXACT spelling including spaces):**
- `nowe zapytanie` — new, no action taken yet
- `w trakcie` — first response sent, lead active
- `wycena wysłana` — offer sent, waiting for decision
- `negocjacje` — negotiating terms
- `kontrakt` — contract phase
- `zamknięte` — closed (won or lost)

**Custom fields:**
- `00ac48bd-...` — `telegram_topic_id` (text)
- `02fbbbb9-...` — `bitrix_lead_id` (text)
- `811b6f5d-...` — `organizer_email` (text)

Custom field values are set via a SEPARATE PUT call after task creation — they cannot be included in the POST.
```

Replace the IDs and status names with the actual values from Step 2.

---

## Step 5 — Common API patterns (use these literally in skills)

### Create a task

```python
import subprocess, json, os, time

token = os.environ["CLICKUP_API_TOKEN"]
list_id = "<LIST_ID>"

payload = {
    "name": "Zapytanie — Organizator ABC — 2026-09-13",
    "description": "email: jan@example.com\nTemat: Koncert Sanah\nBudżet: 25 000",
    "status": "nowe zapytanie",
    "assignees": [50695900, 82609693],  # numeric IDs
    "due_date": int(time.time() * 1000) + 86400000,  # jutro w ms
    "tags": ["lead-nowe-zapytanie"]  # tag musi już istnieć w workspace
}

r = subprocess.run(
    ["curl", "-s", "-X", "POST",
     f"https://api.clickup.com/api/v2/list/{list_id}/task",
     "-H", f"Authorization: {token}",
     "-H", "Content-Type: application/json",
     "-d", json.dumps(payload)],
    capture_output=True, text=True
)
task = json.loads(r.stdout)
task_id = task["id"]
print(task_id)
```

### Set a custom field (separate call — always after task creation)

```python
field_id = "00ac48bd-88a5-4dbb-8a01-2024786f26c2"  # telegram_topic_id
value = "420"

subprocess.run(
    ["curl", "-s", "-X", "POST",
     f"https://api.clickup.com/api/v2/task/{task_id}/field/{field_id}",
     "-H", f"Authorization: {token}",
     "-H", "Content-Type: application/json",
     "-d", json.dumps({"value": value})],
    capture_output=True, text=True
)
```

### Update task status

```python
subprocess.run(
    ["curl", "-s", "-X", "PUT",
     f"https://api.clickup.com/api/v2/task/{task_id}",
     "-H", f"Authorization: {token}",
     "-H", "Content-Type: application/json",
     "-d", json.dumps({"status": "w trakcie"})],
    capture_output=True, text=True
)
```

### Update due_date (tomorrow 9AM)

```python
import datetime
tomorrow_9am = int(
    (datetime.datetime.now()
        .replace(hour=9, minute=0, second=0, microsecond=0)
     + datetime.timedelta(days=1)).timestamp() * 1000
)
subprocess.run(
    ["curl", "-s", "-X", "PUT",
     f"https://api.clickup.com/api/v2/task/{task_id}",
     "-H", f"Authorization: {token}",
     "-H", "Content-Type: application/json",
     "-d", json.dumps({"due_date": tomorrow_9am})],
    capture_output=True, text=True
)
```

### Add a comment (used as history log)

```python
import datetime
date_str = datetime.date.today().isoformat()
comment = f"+ {date_str} Wysłano pierwszą odpowiedź do org."

subprocess.run(
    ["curl", "-s", "-X", "POST",
     f"https://api.clickup.com/api/v2/task/{task_id}/comment",
     "-H", f"Authorization: {token}",
     "-H", "Content-Type: application/json",
     "-d", json.dumps({"comment_text": comment, "notify_all": False})],
    capture_output=True, text=True
)
```

### Create a subtask

```python
subtask = {
    "name": "Zapytanie do Sanah — 2026-09-13 Wrocław",
    "description": "email: manager@sanah.pl\nthread_id: 19e024b311e3e4a7\nzapytano: 2026-05-10",
    "parent": task_id,           # <-- this makes it a subtask
    "due_date": int(time.time() * 1000) + 86400000
}
r = subprocess.run(
    ["curl", "-s", "-X", "POST",
     f"https://api.clickup.com/api/v2/list/{list_id}/task",
     "-H", f"Authorization: {token}",
     "-H", "Content-Type: application/json",
     "-d", json.dumps(subtask)],
    capture_output=True, text=True
)
subtask_id = json.loads(r.stdout)["id"]
```

### List overdue tasks (for morning briefing)

```python
today_ms = int(time.time() * 1000)

r = subprocess.run(
    ["curl", "-s",
     f"https://api.clickup.com/api/v2/list/{list_id}/task"
     f"?due_date_lt_or_eq={today_ms}"
     f"&statuses[]=nowe zapytanie&statuses[]=w trakcie&statuses[]=wycena wysłana"
     f"&include_closed=false",
     "-H", f"Authorization: {token}"],
    capture_output=True, text=True
)
tasks = json.loads(r.stdout).get("tasks", [])
for t in tasks:
    cfs = {cf["name"]: cf.get("value") for cf in t.get("custom_fields", [])}
    print(t["id"], t["name"][:40], cfs.get("telegram_topic_id"))
```

---

## Step 6 — Verify end-to-end

Three tests in order:

1. **Read** — list tasks and confirm the agent can parse IDs and statuses
2. **Create** — create a test task; check it appears in ClickUp with correct status and custom fields
3. **Update** — change status on the test task; verify the status name landed exactly right (look in ClickUp, not just API response)

If the status did not change despite a 200 response → the status name is wrong. Go back to Step 2 and print `repr(s["status"])` for each status.

---

## Gotchas (in order of how badly they cost)

1. **`due_date` must be milliseconds, not seconds.**
   `int(time.time())` = seconds = year 1970 in ClickUp. Always `int(time.time() * 1000)`. API returns 200 regardless — the task gets due_date 1970, which shows as overdue immediately. Silent fail.

2. **Status names must match EXACTLY — case, spaces, Polish characters.**
   `"w trakcie"` ≠ `"W trakcie"` ≠ `"w-trakcie"`. ClickUp returns HTTP 200 even when the status is unrecognized — the field simply does not update. Always run the Step 2 status check and copy the output literally.

3. **Custom fields cannot be set in the task creation POST.**
   The `custom_fields` array in the POST body is silently ignored in API v2. Always do a separate PUT/POST to `/task/{id}/field/{field_id}` for each custom field after creating the task.

4. **Tags must exist in the workspace before use.**
   If you set `"tags": ["new-tag"]` and that tag doesn't exist in the workspace, ClickUp silently drops it. Create tags manually in ClickUp first, then use them from the API.

5. **Assignees require numeric user IDs, not emails or usernames.**
   `"assignees": ["jan@example.com"]` silently fails. Collect user IDs once with the GET /member endpoint (Step 2) and hardcode them in SOUL.md.

6. **Filtering by status in the list endpoint requires `statuses[]` (array syntax).**
   `?status=w+trakcie` does not work. The correct form: `?statuses[]=w%20trakcie&statuses[]=nowe%20zapytanie`.

7. **`GET /task/{id}` vs `GET /list/{id}/task` — different fields returned.**
   The list endpoint returns a paginated array with `page=` and can omit custom fields unless `&include_closed=true`. The task endpoint returns the full record. When reading custom fields, always use the task endpoint: `GET /task/{task_id}?include_subtasks=true`.

8. **Subtasks are created in the same List as the parent** — they are NOT nested under the parent in the API unless you set `parent`. If you forget `"parent": task_id`, you get a top-level task that looks identical to the parent.

9. **API v2 description is plain text only.**
   Line breaks work (`\n`), Markdown does not render. If you need formatted notes, use comments instead.

---

## Quick Q&A flow for the setup agent

When the user says "I want the agent to manage tasks in ClickUp":

1. "Czy masz już workspace w ClickUp i listę gdzie trafiają zadania — czy budujemy od zera?"
2. "Wygeneruj Personal API Token — pokażę gdzie." (Apps tab, Step 1)
3. "Wklej token — zweryfikuję czy działa."
4. Run Step 2: collect team_id, list_id, user IDs, status names (print with repr!), custom field IDs.
5. "Jakie custom pola chcesz żeby agent wypełniał przy tworzeniu taska? Np. temat maila, topic Telegram, ID leada w CRM."
6. Add `CLICKUP_API_TOKEN` to `.env`, recreate container (Step 3).
7. Generate SOUL.md section with actual IDs (Step 4) — do not use placeholders.
8. Build skills using the patterns from Step 5 for: task creation, status update, morning briefing listing, subtask creation.
9. Run verification tests (Step 6).

If status update silently fails → Step 2 status check, repr output. Always first gotcha to rule out.
