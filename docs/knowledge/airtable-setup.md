# Airtable Setup — agent reads & writes Airtable natively

How to give the OpenClaw agent CRUD access to an Airtable base via MCP, including PAT scopes, the right way to register the MCP server, and the traps that cost us extra time (mostly: env reload and config location).

The agent should be able to: list/search records, read schema, create new rows, update fields, with explicit human approval for any write.

---

## Step 1 — Generate the Airtable Personal Access Token (PAT)

User flow:

1. airtable.com → click avatar → **Developer Hub** → **Personal access tokens** → **Create new token**
2. Name: `OpenClaw <agent-name>` (e.g. `OpenClaw Anatol`)
3. **Scopes** — pick the minimum required:
   - `data.records:read` — read rows
   - `data.records:write` — create/update/delete rows
   - `schema.bases:read` — let the agent introspect tables/fields (highly recommended — without it the agent has to be told every field name)
4. **Access** — pick **only the one base** that the agent should reach. **Do not** select "All workspaces". Least privilege.
5. Copy the token starting with `pat...` (shown once).

**Validate before saving** — saves debugging later:

```bash
curl -s "https://api.airtable.com/v0/meta/bases/<BASE_ID>/tables" \
  -H "Authorization: Bearer <PAT>" | head -c 200
```

Response should start with `{"tables":[...`. A 401 means wrong key, 403 means wrong scopes, 404 means wrong base ID.

---

## Step 2 — Inspect the existing schema (before writing config)

Always do this. Real bases have legacy mess: duplicate single-select options, fields that should be `number` but are `singleSelect`, columns named with typos. The agent will trip on all of these. Better to know up front.

```bash
curl -s "https://api.airtable.com/v0/meta/bases/<BASE_ID>/tables" \
  -H "Authorization: Bearer <PAT>" | python3 -c '
import json,sys
d=json.load(sys.stdin)
for t in d["tables"]:
  print(f"\n{t[\"name\"]} ({t[\"id\"]}) — {len(t[\"fields\"])} fields")
  for f in t["fields"]:
    typ=f["type"]
    extra=""
    if typ in ("singleSelect","multipleSelects"):
      extra=f" ({len(f.get(\"options\",{}).get(\"choices\",[]))} options)"
    elif typ in ("multipleRecordLinks","singleRecordLink"):
      extra=f" -> {f.get(\"options\",{}).get(\"linkedTableId\",\"?\")}"
    print(f"  - {f[\"name\"]} [{typ}]{extra}")'
```

Look for:

- **Duplicate single-select options** like `klub` / `KLUB`, `wesele` / `Wesele`, `plener ` / `plener` (trailing space). Ask the user to consolidate before going live — the agent will pick wrong values otherwise.
- **`singleSelect` instead of `number`** on amount fields ("Kwota") — breaks numeric filtering.
- **Single-select fields with many options** (e.g. 74 cities) — agent must pick from the list, cannot freely add. Document the rule in SOUL.md.
- **Linked-record fields** — agent must pass `recXXX` IDs, not strings.
- **Fields that pipelines own** — e.g. `Draft_Queue` / `Draft_Corections` populated by n8n. Mark these read-only for the agent.

Save the inspection — paste it into the SOUL.md update later.

---

## Step 3 — Add credentials to env on the VPS

The credentials live in **`/docker/<project>/.env`** on the VPS — that's the file `docker-compose.yml` reads via `env_file: - .env`. The Hostinger panel "Environment variables" UI for installed apps is just a view onto this same file (whatever you add there gets written to it). So you can either edit the panel OR SSH and edit the file — they're the same place.

```bash
ssh root@<VPS> "echo 'AIRTABLE_API_KEY=<PAT>' >> /docker/<project>/.env"
ssh root@<VPS> "echo 'AIRTABLE_BASE_ID=<BASE_ID>' >> /docker/<project>/.env"
```

(Always `cp .env .env.bak.<timestamp>` first.)

### Critical: `docker compose restart` does NOT reload `.env`

This burned us. `docker compose restart` only restarts the process inside the existing container — env vars are baked in at container creation. To pick up the new `.env`:

```bash
cd /docker/<project> && docker compose up -d <service>
# This RECREATES the container with the new env file
```

Verify the env is actually visible inside the container before continuing:

```bash
docker exec <container> sh -c 'env | grep AIRTABLE'
# expected: two lines, AIRTABLE_API_KEY=... and AIRTABLE_BASE_ID=...
```

---

## Step 4 — Install `mcporter` and `airtable-mcp-server` in the container

OpenClaw ships with the `mcporter` skill (a CLI that calls any MCP server) but the binary itself is not pre-installed. We add both at once:

```bash
docker exec <container> sh -c 'npm install -g mcporter airtable-mcp-server'
```

Verify:

```bash
docker exec <container> sh -c 'which mcporter; mcporter --version; which airtable-mcp-server'
# expected: /data/.npm-global/bin/mcporter, version 0.9.x, /data/.npm-global/bin/airtable-mcp-server
```

The `airtable-mcp-server` package is the one published by `domdomegg` on npm (1.13+ at time of writing) — it does CRUD + schema introspection (`describe_table`, `list_bases`).

---

## Step 5 — Register the MCP server (use the SYSTEM config location)

`mcporter config add` defaults to a **project config** (relative to CWD: `./config/mcporter.json`). That's the trap — the OpenClaw agent runs from `/data/.openclaw/workspace`, so a project config you wrote from `/root` is invisible to it.

**Use the system location** (`/data/.mcporter/mcporter.json` — i.e. `$HOME/.mcporter/mcporter.json`). The agent reads this regardless of CWD.

Add the server, then move/copy the config to the system location:

```bash
docker exec <container> sh -c '
  mcporter config add airtable \
    --command airtable-mcp-server \
    --env AIRTABLE_API_KEY=$AIRTABLE_API_KEY
  mkdir -p /data/.mcporter
  cp /data/config/mcporter.json /data/.mcporter/mcporter.json
'
```

Verify the agent will see it (run from workspace, the same CWD the agent uses):

```bash
docker exec <container> sh -c '
  cd /data/.openclaw/workspace && mcporter list
'
# expected: airtable (16 tools, 0.4s)  ✔ Listed 1 server (1 healthy).
```

If it says `0 tools` or "missing" — env vars aren't reaching the spawned server. Re-check Step 3.

---

## Step 6 — Restart the container so OpenClaw scans the new skill

OpenClaw scans available skills at startup. The `mcporter` skill has `requires.bins: ["mcporter"]` — if the binary wasn't in PATH at startup, the skill won't be in `<available_skills>` for the agent. After installing in Step 4 you must restart:

```bash
cd /docker/<project> && docker compose restart <service>
```

(`restart` is fine here — we already loaded env in Step 3 with `up -d`; this restart is just to re-scan skills.)

Verify:

```bash
docker exec <container> sh -c 'openclaw skills list 2>&1 | grep -i mcporter'
# expected: │ ✓ ready  │ 📦 mcporter ...
```

---

## Step 7 — Update `SOUL.md` with the base map and write rules

Without this, the agent will ask the user for the API key and Base ID every session — the env is invisible to it conceptually unless SOUL.md says "you have it". Append to `/data/.openclaw/workspace/SOUL.md`:

```markdown
## Airtable — operational knowledge base

You have native access to Airtable via `mcporter` (16 MCP tools). Credentials are in env, do not ask the user for them.

**baseId:** `<BASE_ID>`

**Tables (read & write):**

| Table | tableId | Role |
|---|---|---|
| <TABLE_1> | `<tbl...>` | <description, key fields> |
| <TABLE_2> | `<tbl...>` | <description, key fields> |

**Read-only tables (do not write):**

| Table | tableId | Why |
|---|---|---|
| <PIPELINE_TABLE> | `<tbl...>` | n8n owns this |

### Common patterns

(paste 2-3 actual `mcporter call airtable.list_records ...` examples
that match the user's typical questions, e.g. "is artist X available on date Y")

### Write rules (CRITICAL)

Every CREATE/UPDATE/DELETE requires explicit user approval. Pattern:

1. Show the proposal: "I want to add <field>=<value>, <field>=<value>. Approve?"
2. Wait for confirmation (`tak` / `zatwierdzam` / `yes` / etc.)
3. Execute the write
4. Confirm with the new record ID

Never write a record with partial data "I'll fill the rest later".

### Schema gotchas

(list ALL the gotchas you found in Step 2 — single-select rules,
linked-field IDs, type mismatches. Be specific. Future-you will
forget which fields have 74 options vs 3.)
```

Generic templates don't help — paste actual table IDs, actual field names, actual examples. The agent will follow them literally.

---

## Step 8 — User starts a NEW session in their channel

Skills are snapshotted **per session** at session start. Old sessions have a frozen `<available_skills>` list from before mcporter existed — the agent in those sessions genuinely cannot see Airtable, even though everything is configured.

Tell the user explicitly:

> "W Telegramie napisz `/new` zanim spróbujesz pierwszej operacji na Airtable — stara sesja ma zamrożoną listę narzędzi sprzed instalacji."

Without `/new` the agent will say "I don't have an MCP server configured" and ask the user for credentials. That's not a config bug — it's a session snapshot.

---

## Step 9 — Verify end-to-end

In the user's channel, three tests in order:

1. **List**: "ile mamy artystów w `<TABLE>`?" — agent should call `airtable.list_records` with `maxRecords` and report a count
2. **Search**: "podaj wszystkich z gatunku jazz" — agent uses `airtable.search_records`
3. **Create with approval**: "dodaj artystę X z ceną 8000" — agent should propose, wait for `tak`, then create

If test 1 fails with "I need credentials" → user is in an old session, run `/new`.
If test 1 succeeds but test 3 skips approval → SOUL.md write rule isn't strong enough; rephrase it with CRITICAL emphasis.

---

## Gotchas we hit (in order of how badly they cost us)

1. **`docker compose restart` ignores new `.env`.**
   First add to `.env`, then `docker compose up -d <service>` (recreate). After that you can use `restart` for trivial reloads. Burned ~10 min.

2. **mcporter project config (`./config/mcporter.json`) is CWD-relative.**
   Adding it from `/root` means the agent (CWD=`/data/.openclaw/workspace`) doesn't see it. Always copy to the system location `/data/.mcporter/mcporter.json` after `mcporter config add`. Burned ~15 min — agent kept saying "no MCP server configured" while we had a perfectly registered one.

3. **Skill snapshot frozen per session.**
   New skills require a new session (`/new`) to enter the agent's `<available_skills>`. Restart of the container is not enough. Burned ~5 min.

4. **Container-restart-without-skill-rescan.**
   The `mcporter` skill is detected at gateway startup (`requires.bins: ["mcporter"]`). If you install the binary AFTER container start, the skill won't be loaded until the next restart. Order matters: install bins → restart container → skills are scanned → start session. Burned ~5 min.

5. **Hostinger panel "Environment Variables" is just a view onto `.env`.**
   It's not a separate mechanism. Editing the file directly via SSH and editing in the panel both write to the same `/docker/<project>/.env`. Earlier docs that say "add via Hostinger panel" are technically right but misleading — the panel is not authoritative, the file is.

6. **Single-select fields with legacy duplicates are a footgun.**
   If "Rodzaj imprezy" has both `KLUB` and `klub`, agent will pick whichever Airtable returns first — usually wrong. Always inspect schema before writes (Step 2) and ask user to consolidate duplicates.

7. **Without `schema.bases:read`, agent cannot introspect.**
   You'd save tokens by skipping it but then have to enumerate every field name in SOUL.md by hand and update SOUL.md whenever the user adds a column. Worth the scope.

8. **PAT scope wider than one base = security smell.**
   If user has multiple Airtable bases (e.g. personal finance, client work), do not let the agent see all of them. One PAT per agent per base.

---

## What logs to read when "agent says no MCP configured"

1. **From the agent's POV** — does the agent's session even have the skill?
   ```bash
   docker exec <container> sh -c '
     grep -oE "<name>[a-z-]+</name>" /data/.openclaw/agents/main/sessions/sessions.json | sort -u
   '
   ```
   Look for `<name>mcporter</name>`. If absent → user needs `/new` (Step 8).

2. **From mcporter's POV** — does it find the config?
   ```bash
   docker exec <container> sh -c '
     cd /data/.openclaw/workspace && mcporter config list
   '
   ```
   If it shows nothing → config is in project location, copy to system location (Step 5).

3. **From the server's POV** — does it spawn cleanly?
   ```bash
   docker exec <container> sh -c '
     cd /data/.openclaw/workspace && mcporter list airtable --schema 2>&1 | head -5
   '
   ```
   If it errors with auth → env not reaching server, re-check Steps 3–5.

---

## Quick Q&A flow for the setup agent

When the user says "I want the agent to use Airtable":

1. "Czy masz już bazę Airtable czy budujemy razem?"
2. "Wygeneruj PAT — pokażę gdzie." (link instructions from Step 1, scopes: `data.records:read`, `data.records:write`, `schema.bases:read`, access ograniczony do JEDNEJ bazy)
3. "Wklej PAT i URL bazy."
4. Validate token with curl (Step 1) and inspect schema (Step 2) — paste schema summary back to user, flag any gotchas (duplicate single-selects, type mismatches), ask if they want to clean those up first.
5. Add to `.env`, recreate container (Step 3).
6. Install `mcporter` + `airtable-mcp-server` (Step 4), copy config to system location (Step 5), restart (Step 6).
7. Append SOUL.md section with actual table IDs, field names, and 2-3 concrete example calls (Step 7) — don't use generic templates.
8. Tell user to run `/new` in their channel before testing (Step 8).
9. Run the three verification tests (Step 9).

If anything fails, walk down the gotchas list 1→8 in order.
