# Telegram Channel Setup

## Step 1: Create a Telegram bot

1. Open Telegram and search for **@BotFather**
2. Send: `/newbot`
3. Choose a display name (e.g. "Acme Support Bot")
4. Choose a username — must end in `bot` (e.g. `acme_support_bot`)
5. BotFather sends you a token like: `7123456789:AAFxxx...`
6. Save this token as `TELEGRAM_BOT_TOKEN` in your `.env`

**Important:** Keep the token secret. Anyone with the token can control your bot.

---

## Step 2: Find your Telegram user ID (required for allowlist)

You need your numeric Telegram user ID to restrict access. Without an allowlist, **anyone who finds your bot's username can send it commands — including commands that execute on your server.**

**How to get your numeric user ID:**
1. Send any message to your bot
2. Open in browser: `https://api.telegram.org/bot{YOUR_TOKEN}/getUpdates`
3. Find `"from": {"id": 123456789}` — that number is your user ID
4. Save it — you will put it in `allowFrom` in the config

---

## Step 3: Configure Telegram in openclaw.json

OpenClaw reads its config from `/data/.openclaw/openclaw.json` (Docker/VPS setup).

**Add the `channels` section to your `openclaw.json`:**

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "dmPolicy": "allowlist",
      "allowFrom": [YOUR_NUMERIC_USER_ID],
      "groups": {
        "*": {
          "enabled": false
        }
      }
    }
  }
}
```

**Replace `YOUR_NUMERIC_USER_ID`** with your actual numeric ID from Step 2 (e.g. `123456789`).

### Key points

- Use `"${TELEGRAM_BOT_TOKEN}"` — OpenClaw substitutes the value from the environment variable automatically. **Never paste the token directly in the JSON file.** A hardcoded token can leak through backups, screenshots, or prompt injection.
- `dmPolicy: "allowlist"` + `allowFrom` = only listed user IDs can interact with the bot
- `groups: { "*": { enabled: false } }` — disables all group chats; the bot only responds in direct messages
- **OpenClaw detects config file changes automatically — no container or Gateway restart required**

---

## Step 4: Apply config via OpenClaw dashboard

The recommended way to apply this config is through the OpenClaw Control UI (accessible via Tailscale or SSH tunnel):

1. Open the OpenClaw dashboard
2. Send the agent this command:

```
Add to /data/.openclaw/openclaw.json a channels section with this Telegram configuration:
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "dmPolicy": "allowlist",
      "allowFrom": [YOUR_NUMERIC_USER_ID],
      "groups": { "*": { "enabled": false } }
    }
  }
}
```

Or edit the file directly via SSH:

```bash
ssh -i "${VPS_SSH_KEY_PATH}" "${VPS_USERNAME}@${VPS_HOSTNAME}" \
  "cat /data/.openclaw/openclaw.json"
# Read current file, then update it with the channels section
```

---

## Step 5: Verify the connection

Send "hello" to your bot in Telegram. You should receive a response within a few seconds.

To check channel status via CLI:

```bash
openclaw channels status --probe
```

Expected output:
```
telegram: connected (bot: @your_bot_name)
```

---

## Config file format reference

### openclaw.json field names for Telegram

| Field | Value | Notes |
|-------|-------|-------|
| `enabled` | `true` / `false` | Activates the channel |
| `botToken` | `"${TELEGRAM_BOT_TOKEN}"` | Always use env var reference, never the raw token |
| `dmPolicy` | `"allowlist"` / `"open"` / `"closed"` | Use `"allowlist"` for any real deployment |
| `allowFrom` | `[123456789, 987654321]` | Numeric Telegram user IDs |
| `groups["*"].enabled` | `false` | Disables bot in all group chats |
| `agentId` | `"main"` | Routes to a specific agent (optional) |

### Security levels

| dmPolicy | Who can message | Use case |
|----------|----------------|----------|
| `"open"` | Anyone | Never use in production |
| `"allowlist"` | Only listed user IDs | Recommended for all real deployments |
| `"closed"` | Nobody (bot inactive) | Maintenance mode |

---

## Config path note

| Deployment type | Config file location |
|-----------------|----------------------|
| Docker / VPS (recommended) | `/data/.openclaw/openclaw.json` |
| Local / native install | `~/.openclaw/config.json5` |

The `/data/.openclaw/` path is used when OpenClaw runs in a Docker container with a mounted volume. This is the standard Hostinger VPS deployment pattern.

---

## Troubleshooting

### Bot not responding after config change

1. Check that `TELEGRAM_BOT_TOKEN` is set in the VPS environment: `printenv TELEGRAM_BOT_TOKEN`
2. Confirm `enabled: true` in the channels section
3. Verify the JSON is valid — invalid JSON silently breaks the config. Use: `python3 -m json.tool /data/.openclaw/openclaw.json`
4. Check OpenClaw logs: `openclaw logs --tail 30`

### "Unauthorized" error in logs

Token is wrong or expired. In BotFather: `/revoke` → choose your bot → get new token → update `TELEGRAM_BOT_TOKEN` in `/etc/openclaw.env` → the config auto-reloads.

### Message sent but no reply

Your user ID is not in `allowFrom`. Get the correct numeric ID via `getUpdates` (Step 2) and add it to the config. The config reloads automatically.

### Webhook conflict

OpenClaw uses long-polling. If another service previously set a webhook on this bot token, clear it:

```bash
curl "https://api.telegram.org/bot{TOKEN}/deleteWebhook"
```

---

## Sending notifications from n8n to Telegram

Two separate patterns:

**n8n → Telegram directly** (for notifications, not conversations)
- Use n8n's built-in Telegram node
- No OpenClaw involved

**n8n → OpenClaw → Telegram** (for AI-generated responses delivered to Telegram)
- Use `POST /hooks/agent` with `"deliver": true, "channel": "telegram", "to": "<numeric_chat_id>"`
- See `docs/knowledge/n8n-integration.md` for the full webhook contract
