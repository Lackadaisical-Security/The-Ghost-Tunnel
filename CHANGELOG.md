# Changelog

All notable changes to The Ghost Tunnel are documented here.

Format: `[version] - YYYY-MM-DD`  
Categories: `Added` | `Changed` | `Fixed` | `Security` | `Removed` | `Deprecated`

---

## [1.0.0] - 2026-03-09

Initial production release. Full build from scratch.

### Added

**Crypto Core (`src/crypto/encryption_manager.js`)**
- AES-256-GCM encrypt/decrypt with 12-byte random IV per operation
- Argon2id key derivation (timeCost=4, memoryCost=64MiB, parallelism=4)
- HKDF-SHA256 domain-separated sub-key expansion (file, session, metadata, audit)
- BLAKE3 key fingerprint embedded in EMCP file header for key mismatch detection
- 84-byte EMCP file format: magic `GHST` + version + fingerprint + nonce + auth tag
- `rotateFileKey(blob, oldKey, newKey)` — atomic per-file re-encryption
- `secureWipe(buf)` / `secureWipeAll(keys)` — random-fill + multi-pass zero before GC
- `isTextBuffer(buf)` — heuristic for UTF-8 content detection on read

**MCP Server (`src/server/mcp_server.js`)**
- Extends `EventEmitter`; MCP stdio transport via `@modelcontextprotocol/sdk`
- Four tools: `read_secure`, `write_secure`, `list_vault`, `delete_secure`
- Real AES-256-GCM file key wrapping: 60-byte blob `[nonce|tag|ciphertext]` at `vault/.current_key.enc`
- 24-hour automatic key rotation timer (configurable)
- Path traversal guard: rejects `..`, null bytes, double slashes, escapes from vault root
- Emits `vault:access`, `session:created`, `session:revoked`, `keyrotation`, `auth:failure` events
- `initialize(password)` → Argon2id + HKDF derivation → load/generate wrapped file key
- `shutdown()` → cancel timers, wipe all key buffers, log SHUTDOWN audit entry

**Session Manager (`src/session/session_manager.js`)**
- JWT HS256 with UUID v4 `jti` for per-token revocation
- In-memory session map + revocation set (separate so revoked JWTs stay blocked post-expiry)
- `verifySession(token)` validates signature, expiry, revocation, and per-session permissions
- `recordFailedAttempt(ip)` / `clearRateLimit(ip)` — in-process rate limiting backing express-rate-limit
- `revokeAllSessions()` — bulk revoke for emergency lockdown
- Automatic expired-session cleanup on verify

**Audit Logger (`src/audit/audit_logger.js`)**
- Append-only JSONL log at `logs/audit.log.enc`
- HMAC-SHA256 per entry over canonical JSON including `prevHash` (chain integrity)
- `verifyIntegrity()` — full chain re-derivation; resets on `seq === 1` to handle multi-session log files
- Structured fields: `seq`, `timestamp`, `operation`, `resource`, `result`, `sessionId`, `clientInfo`, `extra`, `prevHash`
- `getRecentEntries(limit, operation)` — reverse-chronological filtered read

**File System Interceptor (`src/interceptor/fs_interceptor.js`)**
- Routes `/secure/` path prefix through MCP tunnel on `fs.readFile` / `fs.writeFile`

**HTTP Management API (`src/server/http_server.js`)**
- Localhost-only (`127.0.0.1`) Express app on port 3302
- One-time 32-byte admin token (`crypto.randomBytes(32)`), printed to stderr at startup
- `crypto.timingSafeEqual` on all admin token comparisons
- Rate limiting: 100 req/min global, 5 auth attempts/5min per IP
- Security headers: `X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection`, `Referrer-Policy`, `Cache-Control: no-store`
- CORS restricted to localhost origins
- REST endpoints: status, config, sessions CRUD, vault CRUD, audit entries + verify, key rotation
- WebSocket (`/ws`) authenticated on upgrade; broadcasts all MCP events in real time
- Serves `dashboard/index.html` at `/`

**Dashboard (`dashboard/index.html`)**
- Single-file SPA: HTML + CSS + vanilla JS, zero external dependencies
- Login screen: server URL + admin token input, connects to API + opens WebSocket
- **Overview panel**: stat cards (sessions, vault files, uptime, cipher suite) + live WebSocket event feed with auto-reconnect (exponential back-off, 1s → 30s cap)
- **Sessions panel**: full session table (ID, user, permissions, created, expires), create new session modal (userId + permission checkboxes), copy token to clipboard, per-session revoke
- **Vault panel**: directory browser with breadcrumb navigation, upload/encrypt (text paste or local file browse), inline decrypt+view modal with copy, delete with confirmation
- **Audit Log panel**: operation filter buttons (ALL / READ / WRITE / DELETE / AUTH / SYSTEM), HMAC chain integrity verification with pass/fail badge
- **Settings panel**: config table, admin token masked display + copy, manual key rotation with status feedback, danger zone (revoke all sessions)

**Entry Point (`src/index.js`)**
- Interactive master password via `/dev/tty` (falls back to stdin); input not echoed
- Non-fatal MCP stdio transport failure — HTTP API stays alive independently
- Graceful shutdown: SIGINT, SIGTERM, SIGHUP, uncaughtException, unhandledRejection → key wipe + exit

**CLI Scripts**
- `scripts/generate_salt.js` — print 32 cryptographically random bytes as hex
- `scripts/create_session.js` — create a JWT session from the command line
- `scripts/encrypt_file.js` — encrypt a plaintext file into the vault
- `scripts/decrypt_file.js` — decrypt a vault file to stdout
- `scripts/reset_vault.js` — wipe vault with confirmation prompt
- `scripts/generate_certs.js` — self-signed TLS certificate for management API

**Tests**
- `test/encryption.test.js` — AES-256-GCM round-trip, EMCP header, key derivation, key mismatch detection, secure wipe
- `test/session.test.js` — JWT create/verify, expiry, revocation, permission enforcement, rate limiting
- `test/security.test.js` — Path traversal, injection, timing-safe comparison, HMAC chain tamper detection

**Configuration**
- `config/default.json` — all defaults documented
- `.env.example` — full environment variable reference

**Documentation**
- `README.md` — project overview, quick start, API summary, screenshots
- `docs/ARCHITECTURE.md` — system design, crypto flows, data formats
- `docs/API.md` — complete REST + WebSocket reference
- `docs/SECURITY.md` — threat model, crypto primitives, known limitations
- `docs/DEPLOYMENT.md` — systemd, launchd, Claude Desktop, TLS, env vars
- `docs/CRYPTOGRAPHY.md` — cryptographic specification
- `docs/MCP_INTEGRATION.md` — MCP client integration guide
- `docs/CONFIGURATION.md` — full configuration reference
- `docs/SCRIPTS.md` — CLI scripts reference
- `docs/TROUBLESHOOTING.md` — diagnostic and fix guide
- `CHANGELOG.md` — this file
- `CONTRIBUTING.md` — contribution standards
- `CODE_OF_CONDUCT.md` — project conduct policy
- `SECURITY.md` — vulnerability disclosure policy

---

## [Unreleased]

Planned for future releases:

- Encrypted vault metadata index (fast file listing without decrypting headers)
- Multi-vault support (separate vaults per MCP client)
- Vault file compression before encryption
- TOTP / hardware key second factor for admin token
- REST API pagination for large vaults
- Prometheus metrics endpoint
- Structured stderr logging (JSON mode for log aggregators)
- Windows `dpapi` integration for master password storage
