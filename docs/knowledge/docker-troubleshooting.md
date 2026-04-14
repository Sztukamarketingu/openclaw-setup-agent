# Docker Troubleshooting — OpenClaw on VPS

## Quick diagnostics

Run these first whenever something isn't working:

```bash
# Is the container running?
docker ps

# Recent logs (last 50 lines)
docker logs $(docker ps -q --filter name=openclaw) --tail 50

# Follow logs in real time
docker logs $(docker ps -q --filter name=openclaw) -f

# OpenClaw version
docker exec $(docker ps -q) openclaw --version

# Gateway status
docker exec $(docker ps -q) openclaw gateway status

# Channel status
docker exec $(docker ps -q) openclaw channels status --probe

# Run doctor
docker exec $(docker ps -q) openclaw doctor
```

---

## Common problems and fixes

### Container not running

```bash
docker ps -a   # show all containers including stopped ones
```

If the container is stopped:

```bash
# Restart it
docker start $(docker ps -aq --filter name=openclaw)

# Or use compose
cd /docker/openclaw-* && docker compose up -d
```

If it keeps crashing — check logs:
```bash
docker logs $(docker ps -aq --filter name=openclaw) --tail 100
```

---

### "Config invalid" — OpenClaw won't start

Most common cause: invalid JSON in `openclaw.json`.

```bash
# Validate JSON syntax
docker exec $(docker ps -q) python3 -m json.tool /data/.openclaw/openclaw.json

# Or with jq if available
docker exec $(docker ps -q) sh -c "cat /data/.openclaw/openclaw.json | jq ."

# Auto-fix with doctor
docker exec $(docker ps -q) openclaw doctor --fix
docker restart $(docker ps -q)
```

---

### Environment variables not available in container

After adding/changing variables in Hostinger Environment, you must redeploy:

1. Hostinger panel → VPS → Environment → Add/Edit → **Deploy**
2. Wait for container to restart

Verify a variable is set inside the container:

```bash
docker exec $(docker ps -q) printenv ANTHROPIC_API_KEY
# Should print "set" or the first few characters — not the full key
```

If empty: the Deploy step was not completed or the variable name has a typo.

---

### Telegram bot not responding

```bash
# Check channel status
docker exec $(docker ps -q) openclaw channels status --probe

# Check for auth errors in logs
docker logs $(docker ps -q) --tail 50 | grep -i telegram
docker logs $(docker ps -q) --tail 50 | grep -i error
```

Common causes:
- `TELEGRAM_BOT_TOKEN` env var not set → check Hostinger Environment
- Token was revoked → get new token from @BotFather
- `allowFrom` doesn't include your user ID → add it to `openclaw.json`
- Webhook conflict from another service → clear it: `curl "https://api.telegram.org/bot{TOKEN}/deleteWebhook"`

---

### API key errors ("authentication failed", "invalid API key")

```bash
# Check if the key is set
docker exec $(docker ps -q) printenv ANTHROPIC_API_KEY
docker exec $(docker ps -q) printenv OPENROUTER_API_KEY
```

If set but still failing:
- Check for trailing space or newline in the env var value
- Test the key directly: `curl -H "Authorization: Bearer $KEY" https://api.anthropic.com/v1/models`
- Regenerate the key in the provider console if needed

---

### "Port already in use" / Gateway won't start

```bash
# Check what's using port 18789
ss -tlnp | grep 18789
# Or
netstat -tlnp | grep 18789

# Kill the process if it's a stale OpenClaw instance
kill $(lsof -ti:18789)

# Then restart
cd /docker/openclaw-* && docker compose up -d
```

---

### Container runs but Control UI unreachable

The UI is on `127.0.0.1:18789` — only accessible locally. To reach it:

```bash
# From your laptop — SSH tunnel
ssh -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP
# Then open http://127.0.0.1:18789 in browser
```

Or via Tailscale if configured (see `docs/knowledge/tailscale-setup.md`).

If the UI is unreachable even with the tunnel:
```bash
# Confirm Gateway is listening
docker exec $(docker ps -q) ss -tlnp | grep 18789
```

---

### Out of disk space

```bash
# Check disk usage
df -h

# Check Docker disk usage
docker system df

# Remove unused images (safe to run)
docker image prune -f

# Remove stopped containers
docker container prune -f

# Full cleanup (removes all unused Docker objects — confirm before running)
docker system prune -f
```

---

### Out of memory — container OOM killed

```bash
# Check memory
free -h

# Check container memory usage
docker stats --no-stream

# Check kernel OOM log
dmesg | grep -i oom | tail -20
```

If the VPS has < 2GB RAM:
- Add Node.js compile cache to reduce memory: `NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache`
- Set `OPENCLAW_NO_RESPAWN=1` in Hostinger Environment
- Consider upgrading the VPS plan

---

### Update broke something

Roll back to the previous image:

```bash
# List available images
docker images | grep openclaw

# Run a specific older image tag
cd /docker/openclaw-*
# Edit docker-compose.yml to pin the previous image tag
# Then:
docker compose up -d
```

---

## Useful one-liners

```bash
# Restart OpenClaw
docker restart $(docker ps -q --filter name=openclaw)

# Enter the container shell
docker exec -it $(docker ps -q) sh

# Copy a file from container
docker cp $(docker ps -q):/data/.openclaw/openclaw.json ./openclaw-backup.json

# Copy a file to container
docker cp ./new-config.json $(docker ps -q):/data/.openclaw/openclaw.json

# View all environment variables in container
docker exec $(docker ps -q) env | grep -v PATH

# Run OpenClaw CLI command inside container
docker exec $(docker ps -q) openclaw [command]
```

---

## Quick reference

| What | Command |
|------|---------|
| Container status | `docker ps` |
| Logs | `docker logs $(docker ps -q) --tail 50` |
| Restart | `docker restart $(docker ps -q)` |
| Update | `cd /docker/openclaw-* && docker compose pull && docker compose up -d` |
| Enter shell | `docker exec -it $(docker ps -q) sh` |
| Check env var | `docker exec $(docker ps -q) printenv VAR_NAME` |
| Disk usage | `docker system df` |
| Memory usage | `docker stats --no-stream` |
| Validate config | `docker exec $(docker ps -q) python3 -m json.tool /data/.openclaw/openclaw.json` |
