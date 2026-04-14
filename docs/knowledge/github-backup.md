# GitHub Backup for OpenClaw Config

## Why a separate GitHub account

The backup contains tokens, credentials, and configuration. The agent needs a GitHub token with repo access. If that token leaks, an attacker gets access to every repository on the account.

**Create a separate GitHub account just for the agent.** Even if something goes wrong — your own repositories, projects, and code are safe.

---

## Step 1: Create a dedicated GitHub account

1. Create a new email for the agent (e.g. `my-agent-backup@gmail.com`)
2. Create a GitHub account on that email (e.g. `my-agent-openclaw`)
3. On the agent account, create a new **private** repository (e.g. `openclaw-backup`)

---

## Step 2: Generate a fine-grained access token

On the **agent GitHub account**:

1. Settings → Developer settings → Personal access tokens → **Fine-grained tokens** → Generate new token
2. Token name: e.g. `openclaw-vps`
3. Resource owner: the agent account
4. Expiration: No expiration (or per your preference)
5. Repository access: **All repositories**
6. Permissions → Add permissions:
   - **Administration**: Read and write
   - **Contents**: Read and write
   - (Metadata: Read-only — added automatically)
7. Click **Generate token** — copy the token and the repo URL

---

## Step 3: Configure git credentials in Docker

> ⚠️ **Do not send the GitHub token via Telegram or the dashboard chat.** Messages can be logged. Do this in the VPS terminal.

```bash
docker exec $(docker ps -q) git config --global credential.helper 'store --file /data/.git-credentials'
docker exec $(docker ps -q) bash -c 'echo "https://YOUR_TOKEN@github.com" > /data/.git-credentials'
docker exec $(docker ps -q) chmod 600 /data/.git-credentials
```

Replace `YOUR_TOKEN` with the token from Step 2.

---

## Step 4: Configure automated backup via agent

Credentials are set. Now send to agent via Telegram or dashboard:

```
Configure automatic backup of my OpenClaw config to GitHub:

1. Set git user.name to "OpenClaw Backup" and user.email to "backup@openclaw"
2. Initialize a git repo in /data/.openclaw/
3. Add remote: YOUR_REPO_URL
4. Add .gitignore to skip large/temporary files (cache, sandbox containers)
5. Make the first commit and push all config files
6. Set up a cron job that runs daily at 3:00 to git add, commit, and push

Config files are in /data/.openclaw/
```

Replace `YOUR_REPO_URL` with the repo URL from Step 2.

---

## Step 5: Verify

1. **Check the cron job**: Open the OpenClaw dashboard → Cron Jobs section. You should see a "Daily GitHub Backup" job with schedule `0 3 * * *`.

2. **Check the first push**: Open the repo on the agent's GitHub account and confirm files are there.

---

## What gets backed up

The agent's git setup should include these files:

```
/data/.openclaw/
├── openclaw.json         ← main config (without hardcoded secrets)
├── workspace/
│   ├── SOUL.md           ← agent identity
│   ├── MEMORY.md         ← agent memory
│   └── memory/           ← extended memory files
```

The `.gitignore` should exclude:
- Cache directories
- Sandbox container files
- Any temporary runtime files

---

## Security notes

- The backup repo must be **private**
- The GitHub token is stored in `/data/.git-credentials` with `chmod 600`
- `openclaw.json` contains `"${ENV_VAR_NAME}"` references — no actual API keys in the backup
- Never commit `/etc/openclaw.env` (this file has the actual keys)

---

## Credential rotation

| Credential | Frequency | How |
|-----------|-----------|-----|
| GitHub token | Every 90 days | Revoke in GitHub → generate new → update `.git-credentials` |
| Gateway auth token | Every 90 days | `openclaw doctor --generate-gateway-token` |
| Model API keys | Every 90 days | Provider dashboard + update Hostinger Environment |
| Telegram bot token | Every 180 days | @BotFather → `/revoke` → new token → update Hostinger Environment |
| After any incident | Immediately | All of the above |
| After someone with access leaves | Immediately | All shared credentials |
