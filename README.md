# 🔐 The Ghost Tunnel

> **Zero-knowledge encrypted MCP server** — prevents AI systems from reading files directly unless the authorized Ghost Tunnel MCP server is active and authenticated.

[![Node.js](https://img.shields.io/badge/Node.js-%3E%3D20-brightgreen)](https://nodejs.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue)](LICENSE)
[![Encryption: AES-256-GCM](https://img.shields.io/badge/Encryption-AES--256--GCM-orange)](docs/SECURITY.md)
[![KDF: Argon2id](https://img.shields.io/badge/KDF-Argon2id-red)](docs/SECURITY.md)

**Developer:** [Lackadaisical Security](https://lackadaisical-security.com)

---

## 📸 Dashboard

| Login | Overview |
|-------|----------|
| ![Login Screen](https://github.com/user-attachments/assets/48cd4f41-4d43-4147-9a09-f9d9ffff0af8) | ![Dashboard Overview](https://github.com/user-attachments/assets/5d86f30e-365e-4181-bc6f-8550225335fc) |

| Session Management | Encrypted Vault |
|-------------------|----------------|
| ![Sessions](https://github.com/user-attachments/assets/172b154f-f019-47ae-92a8-53f26940a306) | ![Vault](https://github.com/user-attachments/assets/8c6d167e-6960-43a3-9389-be842f926bbe) |

| Vault File Decrypt View | Audit Log |
|------------------------|-----------|
| ![File View](https://github.com/user-attachments/assets/708c34b7-55ae-4f47-ac71-db8c290dab9a) | ![Audit Log](https://github.com/user-attachments/assets/f2390fc3-59cc-4159-8e3d-f1a7c7ef313d) |

---

## 📋 Table of Contents

- [What Is The Ghost Tunnel?](#-what-is-the-ghost-tunnel)
- [Architecture Overview](#-architecture-overview)
- [Security Model](#-security-model)
- [Quick Start](#-quick-start)
- [Configuration](#-configuration)
- [MCP Tools](#-mcp-tools)
- [HTTP Management API](#-http-management-api)
- [Dashboard](#-dashboard-1)
- [CLI Scripts](#-cli-scripts)
- [Deployment](#-deployment)
- [Testing](#-testing)
- [Documentation](#-documentation)

---

## 🔍 What Is The Ghost Tunnel?

The Ghost Tunnel is a **Model Context Protocol (MCP) server** that acts as an encrypted proxy between an AI assistant (such as Claude) and your filesystem. Without the Ghost Tunnel running and authenticated, an AI has **no path** to read or write protected files — they are stored as opaque AES-256-GCM ciphertext.

### How it works

```
Claude (or any MCP client)
        │  stdio (MCP protocol)
        ▼
┌─────────────────────────────┐
│   Ghost Tunnel MCP Server   │  ← Validates JWT session on every call
│   src/server/mcp_server.js  │  ← Decrypts files on-the-fly in RAM
│                             │  ← Keys NEVER touch disk
└─────────┬───────────────────┘
          │  AES-256-GCM encrypt/decrypt
          ▼
  ./secure_vault/*.emcp        ← All files encrypted at rest (EMCP format)
```

**If the Ghost Tunnel process is not running**, the vault files are meaningless ciphertext. An AI or any process that reads them directly sees only random bytes — there is no offline key material to recover.

---

## 🏗 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    The Ghost Tunnel                      │
│                                                         │
│  ┌──────────────┐    ┌──────────────┐                  │
│  │  MCP Server  │◄──►│  HTTP API    │  :3302 localhost │
│  │  (stdio)     │    │  + WebSocket │                  │
│  └──────┬───────┘    └──────┬───────┘                  │
│         │                   │                           │
│  ┌──────▼───────────────────▼──────┐                   │
│  │       Encryption Manager        │                   │
│  │  Argon2id + HKDF → 4 sub-keys  │                   │
│  │  AES-256-GCM  •  BLAKE3 hash   │                   │
│  └──────┬──────────────────────────┘                   │
│         │                                               │
│  ┌──────▼──────┐  ┌──────────────┐  ┌───────────────┐ │
│  │  Vault      │  │  Session Mgr │  │  Audit Logger │ │
│  │  (EMCP fmt) │  │  JWT HS256   │  │  HMAC-SHA256  │ │
│  └─────────────┘  └──────────────┘  └───────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Key components

| Component | File | Responsibility |
|-----------|------|----------------|
| **Encryption Manager** | `src/crypto/encryption_manager.js` | AES-256-GCM, Argon2id, HKDF, BLAKE3, EMCP format |
| **MCP Server** | `src/server/mcp_server.js` | MCP stdio transport, 4 tools, key rotation |
| **HTTP API** | `src/server/http_server.js` | REST management API, WebSocket event stream |
| **Session Manager** | `src/session/session_manager.js` | JWT lifecycle, rate limiting, revocation |
| **Audit Logger** | `src/audit/audit_logger.js` | HMAC-SHA256 chained tamper-evident log |
| **FS Interceptor** | `src/interceptor/fs_interceptor.js` | Routes `/secure/` paths through the tunnel |
| **Entry Point** | `src/index.js` | Interactive password, graceful shutdown |

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full deep-dive.

---

## 🔒 Security Model

| Property | Implementation |
|----------|---------------|
| **Encryption** | AES-256-GCM (authenticated encryption with 128-bit tag) |
| **Key Derivation** | Argon2id (time=4, mem=64 MiB, p=4) + HKDF-SHA256 |
| **Key Hierarchy** | 4 domain-separated sub-keys: file, session, metadata, audit |
| **File Integrity** | BLAKE3 key fingerprint + GCM auth tag in 84-byte EMCP header |
| **Session Auth** | JWT HS256, 1-hour expiry, UUID jti, per-session revocation |
| **Admin Auth** | 32-byte random token (64 hex chars), `crypto.timingSafeEqual`, session-scoped |
| **Key Rotation** | New random AES-256-GCM key wrapped under master key, all files atomically re-encrypted |
| **Audit** | HMAC-SHA256 chain — each entry covers the previous hash; tampering is detectable |
| **RAM-only keys** | All key material lives only in `Buffer` objects; wiped with `randomFillSync`+`fill(0)` on shutdown |
| **Network** | HTTP API binds only to `127.0.0.1` — never externally reachable |

See [docs/SECURITY.md](docs/SECURITY.md) for the full threat model.

---

## ⚡ Quick Start

### Prerequisites

- **Node.js ≥ 20** (uses native `crypto.hkdfSync`, `crypto.timingSafeEqual`)
- `npm` (included with Node.js)
- Argon2 native addon: requires a C++ toolchain (`build-essential` / Xcode CLT)

### 1. Install

```bash
git clone https://github.com/The-Spectral-Operator/The-Ghost-Tunnel.git
cd The-Ghost-Tunnel
npm install
```

### 2. Configure

```bash
cp .env.example .env
# Edit .env — set NODE_ENV, optional VAULT_PATH overrides
# Do NOT set MASTER_PASSWORD in .env for production; use the interactive prompt
```

### 3. Start

```bash
node src/index.js
```

You will see:

```
[Ghost Tunnel] ═══════════════════════════════════════════════
[Ghost Tunnel]   The Ghost Tunnel - Encrypted MCP Server v1.0
[Ghost Tunnel]   Lackadaisical Security
[Ghost Tunnel] ═══════════════════════════════════════════════
Enter master password: ████████████  ← typed, not echoed

[Ghost Tunnel] ✓ Keys derived and stored in RAM
[HTTP] Management API: http://127.0.0.1:3302
[HTTP] Dashboard:      http://127.0.0.1:3302/
[HTTP] Admin token:    3af9b2...  ← copy this, it expires with the process
```

Open **http://127.0.0.1:3302** in your browser, paste the admin token, and click **Connect**.

### 4. Connect Claude Desktop

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "ghost-tunnel": {
      "command": "node",
      "args": ["/absolute/path/to/The-Ghost-Tunnel/src/index.js"],
      "env": {
        "MASTER_PASSWORD": "your-password-here"
      }
    }
  }
}
```

> **Production tip**: Use the interactive password prompt instead of the env var.  
> See [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) for the systemd service approach.

---

## ⚙ Configuration

All settings live in `config/default.json`. Environment variables override individual fields.

```json
{
  "server": {
    "host": "127.0.0.1",
    "port": 3302,
    "tls": false
  },
  "encryption": {
    "algorithm": "aes-256-gcm",
    "kdf": {
      "type": "argon2id",
      "timeCost": 4,
      "memoryCost": 65536,
      "parallelism": 4
    },
    "keyRotation": {
      "enabled": true,
      "intervalHours": 24
    }
  },
  "session": {
    "tokenExpiry": 3600,
    "maxConcurrentSessions": 5
  },
  "vault": {
    "path": "./secure_vault"
  },
  "audit": {
    "enabled": true,
    "logPath": "./logs/audit.log.enc",
    "rotation": "daily"
  }
}
```

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `MASTER_PASSWORD` | — | Master password (prefer interactive) |
| `VAULT_PATH` | `./secure_vault` | Override vault directory |
| `MANAGEMENT_PORT` | `3302` | HTTP management API port |
| `NODE_ENV` | `production` | `production` or `development` |

---

## 🛠 MCP Tools

The Ghost Tunnel exposes four tools over the MCP stdio transport. Each call is JWT-authenticated and audit-logged.

### `read_secure`

Decrypt and return a vault file.

```json
{
  "name": "read_secure",
  "arguments": {
    "path": "secrets/api-keys.json",
    "sessionToken": "<jwt>"
  }
}
```

### `write_secure`

Encrypt and store content in the vault.

```json
{
  "name": "write_secure",
  "arguments": {
    "path": "secrets/api-keys.json",
    "content": "{\"key\": \"sk-...\"}",
    "sessionToken": "<jwt>"
  }
}
```

### `list_vault`

List vault files at a given path prefix.

```json
{
  "name": "list_vault",
  "arguments": {
    "path": "secrets/",
    "sessionToken": "<jwt>"
  }
}
```

### `delete_secure`

Permanently delete a vault file.

```json
{
  "name": "delete_secure",
  "arguments": {
    "path": "secrets/old-key.json",
    "sessionToken": "<jwt>"
  }
}
```

---

## 🌐 HTTP Management API

All endpoints require the admin token either in the `x-admin-token` request header or as `adminToken` in the JSON body.

```
Base URL: http://127.0.0.1:3302
```

### Status

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/status` | None | Server health, uptime, session count |
| `GET` | `/api/config` | Admin | Safe configuration subset |

### Sessions

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/sessions` | Admin (body) | Create a new JWT session |
| `GET` | `/api/sessions` | Admin (header) | List all active sessions |
| `DELETE` | `/api/sessions/:id` | Admin (header) | Revoke a specific session |
| `DELETE` | `/api/sessions` | Admin (header) | Revoke ALL sessions |

**Create session request body:**
```json
{
  "adminToken": "3af9b2...",
  "userId": "claude-agent",
  "permissions": ["read:secure", "write:secure"]
}
```

### Vault

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/vault/files?path=` | Admin | List directory contents |
| `GET` | `/api/vault/files/content?path=` | Admin | Decrypt and return file content |
| `POST` | `/api/vault/files` | Admin | Encrypt and write a file |
| `DELETE` | `/api/vault/files` | Admin | Delete a vault file |

**Write file request body:**
```json
{
  "path": "secrets/api-keys.json",
  "content": "{\"key\": \"sk-...\"}",
  "encoding": "utf8"
}
```

### Audit

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/audit/entries?limit=100&operation=WRITE` | Admin | Recent log entries |
| `GET` | `/api/audit/verify` | Admin | Verify HMAC chain integrity |

### Keys

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/keys/rotate` | Admin | Trigger manual key rotation |

### WebSocket

```
WS /ws?token=<adminToken>
```

Broadcasts real-time events from the MCP server to all connected dashboard clients:

| Event type | Payload |
|-----------|---------|
| `connected` | Initial status snapshot |
| `vault.access` | `{ operation, path, userId, timestamp }` |
| `session.created` | `{ userId, sessionId }` |
| `session.revoked` | `{ sessionId }` |
| `keyrotation` | `{ filesRotated, timestamp }` |
| `auth.failure` | `{ resource, ip }` |

Full API reference: [docs/API.md](docs/API.md)

---

## 📊 Dashboard

Open **http://127.0.0.1:3302** after starting the server.

| Panel | Description |
|-------|-------------|
| **Overview** | Live stat cards (sessions, vault files, uptime, cipher suite) + real-time WebSocket event feed with auto-reconnect |
| **Sessions** | Create JWT sessions with per-permission checkboxes, copy token to clipboard, revoke individual sessions |
| **Vault** | Directory browser, upload/encrypt files (paste text or browse local file), inline decrypt+view, delete |
| **Audit Log** | Full operation history with operation-type filter buttons, one-click HMAC chain integrity verification |
| **Settings** | Configuration table, admin token copy, manual key rotation trigger, danger zone (revoke all) |

---

## 🖥 CLI Scripts

```bash
# Generate a cryptographically random Argon2id salt
npm run generate-salt

# Create a JWT session token from the command line
npm run create-session

# Encrypt a file into the vault
npm run encrypt-file -- --input ./plaintext.txt --out ./secure_vault/secret.txt

# Decrypt a vault file
npm run decrypt-file -- --input ./secure_vault/secret.txt

# Generate self-signed TLS certs for the management API
npm run gen-certs

# Reset/wipe the vault (DESTRUCTIVE — requires confirmation)
npm run reset-vault
```

---

## 🚀 Deployment

### systemd service (Linux production)

```ini
[Unit]
Description=Ghost Tunnel Encrypted MCP Server
After=network.target

[Service]
Type=simple
User=ghost-tunnel
WorkingDirectory=/opt/ghost-tunnel
ExecStart=/usr/bin/node /opt/ghost-tunnel/src/index.js
Restart=on-failure
RestartSec=5

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/opt/ghost-tunnel/secure_vault /opt/ghost-tunnel/logs

# Pass master password securely via systemd credential
EnvironmentFile=/etc/ghost-tunnel/env

[Install]
WantedBy=multi-user.target
```

### Claude Desktop (macOS/Windows)

```json
{
  "mcpServers": {
    "ghost-tunnel": {
      "command": "node",
      "args": ["/Users/you/The-Ghost-Tunnel/src/index.js"]
    }
  }
}
```

See [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) for Docker, TLS, and multi-user configurations.

---

## 🧪 Testing

```bash
# Run all tests
npm test

# Individual suites
npm run test:encryption   # AES-256-GCM, EMCP format, key derivation
npm run test:session      # JWT lifecycle, rate limiting, revocation
npm run test:security     # Path traversal, injection, timing attacks
```

The test suite covers:
- `test/encryption.test.js` — Crypto primitives, EMCP header, key rotation
- `test/session.test.js` — JWT creation/verification, session management
- `test/security.test.js` — Security edge cases, path traversal guards

---

## 📖 Documentation

| Document | Description |
|----------|-------------|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Deep-dive into system design, data flows, and component interactions |
| [docs/API.md](docs/API.md) | Complete REST API and WebSocket reference with examples |
| [docs/SECURITY.md](docs/SECURITY.md) | Security model, threat model, cryptographic design |
| [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) | Production deployment guide (systemd, Docker, TLS) |
| [BUILD_SPEC_ENCRYPTED_MCP_GHOST_TUNNEL.md](BUILD_SPEC_ENCRYPTED_MCP_GHOST_TUNNEL.md) | Original full build specification |

---

## 📁 Project Structure

```
The-Ghost-Tunnel/
├── src/
│   ├── index.js                    # Entry point — password prompt, startup, shutdown
│   ├── crypto/
│   │   └── encryption_manager.js  # AES-256-GCM, Argon2id, HKDF, BLAKE3, EMCP
│   ├── server/
│   │   ├── mcp_server.js          # MCP stdio transport, 4 tools, key rotation
│   │   └── http_server.js         # REST API, WebSocket, dashboard serving
│   ├── session/
│   │   └── session_manager.js     # JWT lifecycle, rate limiting, revocation
│   ├── audit/
│   │   └── audit_logger.js        # HMAC-SHA256 chained tamper-evident log
│   └── interceptor/
│       └── fs_interceptor.js      # Routes /secure/ paths through the tunnel
├── dashboard/
│   └── index.html                 # Single-file SPA (HTML/CSS/JS)
├── scripts/
│   ├── create_session.js          # CLI: create JWT session
│   ├── encrypt_file.js            # CLI: encrypt a file into the vault
│   ├── decrypt_file.js            # CLI: decrypt a vault file
│   ├── generate_salt.js           # CLI: generate Argon2id salt
│   ├── generate_certs.js          # CLI: self-signed TLS certs
│   └── reset_vault.js             # CLI: wipe vault (destructive)
├── test/
│   ├── encryption.test.js         # Crypto unit tests
│   ├── session.test.js            # Session management tests
│   └── security.test.js           # Security edge-case tests
├── config/
│   └── default.json               # Default configuration
├── docs/
│   ├── ARCHITECTURE.md            # System architecture
│   ├── API.md                     # API reference
│   ├── SECURITY.md                # Security model
│   └── DEPLOYMENT.md              # Deployment guide
├── secure_vault/                  # Encrypted file storage (git-ignored)
├── logs/                          # Encrypted audit logs (git-ignored)
├── certs/                         # TLS certificates (git-ignored)
├── .env.example                   # Environment variable template
└── package.json
```

---

## License

MIT © [Lackadaisical Security](https://lackadaisical-security.com)
