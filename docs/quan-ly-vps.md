# VPS Management

## Table of Contents

- [1. Common Management Commands](#1-common-management-commands)
- [2. Domain + SSL Configuration](#2-domain--ssl-configuration)
- [3. Upgrading Version](#3-upgrading-version)
- [4. Viewing System Information](#4-viewing-system-information)
- [5. Reset to Defaults](#5-reset-to-defaults)
- [6. Troubleshooting](#6-troubleshooting)

---

## 1. Common Management Commands

SSH into your VPS and run the following commands:

### Viewing logs

```bash
# View OpenClaw logs (follow mode)
journalctl -u openclaw -f

# View Caddy logs (reverse proxy)
journalctl -u caddy -f

# View last 200 lines
journalctl -u openclaw --no-pager -n 200
```

Or via API:

```bash
curl -H "Authorization: Bearer $MGMT_KEY" \
  "http://$VPS_IP:9998/api/logs?lines=200&service=openclaw"
```

### Restart

```bash
systemctl restart openclaw
```

Or via API:

```bash
curl -X POST -H "Authorization: Bearer $MGMT_KEY" \
  http://$VPS_IP:9998/api/restart
```

### Stop / Start

```bash
# Stop
systemctl stop openclaw

# Start
systemctl start openclaw
```

Or via API:

```bash
# Stop
curl -X POST -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/stop

# Start
curl -X POST -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/start
```

### Rebuild (restart all services)

```bash
systemctl restart openclaw && systemctl restart caddy
```

Or via API:

```bash
curl -X POST -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/rebuild
```

### Check service status

```bash
systemctl status openclaw caddy openclaw-mgmt
```

Or via API:

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/status
```

### Run CLI commands

```bash
HOME=/opt/openclaw openclaw models scan
HOME=/opt/openclaw openclaw config get
```

Or via API:

```bash
curl -X POST -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"command": "models scan"}' \
  http://$VPS_IP:9998/api/cli
```

---

## 2. Domain + SSL Configuration

### View current domain

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/domain
```

### View current SSL issuer state

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/domain/issuer
```

Use this to quickly confirm whether Caddy is currently serving a Let's Encrypt certificate or has switched to ZeroSSL fallback.

### Change domain (auto-provision ACME SSL: Let's Encrypt, fallback ZeroSSL)

**Requirement:** Domain must already have an A record pointing to the VPS IP.

```bash
curl -X PUT \
  -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "openclaw.example.com", "email": "admin@example.com"}' \
  http://$VPS_IP:9998/api/domain
```

Optionally add an email for ACME notifications:

```bash
curl -X PUT \
  -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "openclaw.example.com", "email": "admin@example.com"}' \
  http://$VPS_IP:9998/api/domain
```

> **Note:**
> - Domain must be lowercase, no `https://`
> - DNS must already resolve to your VPS IP, or the API will return an error
> - If Caddy fails to start with the new domain, system will auto-rollback to the previous IP config

### Manual configuration on VPS

Edit `/opt/openclaw/Caddyfile`:

**With domain:**
```
{
  email admin@example.com
  cert_issuer acme {
    dir https://acme-v02.api.letsencrypt.org/directory
  }
  cert_issuer acme {
    dir https://acme.zerossl.com/v2/DV90
  }
}

openclaw.example.com {
    reverse_proxy 127.0.0.1:18789
}
```

**With IP (self-signed):**
```
180.93.138.155 {
    tls internal
    reverse_proxy 127.0.0.1:18789
}
```

After editing, restart Caddy:

```bash
systemctl restart caddy
```

---

## 3. Upgrading Version

### Via SSH

```bash
npm update -g openclaw@latest && systemctl restart openclaw
```

### Via API

```bash
curl -X POST -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/upgrade
```

> API will immediately return `202 Accepted`, update happens in background. Check status via `/api/status`.

### View current version

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/version
```

---

## 4. Viewing System Information

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/system
```

Example result:

```json
{
  "ok": true,
  "hostname": "openclaw1",
  "ip": "180.93.138.155",
  "os": "Ubuntu 24.04 LTS",
  "uptime": 86400,
  "loadAvg": [0.5, 0.3, 0.2],
  "memory": {
    "total": "4096MB",
    "free": "2048MB",
    "used": "2048MB"
  },
  "disk": {
    "total": "80G",
    "used": "15G",
    "available": "65G",
    "usagePercent": "19%"
  },
  "nodeVersion": "v24.0.0",
  "openclawVersion": "1.0.0"
}
```

---

## 5. Reset to Defaults

> **WARNING:** This action will **DELETE ALL data and configuration**, resetting the system to its initial state.

```bash
curl -X POST \
  -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"confirm": "RESET"}' \
  http://$VPS_IP:9998/api/reset
```

The system will:
1. Stop the OpenClaw service
2. Delete data (`data/`)
3. Restore default (Anthropic) configuration
4. Restart the service

> You must send `{"confirm": "RESET"}` to confirm. Otherwise, the API will return an error.

---

## 6. Troubleshooting

### OpenClaw will not start

```bash
# Check status
systemctl status openclaw caddy openclaw-mgmt

# View error logs
journalctl -u openclaw --no-pager -n 50

# Try restarting
systemctl restart openclaw

# If still failing, rebuild
systemctl restart openclaw && systemctl restart caddy
```

### Cannot access Dashboard

1. **Check services are running:**
   ```bash
   systemctl status openclaw caddy openclaw-mgmt
   ```

2. **Check firewall:**
   ```bash
   ufw status
   # Ports 80 and 443 must be allowed
   ```

3. **Check Caddy:**
   ```bash
   journalctl -u caddy
   ```

4. **Check DNS** (if using a domain):
   ```bash
   dig openclaw.example.com
   # Should return your VPS public IP
   ```

### API key not working

1. **Check if key is valid:**
   ```bash
   curl -X POST -H "Authorization: Bearer $MGMT_KEY" \
     -H "Content-Type: application/json" \
     -d '{"provider": "anthropic", "apiKey": "sk-ant-xxx"}' \
     http://$VPS_IP:9998/api/config/test-key
   ```

2. **Check auth-profiles.json:**
   ```bash
   cat /opt/openclaw/config/agents/main/agent/auth-profiles.json
   ```

3. **Restore/update key:**
   ```bash
   curl -X PUT -H "Authorization: Bearer $MGMT_KEY" \
     -H "Content-Type: application/json" \
     -d '{"provider": "anthropic", "apiKey": "sk-ant-xxx-new"}' \
     http://$VPS_IP:9998/api/config/api-key
   ```

### SSL not working

1. **Check DNS is correct:**
   ```bash
   dig +short your-domain.com
   # Result must be your VPS IP
   ```

2. **Check Caddy logs:**
   ```bash
   journalctl -u caddy | grep -i "tls\|acme\|certificate"
   ```

3. **Try setting the domain again:**
   ```bash
   curl -X PUT -H "Authorization: Bearer $MGMT_KEY" \
     -H "Content-Type: application/json" \
     -d '{"domain": "your-domain.com"}' \
     http://$VPS_IP:9998/api/domain
   ```

### Management API not responding

```bash
# Check service
systemctl status openclaw-mgmt

# Restart service
systemctl restart openclaw-mgmt

# View logs
journalctl -u openclaw-mgmt -f
```

### Important environment variables — Do not delete

The following variables in `/opt/openclaw/.env` **MUST NOT BE DELETED**; if missing, you will lose access to the system:

| Variable                 | Description                                   |
|--------------------------|-----------------------------------------------|
| `OPENCLAW_GATEWAY_TOKEN` | Dashboard access token                        |
| `OPENCLAW_MGMT_API_KEY`  | Management API key (provisioned by my.hitechcloud.vn, do not change manually) |
| `OPENCLAW_VERSION`       | OpenClaw version                              |
| `OPENCLAW_GATEWAY_PORT`  | Internal gateway port                         |