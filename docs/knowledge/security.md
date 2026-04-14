# Security Best Practices for OpenClaw on VPS

## Core rules — always apply

### 1. No secrets in config files

Every API key, token, or password must be an environment variable reference:

```json5
// Correct
token: "${OPENCLAW_HOOKS_TOKEN}",
apiKey: "${ANTHROPIC_API_KEY}",

// Never do this
token: "sk-ant-abc123...",
apiKey: "sk-abc123...",
```

### 2. No secrets in the repository

`.env` must be in `.gitignore`. Check before every commit:

```bash
git status  # .env should never appear here
cat .gitignore  # must contain .env
```

### 3. UI stays on loopback

The OpenClaw Control UI (`http://127.0.0.1:18789`) must not be exposed on a public IP without authentication.

```json5
gateway: {
  listen: "127.0.0.1:18789",   // Loopback only — correct
  // NEVER: "0.0.0.0:18789"    // Exposes to internet — wrong
},
```

### 4. Separate tokens for dashboard and hooks

`OPENCLAW_GATEWAY_TOKEN` and `OPENCLAW_HOOKS_TOKEN` must be different values. Generate each independently:

```bash
openssl rand -hex 32   # for OPENCLAW_GATEWAY_TOKEN
openssl rand -hex 32   # for OPENCLAW_HOOKS_TOKEN
```

### 5. Telegram bot not open to everyone

Never leave `dmPolicy: "open"` in production. Set allowlist or closed:

```json5
channels: {
  telegram: {
    dmPolicy: "allowlist",
    allowlist: ["@yourusername"],
  },
},
```

### 6. Hooks token in Authorization header only

n8n and other systems must send the token as a header, never in the query string:

```
Correct:   Authorization: Bearer <token>
Wrong:     ?token=<token>    ← OpenClaw returns 400
```

---

## Remote access to Control UI

Never open port 18789 to the internet. Use one of:

### Option A: SSH tunnel (simplest)

```bash
# Run this on your local machine
ssh -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP
```

Then open `http://127.0.0.1:18789` in your local browser.

Keep the terminal open while using the UI. Add `-N -f` flags to run in background:

```bash
ssh -N -f -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP
```

### Option B: Tailscale (recommended for team access)

1. Install Tailscale on VPS: `curl -fsSL https://tailscale.com/install.sh | sh`
2. Run: `tailscale up`
3. Install Tailscale on your devices
4. Access OpenClaw via VPS Tailscale IP: `http://100.x.x.x:18789`

Configure gateway to listen on Tailscale interface:

```json5
gateway: {
  listen: "100.x.x.x:18789",    // Tailscale IP of VPS
  auth: {
    token: "${OPENCLAW_GATEWAY_TOKEN}",
  },
},
```

### Option C: Reverse proxy with auth (for team deployment)

Use nginx or Caddy with basic auth or OAuth in front of port 18789. Only expose HTTPS (port 443).

Example Caddy config:

```
openclaw.yourdomain.com {
    basicauth {
        admin $2a$14$...    # bcrypt hash
    }
    reverse_proxy 127.0.0.1:18789
}
```

---

## Environment variable security on VPS

Store OpenClaw env vars in a dedicated file with restricted permissions:

```bash
# Create the file
sudo touch /etc/openclaw.env
sudo chmod 600 /etc/openclaw.env    # Only root can read
sudo chown root:root /etc/openclaw.env

# Write variables
sudo tee /etc/openclaw.env << 'EOF'
ANTHROPIC_API_KEY=sk-ant-...
TELEGRAM_BOT_TOKEN=7123...
OPENCLAW_HOOKS_TOKEN=abc123...
EOF
```

Reference in systemd unit (`systemctl edit openclaw`):

```ini
[Service]
EnvironmentFile=/etc/openclaw.env
```

### Never do this

```bash
# Wrong — key visible in process list to all users
openclaw gateway start --api-key sk-ant-xxx

# Wrong — key in shell history
export ANTHROPIC_API_KEY=sk-ant-xxx
```

---

## Backup

Back up these paths before any changes:

```bash
# Back up OpenClaw config and state
tar -czf openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/

# Copy to safe location
scp openclaw-backup-*.tar.gz backup-server:~/backups/
```

Include in backup:
- `~/.openclaw/config.json5`
- `~/.openclaw/workspace/` (SOUL.md, knowledge files)
- `/etc/openclaw.env` (store separately and securely — contains secrets)

Do not commit backups to git if they contain `openclaw.env`.

---

## VPS firewall

Recommended Hostinger VPS firewall rules:

| Port | Protocol | Source | Action | Purpose |
|------|----------|--------|--------|---------|
| 22 | TCP | Your IP | Allow | SSH access |
| 18789 | TCP | * | Deny | OpenClaw UI — never expose |
| 443 | TCP | * | Allow | HTTPS (if reverse proxy) |
| 80 | TCP | * | Allow | HTTP redirect to HTTPS |

Apply via Hostinger panel → VPS → Firewall, or via `ufw`:

```bash
ufw allow 22/tcp
ufw deny 18789/tcp
ufw enable
```
