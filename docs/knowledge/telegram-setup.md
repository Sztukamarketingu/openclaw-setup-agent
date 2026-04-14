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

## Step 2: Find your Telegram chat ID (for allowlist setup)

To restrict who can use the bot, you need Telegram user IDs or usernames.

**Get your username:**
Telegram → Settings → Username (e.g. `@yourname`)

**Get a numeric chat ID:**
1. Send a message to your bot
2. Open: `https://api.telegram.org/bot{TOKEN}/getUpdates`
3. Find `"from": {"id": 123456789}` — that's the user ID

---

## Step 3: Configure Telegram in OpenClaw config

### Open bot (anyone can message — POC only)

```json5
channels: {
  telegram: {
    enabled: true,
    token: "${TELEGRAM_BOT_TOKEN}",
    dmPolicy: "open",
  },
},
```

### Allowlist (recommended for production)

```json5
channels: {
  telegram: {
    enabled: true,
    token: "${TELEGRAM_BOT_TOKEN}",
    dmPolicy: "allowlist",
    allowlist: ["@yourusername", "@colleague"],
  },
},
```

### Route to specific agent

```json5
channels: {
  telegram: {
    enabled: true,
    token: "${TELEGRAM_BOT_TOKEN}",
    dmPolicy: "allowlist",
    allowlist: ["@yourusername"],
    agentId: "main",                 // Which agent handles Telegram messages
  },
},
```

---

## Step 4: Test the connection

After deploying config and restarting Gateway:

```bash
openclaw channels status --probe
```

Expected output for a working channel:
```
telegram: connected (bot: @your_bot_name)
```

Then send "hello" to your bot in Telegram. The agent should reply within a few seconds.

---

## Troubleshooting

### Bot not responding at all

1. Check token is correct in `/etc/openclaw.env`
2. Confirm `channels.telegram.enabled: true` in config
3. Check Gateway is running: `openclaw gateway status`
4. Check logs: `openclaw logs --tail 30`

### "Unauthorized" error in logs

Token is wrong or expired. Generate a new token in BotFather:
`/revoke` → choose your bot → copy new token → update `.env` and VPS env file → restart Gateway

### Bot responds but wrong agent

Check `agentId` in the channel config. Confirm the agent ID exists in `agents.list`.

### "This bot is restricted" message to users

The bot's `dmPolicy` is set to `allowlist` and the user is not on it. Add their `@username` to the allowlist, update config, restart Gateway.

### BotFather-verified: token is correct but still not working

Telegram webhooks and long-polling can conflict. OpenClaw uses long-polling by default. If you previously set a webhook on the bot (e.g. via another service), clear it:

```bash
curl "https://api.telegram.org/bot{TOKEN}/deleteWebhook"
```

---

## Sending notifications FROM n8n TO Telegram

If you want n8n to send Telegram messages (not through OpenClaw):
- Use n8n's built-in Telegram node
- This is separate from OpenClaw's Telegram channel
- OpenClaw handles inbound conversations; n8n can handle outbound notifications

If you want n8n to trigger OpenClaw which then delivers to Telegram:
- See `docs/knowledge/n8n-integration.md`
- Use `POST /hooks/agent` with `"deliver": true, "channel": "telegram", "to": "<chat_id>"`
