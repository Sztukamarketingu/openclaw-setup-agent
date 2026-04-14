# Discord Channel Setup

## When to choose Discord over Telegram

| Use case | Better choice |
|----------|--------------|
| Personal assistant, field team, quick comms | Telegram |
| Developer community, tech team, gaming | Discord |
| Community with channels, roles, threads | Discord |
| B2B internal team tool | Either |

Discord offers richer server structure (channels, roles, threads) but is more complex to set up than Telegram.

---

## Step 1: Create a Discord application and bot

1. Go to https://discord.com/developers/applications
2. Click **New Application** — give it a name (e.g. "Acme Assistant")
3. In the left sidebar → **Bot** → click **Add Bot**
4. Under **Token** → click **Reset Token** → copy the token
5. Save it as `DISCORD_BOT_TOKEN` in your `.env`

**Privileged Gateway Intents** (required for reading messages):
- Enable: **Message Content Intent**
- Enable: **Server Members Intent** (if using role-based access)

---

## Step 2: Invite the bot to your server

1. Left sidebar → **OAuth2** → **URL Generator**
2. Scopes: check `bot`
3. Bot Permissions: check `Send Messages`, `Read Messages/View Channels`, `Read Message History`
4. Copy the generated URL → open in browser → select your server → Authorize

---

## Step 3: Get channel and server IDs

Enable Developer Mode in Discord:
- User Settings → Advanced → **Developer Mode** ON

Right-click any channel → **Copy Channel ID** — save this as `DISCORD_CHANNEL_ID`.
Right-click your server name → **Copy Server ID** — save as `DISCORD_SERVER_ID`.

---

## Step 4: Add Discord token to Hostinger Environment

In Hostinger panel → VPS → **Environment**, add:

```
DISCORD_BOT_TOKEN=YOUR_TOKEN_FROM_DISCORD_DEVELOPER_PORTAL
```

Click **Deploy**.

---

## Step 5: Configure Discord in openclaw.json

Send to agent via OpenClaw dashboard:

```
Add Discord channel configuration to /data/.openclaw/openclaw.json:

{
  "channels": {
    "discord": {
      "enabled": true,
      "botToken": "${DISCORD_BOT_TOKEN}",
      "dmPolicy": "allowlist",
      "allowFrom": ["YOUR_DISCORD_USER_ID"],
      "servers": {
        "YOUR_SERVER_ID": {
          "channels": {
            "YOUR_CHANNEL_ID": {
              "enabled": true,
              "agentId": "main"
            }
          }
        }
      }
    }
  }
}
```

Replace:
- `YOUR_DISCORD_USER_ID` — your Discord user ID (right-click your name → Copy User ID)
- `YOUR_SERVER_ID` — from Step 3
- `YOUR_CHANNEL_ID` — from Step 3

---

## Step 6: Verify

Send a message in the configured Discord channel. The agent should respond.

```bash
docker exec $(docker ps -q) openclaw channels status --probe
```

Expected: `discord: connected (bot: YourBotName#1234)`

---

## Running Telegram and Discord simultaneously

Both channels can run at the same time, routing to the same or different agents:

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "dmPolicy": "allowlist",
      "allowFrom": [123456789],
      "agentId": "main",
      "groups": { "*": { "enabled": false } }
    },
    "discord": {
      "enabled": true,
      "botToken": "${DISCORD_BOT_TOKEN}",
      "dmPolicy": "allowlist",
      "allowFrom": ["987654321098765432"],
      "servers": {
        "SERVER_ID": {
          "channels": {
            "CHANNEL_ID": { "enabled": true, "agentId": "main" }
          }
        }
      }
    }
  }
}
```

---

## Troubleshooting

### Bot is in server but not responding

1. Confirm **Message Content Intent** is enabled in Discord Developer Portal
2. Confirm the channel ID in config matches the channel where you're messaging
3. Confirm `DISCORD_BOT_TOKEN` is set: `docker exec $(docker ps -q) printenv DISCORD_BOT_TOKEN`
4. Check logs: `docker logs $(docker ps -q) --tail 30 | grep -i discord`

### Bot responding in wrong channel

Check `servers.SERVER_ID.channels` in config — only listed channel IDs receive messages.

### "Missing Access" error in logs

Bot doesn't have permission to read or write in that channel. Go to Discord server → channel settings → Permissions → add bot with Send Messages + View Channel.

### Bot responds to everyone

`dmPolicy: "allowlist"` with `allowFrom` populated should restrict access. Check that your Discord user ID is in `allowFrom` and that it's the numeric ID (not username).
