# Hostinger VPS API — Reference

## Authentication

All Hostinger API requests require a Bearer token:

```
Authorization: Bearer {HOSTINGER_API_KEY}
Content-Type: application/json
```

API base URL: `https://api.hostinger.com/v1`

Get your API key: Hostinger panel → Account → API tokens
Full API docs: https://developers.hostinger.com

---

## Key endpoints used by the agent

### List VPS instances

```http
GET /vps/virtual-machines
```

Response includes: `id`, `hostname`, `ip_address`, `status`, `plan`, `location`.

Use the `id` for all subsequent VPS-specific requests.

### Get single VPS details

```http
GET /vps/virtual-machines/{vmId}
```

Use to confirm VPS status before deployment. Status should be `running`.

### Get VPS firewall rules (verification)

```http
GET /vps/virtual-machines/{vmId}/firewall
```

Use to verify port 22 (SSH) is open for deployment.

---

## SSH access (used for deployment)

Hostinger VPS supports SSH key authentication. The agent uses SSH for:
- Setting environment variables
- Uploading configuration files
- Restarting OpenClaw Gateway
- Reading logs

### SSH command pattern

```bash
ssh -i "${VPS_SSH_KEY_PATH}" \
    -p "${VPS_PORT}" \
    -o StrictHostKeyChecking=no \
    "${VPS_USERNAME}@${VPS_HOSTNAME}" \
    "<command>"
```

### SCP for file upload

```bash
scp -i "${VPS_SSH_KEY_PATH}" \
    -P "${VPS_PORT}" \
    local-file.txt \
    "${VPS_USERNAME}@${VPS_HOSTNAME}:/remote/path/file.txt"
```

---

## Setting environment variables on the VPS

OpenClaw reads environment variables from the system environment. The recommended approach is to write them to a dedicated file that gets loaded by the systemd service.

### Write env file via SSH

```bash
ssh -i "${VPS_SSH_KEY_PATH}" "${VPS_USERNAME}@${VPS_HOSTNAME}" \
"cat > /etc/openclaw.env << 'EOF'
ANTHROPIC_API_KEY=your_value
TELEGRAM_BOT_TOKEN=your_value
OPENCLAW_HOOKS_TOKEN=your_value
EOF
chmod 600 /etc/openclaw.env"
```

### Configure systemd to load the env file

```bash
ssh -i "${VPS_SSH_KEY_PATH}" "${VPS_USERNAME}@${VPS_HOSTNAME}" \
"systemctl edit openclaw --force << 'EOF'
[Service]
EnvironmentFile=/etc/openclaw.env
EOF
systemctl daemon-reload"
```

### Verify variables are loaded

```bash
ssh -i "${VPS_SSH_KEY_PATH}" "${VPS_USERNAME}@${VPS_HOSTNAME}" \
"systemctl show openclaw --property=Environment"
```

---

## OpenClaw paths on the VPS

| Path | Contents |
|------|----------|
| `~/.openclaw/config.json5` | Main OpenClaw configuration |
| `~/.openclaw/workspace/` | Agent workspace (SOUL.md, knowledge files) |
| `~/.openclaw/workspace/SOUL.md` | Agent identity and personality |
| `/etc/openclaw.env` | Environment variables (recommended location) |

---

## Common Hostinger VPS notes

- Default root access is available on most Hostinger VPS plans
- SSH is enabled by default; find the root password in the Hostinger panel on first setup
- Hostinger VPS runs Ubuntu 22.04 or 24.04 in most regions
- Node.js may need to be installed or updated (OpenClaw requires Node 22 LTS 22.16+ or Node 24)

### Check Node.js version on VPS

```bash
ssh -i "${VPS_SSH_KEY_PATH}" "${VPS_USERNAME}@${VPS_HOSTNAME}" "node --version"
```

### Install/update Node.js if needed

```bash
ssh -i "${VPS_SSH_KEY_PATH}" "${VPS_USERNAME}@${VPS_HOSTNAME}" \
"curl -fsSL https://deb.nodesource.com/setup_24.x | bash - && apt-get install -y nodejs"
```

---

## Troubleshooting SSH connection

**Permission denied (publickey):**
- Confirm the correct key path in `VPS_SSH_KEY_PATH`
- Check that the public key is in `/root/.ssh/authorized_keys` on the VPS
- Use `ssh -v` for verbose output to diagnose

**Connection refused:**
- Confirm the VPS is running (Hostinger panel → VPS → status)
- Confirm `VPS_PORT` is correct (default 22)
- Check Hostinger firewall — port 22 must be allowed

**Host key verification failed:**
- Add `-o StrictHostKeyChecking=no` for first connection, or
- Run `ssh-keyscan ${VPS_HOSTNAME} >> ~/.ssh/known_hosts` locally
