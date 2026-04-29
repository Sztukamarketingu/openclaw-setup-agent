# Google Workspace Setup — agent reads & sends Gmail, Drive, Calendar natively

How to give the OpenClaw agent native access to one or more Google Workspace accounts (Gmail, Drive, Calendar, Docs, Sheets) via the bundled `gog` skill (gogcli). Covers OAuth setup for headless servers, multi-account/multi-project handling, and the traps that cost real time (mostly: browser flow on a VPS doesn't work, keyring needs no-TTY password, binary doesn't survive container recreate).

The agent should be able to: list/search emails, read full message bodies, search Drive files, check Calendar — all autonomously. Write actions (send, delete) require explicit human approval enforced via SOUL.md.

---

## Step 1 — Decide on the Google Cloud project layout

Two reasonable patterns:

**Pattern A — One project, multiple accounts (simpler):**
One Cloud project, one OAuth client. Each user account in your Workspace authorizes the same client. Use this when all accounts are owned by the same organization and you control the project.

**Pattern B — One OAuth client per project (multi-tenant):**
Different accounts authorize different OAuth clients in different projects. Use this when accounts live in different Workspace orgs, or when an account is already linked to an existing Cloud project (e.g. a previous Google Chat integration) and you want to keep that project as the OAuth surface for that account.

`gog` supports both via the `--client <name>` flag. Multi-client adds one register step but otherwise costs nothing.

**Decision rule:** if user says "ten mail jest na innym projekcie" — go with Pattern B without arguing.

---

## Step 2 — Create the OAuth client in Google Cloud Console

For each project (1 in Pattern A, N in Pattern B):

1. **console.cloud.google.com** → top bar → New Project (or pick existing)
2. **APIs & Services** → search and **Enable**: `Gmail API`, `Google Drive API`, `Google Calendar API`. Add `Docs API`, `Sheets API` if you want write access to docs/sheets.
3. **OAuth consent screen**:
   - User Type: **Internal** (only works if your project lives in a Google Workspace org — if you're on a personal account, External + add test users).
   - App name: `Anatol OpenClaw` (or whatever your agent is called)
   - Scopes — **leave empty here**. `gog` requests scopes per-auth via the URL; the consent screen list mostly affects the verification badge and what's shown to users. For Internal apps, missing scopes here don't block authorization.
4. **Credentials** → **Create Credentials** → **OAuth 2.0 Client ID**:
   - Application type: **Desktop app** ("Aplikacja komputerowa")
   - Name: `OpenClaw gog` (or `OpenClaw gog — <project name>`)
   - **No redirect URI fields** — Desktop apps don't need them. Google allows any `127.0.0.1:<port>` automatically for installed apps.
5. **Download JSON** → save as `client_secret_<id>.json`. The file looks like:

   ```json
   {"installed":{"client_id":"<id>","project_id":"<project>","client_secret":"GOCSPX-...","redirect_uris":["http://localhost"]}}
   ```

   The `installed` key (not `web`) confirms it's a Desktop app.

---

## Step 3 — Install the `gog` binary in the container

The OpenClaw skill is named `gog` (not `gogcli`). The skill is bundled but the actual binary `gog` is **not** pre-installed. On Linux, get it from GitHub Releases (the website only shows brew install — easy to miss the Linux binaries):

```bash
ssh root@<VPS> "
  cd /tmp
  curl -sL https://github.com/steipete/gogcli/releases/download/v0.14.0/gogcli_0.14.0_linux_amd64.tar.gz -o gogcli.tar.gz
  tar -xzf gogcli.tar.gz
  # binary 'gog' extracted to /tmp/gog
"
```

Check VPS architecture first if unsure: `ssh root@<VPS> "uname -m"` — `x86_64` = `linux_amd64`, `aarch64` = `linux_arm64`.

**Critical: where to put the binary.** Two locations:

```bash
ssh root@<VPS> "
  # 1. Persistent copy in the volume (survives recreate)
  mkdir -p /docker/<project>/data/.bin
  cp /tmp/gog /docker/<project>/data/.bin/gog

  # 2. Active copy in container's PATH (where the skill expects it)
  docker cp /tmp/gog <container>:/usr/local/bin/gog
  docker exec <container> chmod +x /usr/local/bin/gog
"
```

Why both: `/usr/local/bin/gog` is what the `gog` skill checks (looks for `gog` in PATH). But `/usr/local/bin/` is **not** part of the Docker volume — `docker compose up -d` (recreate) wipes it. The persistent copy in `/data/.bin/gog` is your safety net (see "After every recreate" below).

Verify:

```bash
docker exec <container> sh -c 'gog --version'
# expected: v0.14.0 (...)

docker exec <container> sh -c 'openclaw skills info gog | head -5'
# expected: 🎮 gog ✓ Ready
```

---

## Step 4 — Set the keyring password in `.env` BEFORE auth

`gog` stores OAuth refresh tokens in an encrypted keyring file. The default keyring backend prompts interactively for a password — which fails in a non-TTY container. You'll see:

> `OAuth completed, but saving the refresh token failed: store token: store file keyring item: no TTY available for keyring file backend password prompt; set GOG_KEYRING_PASSWORD`

The fix is two parts:

```bash
# 1. Set a password in the .env (any string — it just encrypts the local file)
ssh root@<VPS> "echo 'GOG_KEYRING_PASSWORD=<some-strong-string>' >> /docker/<project>/.env"

# 2. Recreate the container so the env is loaded (not 'restart' — see Airtable doc Step 3)
ssh root@<VPS> "cd /docker/<project> && docker compose up -d"

# 3. After recreate, restore /usr/local/bin/gog from the volume backup
ssh root@<VPS> "
  docker cp /docker/<project>/data/.bin/gog <container>:/usr/local/bin/gog
  docker exec <container> chmod +x /usr/local/bin/gog
"

# 4. Tell gog to use the file keyring backend (one-time)
ssh root@<VPS> "docker exec <container> gog auth keyring file"
# expected: keyring_backend  file
```

Verify the env reached the container:

```bash
docker exec <container> sh -c 'env | grep GOG_KEYRING_PASSWORD'
# expected: GOG_KEYRING_PASSWORD=...
```

Without this, **every** `gog auth ...` command will run OAuth fine and then fail on save. Frustrating because the browser-side worked but no token was persisted.

---

## Step 5 — Register OAuth credentials in gog

Upload each `client_secret_*.json` to the VPS volume, then register it with `gog`:

```bash
# Upload to VPS
scp client_secret_<id>.json root@<VPS>:/docker/<project>/data/.openclaw/gogcli_<label>_credentials.json

# Pattern A (one client, called 'default'):
ssh root@<VPS> "docker exec <container> gog auth credentials set /data/.openclaw/gogcli_<label>_credentials.json"
# expected: client  default

# Pattern B (multi-client — give each a name):
ssh root@<VPS> "docker exec <container> gog auth credentials set /data/.openclaw/gogcli_tomek_credentials.json --client impresariat"
# expected: client  impresariat
```

Verify:

```bash
docker exec <container> gog auth credentials list
# expected:
# CLIENT       PATH                                              DOMAINS
# default      /data/.config/gogcli/credentials.json             ...
# impresariat  /data/.config/gogcli/credentials-impresariat.json ...
```

---

## Step 6 — Authorize each account (server-friendly OAuth)

**Do not use the default browser flow on a VPS.** It opens a random port on the container's loopback, has a short internal timeout, and even with an SSH tunnel the moving target makes it unreliable. We tried — burned ~30 minutes.

Use the `--remote --step 1/2` flow (it's literally designed for headless servers):

```bash
# Step 1 — generate the auth URL (no waiting, no listener)
ssh root@<VPS> "docker exec <container> gog auth add <user@domain> \
  --client <client-name> \
  --remote --step 1 \
  --services gmail,drive,calendar,docs,sheets"
```

This prints `auth_url	https://accounts.google.com/o/oauth2/auth?...` and exits.

**Send that URL to the user.** They open it in their normal browser, log in as the target account, click through the consent. After consent, the browser tries to redirect to `http://127.0.0.1:<random-port>/oauth2/callback?code=...` and fails with "site can't be reached" — **this is expected**. The user copies the **full URL from the address bar** (it has the auth code) and sends it back.

```bash
# Step 2 — exchange the code (paste the full callback URL the user copied)
ssh root@<VPS> "docker exec <container> gog auth add <user@domain> \
  --client <client-name> \
  --remote --step 2 \
  --services gmail,drive,calendar,docs,sheets \
  --auth-url 'http://127.0.0.1:<port>/oauth2/callback?state=...&code=...&scope=...'"
# expected: email <addr>  services calendar,docs,drive,gmail,sheets  client <name>
```

Repeat for each account. Tokens are stored encrypted at `/data/.config/gogcli/keyring/`.

### Why scope-narrow with `--services`

`gog auth add` without `--services` requests **everything** — including `classroom`, `ads`, `chat`, `contacts`, `forms`, `appscript`. For a concert agency assistant that's absurd overreach. Always pass `--services gmail,drive,calendar,docs,sheets` (or the subset you actually need). Available services: `gmail,calendar,chat,classroom,drive,docs,slides,contacts,tasks,sheets,people,forms,appscript,ads`.

Note: Google's response URL may show extra scopes you didn't request — that's `include_granted_scopes=true` returning previously-granted scopes for this client. Doesn't matter; the token only works for what you asked for in `--services`.

---

## Step 7 — Verify auth and token storage

```bash
docker exec <container> gog auth list
# expected (one row per account):
# joanna.k@<domain>      default      calendar,docs,drive,gmail,sheets   <iso-timestamp>  oauth
# t.szwecki@<domain>     impresariat  calendar,docs,drive,gmail,sheets   <iso-timestamp>  oauth
```

End-to-end test from inside the container:

```bash
docker exec <container> gog -a <user@domain> --client <client> gmail messages list 'in:inbox' --max 3
# expected: TSV table with ID, THREAD, DATE, FROM, SUBJECT, LABELS

docker exec <container> gog -a <user@domain> --client <client> drive search "name contains 'umowa'" --max 5
# expected: list of files matching the query
```

If `gmail messages list` errors with `unknown flag --max-results`, the right flag is `--max` (gog uses short form). Use `gog gmail messages list --help` to discover real flag names — they don't always match Gmail API names.

---

## Step 8 — Decide on `tools.exec.security`

This is where most setups stall. The exec config in `openclaw.json` controls how shell commands (which is what `gog` calls are, under the hood) get authorized:

```jsonc
"tools": {
  "exec": {
    "security": "allowlist",
    "ask": "on-miss",
    ...
  }
}
```

**The trap:** `security: "allowlist"` with no actual allowlist field populated = **every** command misses, so the agent prompts for approval **every time**. The user clicks "approve" and the agent immediately needs another approval for the next command. We hit this — user got 10+ prompts in a single conversation before we caught it.

**Two reasonable resolutions:**

**Option A — `security: "full"` (chosen for personal-assistant setups):**

```jsonc
"tools": {
  "exec": {
    "security": "full",
    "notifyOnExit": true
  }
}
```

No approval prompts, full shell access. Safety is enforced via SOUL.md rules (Step 9 below) — the agent must ask before destructive actions. Trust model: container is isolated, OAuth scopes are minimal, agent is your assistant not a stranger.

**Option B — `security: "deny"` with explicit per-command approvals:**

For environments where you want belt-and-suspenders. Documented in `docs/knowledge/security.md`. More friction, fewer footguns.

For impresariat-style assistants we used Option A. Apply via hot-reload (no restart needed):

```bash
ssh root@<VPS> "
  python3 -c \"
import json
p='/docker/<project>/data/.openclaw/openclaw.json'
c=json.load(open(p))
c['tools']['exec']['security']='full'
c['tools']['exec'].pop('ask',None)
json.dump(c,open(p,'w'),indent=2)
\"
"
# Watch the gateway logs — you should see [reload] config change applied
ssh root@<VPS> "docker logs <container> --tail 5"
```

---

## Step 9 — Update `SOUL.md` with gog instructions

The agent doesn't know about `gog` until SOUL.md tells it. Without the section, it'll either ignore the skill or hallucinate commands. Append to `/data/.openclaw/workspace/SOUL.md`:

```markdown
## Google Workspace via `gog` (Gmail / Drive / Calendar / Docs / Sheets)

You have native access to <N> Google accounts via `gog` CLI. Credentials and keyring password are in env (`GOG_KEYRING_PASSWORD`); do not ask the user for them.

**Accounts — different flags:**

| Account | `--client` flag | `-a` flag |
|---|---|---|
| <Name 1> | `default` (or omit) | `<email1>@domain` |
| <Name 2> | `<other-client-name>` | `<email2>@domain` |

### Common commands

```bash
# List 5 latest inbox messages for <Name 1>
gog -a <email1>@domain gmail messages list 'in:inbox' --max 5

# Search history with a specific contact (across both directions)
gog -a <email2>@domain --client <client> gmail messages list 'from:contact@x OR to:contact@x' --max 20

# Full message body (after listing — use the ID)
gog -a <email1>@domain gmail messages get <messageId> --include-body

# Search Drive for a file by name
gog -a <email2>@domain --client <client> drive search "name contains 'umowa'" --max 10

# Calendar events for a date range
gog -a <email1>@domain calendar events list --time-min '2026-04-30T00:00:00Z' --time-max '2026-04-30T23:59:59Z'
```

If unsure of a flag, run `gog <area> --help` (e.g. `gog gmail --help`).

### Account selection (which inbox to use)

- **React contextually:** if a mail came in to <Name 1>, reply from <Name 1>. If to <Name 2>, reply from <Name 2>.
- **If unclear:** ask explicitly before action — "Send from your inbox or from <Name 2>'s?"
- **For new outbound (not replies):** ask the user which account to use as sender.

### Write rules (CRITICAL)

Every action that changes state in Google Workspace requires **explicit human approval**:

1. **Sending email** (`gog gmail send`, `gog gmail drafts send`) → show full draft in the user's channel first, wait for confirmation
2. **Deleting / overwriting Drive files** → always ask
3. **Modifying Calendar events** → ask unless creation from a clear instruction

**Read & search** (gmail messages list/get, drive search, calendar list) — proceed freely, do not ask.

### Security

- Never log message bodies to debug logs
- Never quote messages from outside the team to outside parties
- Treat attachments (CVs, invoices, contracts) as confidential — download only when needed
- Never recite or log `GOG_KEYRING_PASSWORD`
```

Use **actual emails, actual client names** in the table — generic placeholders cause the agent to hallucinate or ask the user to confirm what's already configured.

---

## Step 10 — User runs `/new` and verifies

Same skill-snapshot rule as Airtable: existing sessions don't see the new `gog` skill. Tell the user explicitly:

> "W Telegramie napisz `/new` zanim spróbujesz pierwszej operacji na Gmail/Drive — stara sesja ma zamrożoną listę narzędzi sprzed instalacji `gog`."

Three verification tests in the user's channel:

1. **Read**: "pokaż mi 3 ostatnie maile <Account 1>" — agent calls `gog ... gmail messages list 'in:inbox' --max 3` and shows results
2. **Cross-account**: "ostatnie 3 maile <Account 2>" — agent uses the right `-a` and `--client` flags
3. **Write with approval**: "wyślij krótkie potwierdzenie do <contact>" — agent should propose draft, wait for `ok`, then send. If it sends without asking → SOUL.md write rule isn't strong enough; rephrase with **CRITICAL** emphasis.

---

## After every `docker compose up -d` (recreate)

`/usr/local/bin/gog` is wiped on container recreate. Re-copy from the persistent volume:

```bash
ssh root@<VPS> "
  docker cp /docker/<project>/data/.bin/gog <container>:/usr/local/bin/gog
  docker exec <container> chmod +x /usr/local/bin/gog
"
```

Tokens at `/data/.config/gogcli/` survive (volume-mounted). Just the binary needs replacing.

A more robust fix is to bake `gog` into a custom image, or add a `command:` startup script in `docker-compose.yml` that re-copies on start — both are out of scope here but worth doing if you recreate often.

---

## Gotchas we hit (in order of how badly they cost us)

1. **Browser-based OAuth on a VPS does not work.** `gog auth add <email>` opens a random port on the container's loopback, has an internal ~60s timeout, and you can't reliably tunnel a random port. Skip directly to `--remote --step 1/2`. Burned ~30 min.

2. **Keyring needs `GOG_KEYRING_PASSWORD` BEFORE first auth.** Without it, OAuth flow completes (user clicked through Google) but the token can't save (no TTY for keyring password prompt). You then have to redo OAuth from scratch. Set it in `.env` first, recreate the container, run `gog auth keyring file`, **then** start authorizing. Burned ~10 min.

3. **`/usr/local/bin/gog` does not survive `docker compose up -d`.** The recreate wipes it. Always keep a persistent copy in `/data/.bin/gog` and copy it back after every recreate. The first time we recreated to load `GOG_KEYRING_PASSWORD`, the binary was gone and tokens "didn't work" — actually the binary was missing. Burned ~5 min.

4. **`security: "allowlist"` with no actual allowlist field = approval spam.** The agent prompts every single command. There's no documented field for the allowlist patterns we could find — switch to `security: "full"` and put the safety in SOUL.md. Burned ~10 min once user reported "10x Allow prompts in a row".

5. **Skill is named `gog`, binary is named `gog`, repo/site is `gogcli`.** Easy to confuse. The OpenClaw skill list shows `🎮 gog`. Calling `openclaw skills install gogcli` fails ("not found"). The bundled skill is already there and just needs the binary in PATH.

6. **gog default scopes request EVERYTHING.** classroom, ads, contacts, chat — all included unless you pass `--services`. The user's consent screen will show a long list, which is alarming for them and unnecessary. Always narrow scope with `--services`.

7. **Two accounts on different Cloud projects is fine.** `gog` supports multiple OAuth clients via `--client <name>`. If user says "this email is on a different project" — register a second client (Step 5) and authorize that account against it (Step 6 with `--client <name>`). No project consolidation needed.

8. **Internal vs External consent screen.** If the user is on Google Workspace (custom domain), pick **Internal** — no app verification needed, all Workspace users can authorize. External requires test users until you submit for verification. Always check what they're on before suggesting Internal.

9. **OpenClaw CLI commands `approve`, `exec` may be unavailable.** Some bundled CLI surfaces are excluded by `plugins.allow`. The runtime/agent works regardless — these are only the operator's CLI tools. Don't panic when `openclaw exec --help` errors with "command unavailable" — `tools.exec` still functions for the agent.

---

## What logs to read when "agent says it can't access Gmail"

1. **Skill ready?**
   ```bash
   docker exec <container> openclaw skills info gog
   # expected: 🎮 gog ✓ Ready
   ```
   If `△ needs setup` → binary is missing from PATH (Step 3). Re-copy from `/data/.bin/`.

2. **Tokens saved?**
   ```bash
   docker exec <container> gog auth list
   ```
   If empty → re-run Step 6. If "decryption failed" → `GOG_KEYRING_PASSWORD` differs from when the token was saved (Step 4 was redone with a different password).

3. **Direct call works?**
   ```bash
   docker exec <container> gog -a <email> --client <client> gmail messages list 'in:inbox' --max 1
   ```
   If 401/403 → token expired or revoked, re-run Step 6.
   If `unknown flag` → wrong flag name, use `--help`.

4. **Agent's session has the skill?**
   Same as Airtable — sessions are snapshotted. If session is older than the install, run `/new` (Step 10).

---

## Quick Q&A flow for the setup agent

When the user says "I want the agent to read/send Gmail":

1. "Czy masz Google Workspace z własną domeną, czy używasz osobistego konta Google?" (Workspace → Internal app, simpler. Personal → External + test users, more friction.)
2. "Ile kont chcesz podpiąć i czy są w tym samym projekcie Cloud Console?" — decide Pattern A vs B (Step 1).
3. "Wejdź na console.cloud.google.com — przeprowadzę przez OAuth setup." (Step 2: enable APIs, consent screen Internal, create Desktop OAuth client, download JSON.)
4. **Order matters from here:**
   - Install `gog` binary in `/data/.bin/` and `/usr/local/bin/` (Step 3)
   - Add `GOG_KEYRING_PASSWORD` to `.env`, `docker compose up -d`, restore binary, set keyring backend (Step 4)
   - Register OAuth credentials in gog with `--client` per project (Step 5)
   - Authorize each account via `--remote --step 1/2` (Step 6) — give user the URL, get callback URL back, exchange
5. Verify (Step 7) — check `gog auth list` and one `gmail messages list` call from inside the container.
6. Decide on `tools.exec.security` — recommend `"full"` for personal-assistant setups (Step 8).
7. Append SOUL.md section with **actual emails and actual client names** in the table (Step 9). Don't ship generic placeholders.
8. Tell user to run `/new` in their channel before testing (Step 10).
9. Run the three verification tests (Step 10).

If anything fails, walk down the gotchas list 1→9 in order. Most failures are gotcha #1 (browser flow) or #2 (keyring password timing) — diagnose those before going deeper.
