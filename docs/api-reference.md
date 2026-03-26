# API Reference — Management API

## Table of Contents

- [General Information](#general-information)
- [Authentication](#authentication)
- [Rate Limiting](#rate-limiting)
- [Service Information](#service-information)
  - [GET /api/info](#get-apiinfo)
  - [GET /api/status](#get-apistatus)
  - [GET /api/system](#get-apisystem)
- [Domain & SSL](#domain--ssl)
  - [GET /api/domain](#get-apidomain)
  - [PUT /api/domain](#put-apidomain)
- [Version](#version)
  - [GET /api/version](#get-apiversion)
  - [POST /api/upgrade](#post-apiupgrade)
- [Service Control](#service-control)
  - [POST /api/restart](#post-apirestart)
  - [POST /api/stop](#post-apistop)
  - [POST /api/start](#post-apistart)
  - [POST /api/rebuild](#post-apirebuild)
  - [POST /api/reset](#post-apireset)
- [Logs](#logs)
  - [GET /api/logs](#get-apilogs)
- [Configuration](#configuration)
  - [GET /api/config](#get-apiconfig)
  - [PUT /api/config/provider](#put-apiconfigprovider)
  - [PUT /api/config/api-key](#put-apiconfigapi-key)
  - [POST /api/config/test-key](#post-apiconfigtest-key)
- [Messaging Channels](#messaging-channels)
  - [GET /api/channels](#get-apichannels)
  - [PUT /api/channels/:channel](#put-apichannelschannel)
  - [DELETE /api/channels/:channel](#delete-apichannelschannel)
- [Environment Variables](#environment-variables)
  - [GET /api/env](#get-apienv)
  - [PUT /api/env/:key](#put-apienvkey)
  - [DELETE /api/env/:key](#delete-apienvkey)
- [CLI Proxy](#cli-proxy)
  - [POST /api/cli](#post-apicli)
- [Common Error Codes](#common-error-codes)

---

## General Information

| Property         | Value                                     |
|------------------|-------------------------------------------|
| **Base URL**     | `http://<VPS_IP>:9998`                    |
| **Port**         | 9998                                      |
| **Protocol**     | HTTP                                      |
| **Content-Type** | `application/json`                        |
| **Body Size Max**| 100KB                                     |

---

## Authentication

All API endpoints require **Bearer Token** authentication:

```
Authorization: Bearer <OPENCLAW_MGMT_API_KEY>
```

The Management API Key is issued by my.hitechcloud.vn when the service is created. Check your key in the my.hitechcloud.vn control panel.

> **Important:** Do **not** manually change `OPENCLAW_MGMT_API_KEY` in the `.env` file. Changing it will break the my.hitechcloud.vn panel's connection.

---

## Rate Limiting

- **10** failed authentication attempts → IP is blocked for **15 minutes**
- Blocked response: `429 Too Many Requests`

---

## Service Information

### GET /api/info

Service overview information.

**Response:**

```json
{
  "ok": true,
  "domain": "openclaw.example.com",
  "ip": "180.93.138.155",
  "pairUrl": "http://180.93.138.155:9998/pair?token=abc...",
  "gatewayToken": "abc123...",
  "mgmtApiKey": "def456...7890",
  "status": "running",
  "version": "1.0.0"
}
```

| Field          | Type          | Description                                 |
|----------------|---------------|---------------------------------------------|
| `domain`       | string/null   | Current domain (null if using IP)           |
| `ip`           | string        | VPS IP address                              |
| `pairUrl`      | string        | Device pairing URL (with token)             |
| `gatewayToken` | string        | Gateway access token                        |
| `mgmtApiKey`   | string        | Management API Key (shows 8 head + 4 tail)  |
| `status`       | string        | `running` / `stopped` / `inactive` / `not_found` |
| `version`      | string        | OpenClaw version                            |

**Example:**

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/info
```

---

### GET /api/status

Detailed status of services.

**Response:**

```json
{
  "ok": true,
  "openclaw": {
    "status": "running",
    "startedAt": "2026-02-14T10:00:00Z"
  },
  "caddy": {
    "status": "running"
  },
  "version": "1.0.0",
  "gatewayPort": "18789"
}
```

**Example:**

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/status
```

---

### GET /api/system

System info (CPU, RAM, disk, OS).

**Response:**

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

**Example:**

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/system
```

---

## Domain & SSL

### GET /api/domain

View current domain and SSL status.

**Response:**

```json
{
  "ok": true,
  "domain": "openclaw.example.com",
  "ip": "180.93.138.155",
  "ssl": true,
  "selfSignedSSL": false,
  "caddyfile": "openclaw.example.com {\n    tls {\n        issuer acme {...}\n    }\n    reverse_proxy 127.0.0.1:18789\n}"
}
```

| Field           | Description                                     |
|-----------------|-------------------------------------------------|
| `ssl`           | `true` if Let's Encrypt SSL is enabled          |
| `selfSignedSSL` | `true` if using self-signed cert (IP only)      |

**Example:**

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/domain
```

---

### PUT /api/domain

Change the domain and auto-configure Let's Encrypt SSL.

**Request body:**

```json
{
  "domain": "openclaw.example.com",
  "email": "admin@example.com"
}
```

| Field    | Required | Description                                  |
|----------|----------|----------------------------------------------|
| `domain` | Yes      | FQDN (all lowercase, DNS already pointed to VPS) |
| `email`  | No       | Email for Let's Encrypt registration         |

**Successful response:**

```json
{
  "ok": true,
  "domain": "openclaw.example.com"
}
```

**Error response:**

```json
{
  "ok": false,
  "error": "DNS for openclaw.example.com resolves to 1.2.3.4 — does not match server IP (180.93.138.155)."
}
```

> If Caddy can't start with the new domain, config will automatically roll back to IP-based.

**Example:**

```bash
curl -X PUT -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "openclaw.example.com"}' \
  http://$VPS_IP:9998/api/domain
```

---

## Version

### GET /api/version

Get current OpenClaw version.

**Response:**

```json
{
  "ok": true,
  "version": "1.0.0"
}
```

**Example:**

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/version
```

---

### POST /api/upgrade

Update OpenClaw to the latest version (runs in background).

**Request body:** None

**Response:** `202 Accepted`

```json
{
  "ok": true,
  "message": "Upgrade started. Check /api/status for progress."
}
```

> The upgrade process runs in the background. Use `/api/status` to check when the service becomes `running` again.

**Example:**

```bash
curl -X POST -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/upgrade
```

---

## Service Control

### POST /api/restart

Restart the OpenClaw service.

**Response:**

```json
{
  "ok": true,
  "status": "running"
}
```

**Example:**

```bash
curl -X POST -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/restart
```

---

### POST /api/stop

Stop the OpenClaw service.

**Response:**

```json
{
  "ok": true,
  "message": "OpenClaw stopped."
}
```

**Example:**

```bash
curl -X POST -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/stop
```

---

### POST /api/start

Start the OpenClaw service.

**Response:**

```json
{
  "ok": true,
  "status": "running"
}
```

**Example:**

```bash
curl -X POST -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/start
```

---

### POST /api/rebuild

Restart both OpenClaw and Caddy.

**Response:**

```json
{
  "ok": true,
  "status": "running"
}
```

**Example:**

```bash
curl -X POST -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/rebuild
```

---

### POST /api/reset

Remove all data and restore default settings.

**Request body:**

```json
{
  "confirm": "RESET"
}
```

> **You must** send `{"confirm": "RESET"}` to confirm. This action is **IRREVERSIBLE**.

**Response:**

```json
{
  "ok": true,
  "status": "running",
  "message": "Reset complete. Config reverted to defaults."
}
```

**Example:**

```bash
curl -X POST -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"confirm": "RESET"}' \
  http://$VPS_IP:9998/api/reset
```

---

## Logs

### GET /api/logs

View service logs.

**Query parameters:**

| Parameter | Default | Description                      |
|-----------|---------|----------------------------------|
| `lines`   | 100     | Number of lines (1–1000)         |
| `service` | `openclaw` | Service: `openclaw` or `caddy` |

**Response:**

```json
{
  "ok": true,
  "service": "openclaw",
  "lines": 100,
  "logs": "2026-02-14 10:00:00 Server started on port 18789\n..."
}
```

**Example:**

```bash
# OpenClaw logs, 200 lines
curl -H "Authorization: Bearer $MGMT_KEY" \
  "http://$VPS_IP:9998/api/logs?lines=200&service=openclaw"

# Caddy logs
curl -H "Authorization: Bearer $MGMT_KEY" \
  "http://$VPS_IP:9998/api/logs?service=caddy"
```

---

## Configuration

### GET /api/config

View current configuration (sensitive values are masked).

**Response:**

```json
{
  "ok": true,
  "provider": "anthropic",
  "model": "anthropic/claude-opus-4-5",
  "apiKeys": {
    "anthropic": "sk-ant-xx...xxxx",
    "openai": null,
    "gemini": null
  },
  "config": {
    "agents": { "..." },
    "gateway": { "..." },
    "browser": { "..." },
    "channels": { "..." },
    "plugins": { "..." }
  }
}
```

**Example:**

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/config
```

---

### PUT /api/config/provider

Change AI provider and model.

**Request body:**

```json
{
  "provider": "anthropic",
  "model": "anthropic/claude-sonnet-4-20250514"
}
```

| Field      | Required | Valid values                                      |
|------------|----------|---------------------------------------------------|
| `provider` | Yes      | `anthropic`, `openai`, `gemini`                   |
| `model`    | Yes      | Full model ID (ex: `anthropic/claude-sonnet-4-20250514`) |

**Response:**

```json
{
  "ok": true,
  "provider": "anthropic",
  "model": "anthropic/claude-sonnet-4-20250514"
}
```

> When changed, system loads provider template but preserves all other settings (channels, plugins). OpenClaw automatically restarts.

**Example:**

```bash
curl -X PUT -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"provider": "openai", "model": "openai/gpt-4o"}' \
  http://$VPS_IP:9998/api/config/provider
```

---

### PUT /api/config/api-key

Update API key for a provider.

**Request body:**

```json
{
  "provider": "anthropic",
  "apiKey": "sk-ant-xxx..."
}
```

| Field      | Required | Description      |
|------------|----------|------------------|
| `provider` | Yes      | `anthropic`, `openai`, `gemini` |
| `apiKey`   | Yes      | The API key      |

**Response:**

```json
{
  "ok": true,
  "provider": "anthropic",
  "apiKey": "sk-ant-xx...xxxx"
}
```

> The key is saved in `auth-profiles.json` (priority) and `.env` (fallback). OpenClaw auto restarts.

**Example:**

```bash
curl -X PUT -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"provider": "gemini", "apiKey": "AIzaSy..."}' \
  http://$VPS_IP:9998/api/config/api-key
```

---

### POST /api/config/test-key

Test if an API key is valid (does not save it).

**Request body:**

```json
{
  "provider": "anthropic",
  "apiKey": "sk-ant-xxx..."
}
```

**Success Response:**

```json
{
  "ok": true,
  "error": null
}
```

**Failure Response:**

```json
{
  "ok": false,
  "error": "API key invalid or expired"
}
```

**Example:**

```bash
curl -X POST -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"provider": "openai", "apiKey": "sk-xxx..."}' \
  http://$VPS_IP:9998/api/config/test-key
```

---

## Messaging Channels

### GET /api/channels

List all messaging channels and status.

**Response:**

```json
{
  "ok": true,
  "channels": {
    "telegram": {
      "configured": true,
      "enabled": true,
      "token": "12345678...wxYZ"
    },
    "discord": {
      "configured": false,
      "enabled": false,
      "token": null
    },
    "slack": {
      "configured": false,
      "enabled": false,
      "token": null
    },
    "zalo": {
      "configured": false,
      "enabled": false,
      "token": null
    }
  }
}
```

**Example:**

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/channels
```

---

### PUT /api/channels/:channel

Add or update a messaging channel.

**Path parameter:** `channel` = `telegram` | `discord` | `slack` | `zalo`

**Request body:**

```json
{
  "token": "bot-token-here",
  "appToken": "xapp-...",
  "dmPolicy": "open",
  "allowFrom": ["*"]
}
```

| Field      | Required | Description                                |
|------------|----------|--------------------------------------------|
| `token`    | Yes      | Bot token                                  |
| `appToken` | No       | App-Level Token (Slack only)               |
| `dmPolicy` | No       | `"open"` (anyone can DM, default)          |
| `allowFrom`| No       | User/group list: `["*"]` = all (default)   |

**Response:**

```json
{
  "ok": true,
  "channel": "telegram",
  "token": "12345678...wxYZ"
}
```

> For Discord, Slack, Zalo: the corresponding plugin is auto-enabled.

**Example:**

```bash
# Telegram
curl -X PUT -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"token": "123456789:ABCdef..."}' \
  http://$VPS_IP:9998/api/channels/telegram

# Slack (needs appToken too)
curl -X PUT -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"token": "xoxb-...", "appToken": "xapp-..."}' \
  http://$VPS_IP:9998/api/channels/slack
```

---

### DELETE /api/channels/:channel

Remove a messaging channel.

**Path parameter:** `channel` = `telegram` | `discord` | `slack` | `zalo`

**Response:**

```json
{
  "ok": true,
  "channel": "telegram",
  "removed": true
}
```

**Example:**

```bash
curl -X DELETE -H "Authorization: Bearer $MGMT_KEY" \
  http://$VPS_IP:9998/api/channels/discord
```

---

## Environment Variables

### GET /api/env

View all environment variables (sensitive values are masked).

**Response:**

```json
{
  "ok": true,
  "env": {
    "OPENCLAW_VERSION": "latest",
    "OPENCLAW_GATEWAY_PORT": "18789",
    "OPENCLAW_GATEWAY_TOKEN": "abc1...7890",
    "OPENCLAW_MGMT_API_KEY": "def4...1234"
  }
}
```

> Any value containing `TOKEN`, `KEY`, `SECRET`, `PASSWORD` is masked.

**Example:**

```bash
curl -H "Authorization: Bearer $MGMT_KEY" http://$VPS_IP:9998/api/env
```

---

### PUT /api/env/:key

Add or change an environment variable.

**Path parameter:** `key` = variable name (UPPER_SNAKE_CASE)

**Request body:**

```json
{
  "value": "your-value"
}
```

**Response:**

```json
{
  "ok": true,
  "key": "CUSTOM_VAR",
  "applied": true,
  "note": "Restart service for changes to take effect"
}
```

> **Note:**
> - Variable names must be uppercase, with underscores: `CUSTOM_VAR`
> - You **cannot** change `OPENCLAW_MGMT_API_KEY` via this endpoint
> - Restart service for changes to be effective

**Example:**

```bash
curl -X PUT -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"value": "my-value"}' \
  http://$VPS_IP:9998/api/env/MY_CUSTOM_VAR
```

---

### DELETE /api/env/:key

Delete an environment variable.

**Path parameter:** `key` = variable name

**Response:**

```json
{
  "ok": true,
  "key": "MY_CUSTOM_VAR",
  "removed": true
}
```

> **Protected variables** (cannot delete): `OPENCLAW_GATEWAY_TOKEN`, `OPENCLAW_MGMT_API_KEY`, `OPENCLAW_VERSION`, `OPENCLAW_GATEWAY_PORT`

**Example:**

```bash
curl -X DELETE -H "Authorization: Bearer $MGMT_KEY" \
  http://$VPS_IP:9998/api/env/MY_CUSTOM_VAR
```

---

## CLI Proxy

### POST /api/cli

Run an OpenClaw CLI command.

**Request body:**

```json
{
  "command": "models scan"
}
```

| Field     | Required | Description                               |
|-----------|----------|-------------------------------------------|
| `command` | Yes      | CLI command (no special chars: `;`, `&`, `|`, `` ` ``, `$`, `(`, `)`, `{`, `}`) |

**Success:**

```json
{
  "ok": true,
  "output": "Found 5 models:\n..."
}
```

**Failure:**

```json
{
  "ok": false,
  "output": "Error: command not found"
}
```

**Example:**

```bash
# Model scan
curl -X POST -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"command": "models scan"}' \
  http://$VPS_IP:9998/api/cli

# View config
curl -X POST -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"command": "config"}' \
  http://$VPS_IP:9998/api/cli

# View version
curl -X POST -H "Authorization: Bearer $MGMT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"command": "version"}' \
  http://$VPS_IP:9998/api/cli
```

---

## Common Error Codes

| HTTP Code | Description                | Response                                      |
|-----------|----------------------------|-----------------------------------------------|
| `200`     | Success                    | `{"ok": true, ...}`                           |
| `202`     | Accepted, processing (upgrade) | `{"ok": true, "message": "..."}`           |
| `400`     | Invalid data               | `{"ok": false, "error": "..."}`               |
| `401`     | Missing or invalid API key | `{"ok": false, "error": "Invalid or missing API key"}` |
| `403`     | Forbidden                  | `{"ok": false, "error": "Cannot modify..."}`  |
| `404`     | Not found                  | `{"ok": false, "error": "Not found"}`         |
| `429`     | Too many failed attempts   | `{"ok": false, "error": "Too many failed attempts..."}` |
| `500`     | Server error               | `{"ok": false, "error": "..."}`               |