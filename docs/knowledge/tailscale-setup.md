# Tailscale — Secure Remote Access

## What Tailscale does

Tailscale creates a private WireGuard network (tailnet) between your devices. Your VPS and laptop become part of the same private network — no public ports needed, no exposed services.

**Benefits for OpenClaw:**
- Access the Control UI from anywhere without SSH tunnel commands
- SSH to VPS through Tailscale — no need to keep port 22 open publicly
- Share access with team members without exposing the VPS to the internet
- OpenClaw webhooks can be reached from n8n over Tailscale (same private network)

---

## Install Tailscale on the VPS

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

Follow the login URL printed in the terminal — authenticate with your Tailscale account (or create one at https://tailscale.com).

After login:

```bash
# Get Tailscale IP of this VPS
tailscale ip -4

# Check connection status
tailscale status
```

The VPS now has a stable Tailscale IP (e.g. `100.64.x.x`) that doesn't change even if the public IP changes.

---

## Install Tailscale on your devices

- **Mac/Windows:** download from https://tailscale.com/download
- **iPhone/Android:** Tailscale app from App Store / Google Play
- **Linux:** `curl -fsSL https://tailscale.com/install.sh | sh && tailscale up`

All devices login to the same Tailscale account → they're on the same private network.

---

## Access OpenClaw UI via Tailscale

After Tailscale is set up on both the VPS and your device:

1. Configure OpenClaw Gateway to listen on the Tailscale IP:

```json5
gateway: {
  listen: "100.x.x.x:18789",   // Replace with your VPS Tailscale IP
  auth: {
    token: "${OPENCLAW_GATEWAY_TOKEN}",
  },
},
```

2. Open `http://100.x.x.x:18789` in your browser from any device on your tailnet.

No tunnel command needed. Works from phone too.

---

## SSH via Tailscale (recommended)

Configure Tailscale SSH to replace public SSH:

```bash
# Enable Tailscale SSH on the VPS
tailscale up --ssh
```

Now connect using the machine name instead of IP:

```bash
ssh root@your-vps-machine-name
```

Tailscale handles authentication — no need to manage SSH keys manually. You can also close port 22 on the public firewall entirely.

---

## Tailscale Serve — HTTPS access inside tailnet

Tailscale Serve adds HTTPS to your service within the tailnet only (not public internet):

```bash
# Serve OpenClaw UI over HTTPS on your tailnet
sudo tailscale serve --bg http://localhost:18789
```

Access via: `https://your-machine-name.your-tailnet.ts.net`

This gives you:
- Automatic HTTPS certificate (valid for your tailnet)
- Access from any Tailscale device
- No exposure to public internet

---

## Tailscale Funnel — limited public exposure (optional)

Tailscale Funnel makes a service accessible from the public internet through Tailscale's servers. Only use this if you need a public URL for webhooks (e.g. n8n running outside your tailnet needs to reach OpenClaw).

```bash
sudo tailscale funnel 18789
```

**Security warning:** Tailscale Funnel exposes the service publicly. Always require `gateway.auth.token` before enabling Funnel.

Recommended: keep n8n and OpenClaw on the same tailnet and use Serve, not Funnel.

---

## Add team members

1. Invite them in Tailscale admin console → Users → Invite
2. They install Tailscale on their devices and join your tailnet
3. They can access `http://100.x.x.x:18789` from their devices

To restrict which team members can access the VPS, use Tailscale ACLs in the admin console.

---

## Troubleshooting

### Tailscale disconnected

```bash
tailscale status
sudo systemctl restart tailscaled
tailscale up
```

### Can't reach VPS from device

```bash
# On VPS
tailscale ping <device-tailscale-ip>

# On device
tailscale ping <vps-tailscale-ip>
```

If no response: check that both devices are logged into the same Tailscale account and both show as "Connected" in the Tailscale admin console.

### UI unreachable after switching gateway.listen to Tailscale IP

- Confirm the Tailscale IP in `gateway.listen` matches the actual VPS Tailscale IP: `tailscale ip -4`
- Confirm your device is connected to Tailscale
- Check OpenClaw is running: `docker ps`

### Tailscale Serve shows "Wrong version number" (SSL error)

OpenClaw runs HTTP; Tailscale Serve adds HTTPS. If they get out of sync:

```bash
sudo tailscale serve reset
sudo tailscale serve --bg http://localhost:18789
```
