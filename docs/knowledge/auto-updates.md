# Auto-Updates — OpenClaw and OS

## Two separate update systems

| System | Tool | Frequency |
|--------|------|-----------|
| **OpenClaw** (Docker container) | `docker compose pull` via cron | Daily at 4:00 |
| **Ubuntu OS** (kernel, SSH, packages) | `unattended-upgrades` | Daily automatically |

---

## Part 1: OpenClaw auto-update (Docker)

OpenClaw runs in a Docker container. New versions are new images. Without auto-update you must manually SSH in and update.

> **Why not `openclaw update`?** That command updates an npm installation (bare-metal). In Docker, changes inside the container are temporary — on next restart the container reverts to the version from the image. The correct Docker update is pulling a new image (`docker compose pull`).

### Step 1: Find your compose directory

Hostinger installs OpenClaw in `/docker/openclaw-XXXX/` where `XXXX` is a random string (different on each VPS).

```bash
ls -d /docker/openclaw-*
```

The `-d` flag shows the folder name instead of its contents. Note the path — you'll use it in the script.

### Step 2: Create the update script

Connect via SSH and create:

```bash
nano /root/openclaw-update.sh
```

Paste this content:

```bash
#!/bin/bash
cd /docker/openclaw-* || exit 1
docker compose pull
docker compose up -d
docker image prune -f
echo "$(date): Update completed" >> /var/log/openclaw-update.log
```

Save: `Ctrl+O` → Enter → `Ctrl+X`

What each line does:

| Line | Purpose |
|------|---------|
| `cd /docker/openclaw-*` | Enter compose directory (glob matches the one folder) |
| `docker compose pull` | Check for new image — download if found, skip if not |
| `docker compose up -d` | Restart container with new image (only if image changed) |
| `docker image prune -f` | Remove old unused images (frees disk space) |
| `echo ... >> /var/log/...` | Log the run date for audit |

### Step 3: Make it executable

```bash
chmod +x /root/openclaw-update.sh
```

### Step 4: Add to cron

```bash
crontab -e
```

Add at the bottom:

```
0 4 * * * /root/openclaw-update.sh
```

Save and exit.

Verify it saved:
```bash
crontab -l
```

You should see `0 4 * * * /root/openclaw-update.sh`.

### Step 5: Test manually

```bash
/root/openclaw-update.sh
cat /var/log/openclaw-update.log
```

If you see a timestamped entry — everything works.

### Check current OpenClaw version

```bash
docker exec $(docker ps -q) openclaw --version
```

---

## Cron schedule reference

| Entry | When |
|-------|------|
| `0 4 * * *` | Daily at 4:00 |
| `30 3 * * *` | Daily at 3:30 |
| `0 */6 * * *` | Every 6 hours |

---

## Part 2: OS security patches (unattended-upgrades)

Hostinger VPS is unmanaged — Ubuntu does not update itself. Without this step, security vulnerabilities in the kernel, OpenSSH, and other system packages stay open until you manually run `apt upgrade`.

`unattended-upgrades` checks daily for security patches and installs them silently in the background. This covers the **OS only** — not OpenClaw (that's handled by the Docker cron above).

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

Select "Yes" in the configuration window. The system now patches itself automatically.
