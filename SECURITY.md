# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| 1.x     | ✅ Yes     |
| < 1.0   | ❌ No      |

Only the latest release receives security fixes. Staying current is your responsibility.

---

## Reporting a Vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Public disclosure before a patch is available puts every Ghost Tunnel deployment at risk. Don't do it.

### How to Report

Email: **security@lackadaisical-security.com**

Include:
- Description of the vulnerability
- Steps to reproduce (PoC code preferred — if you found it you can demonstrate it)
- Affected versions
- Your assessment of impact
- Any suggested fix (optional but appreciated)

PGP-encrypt your report if the details are sensitive. Public key available at: https://lackadaisical-security.com/.well-known/pgp

### Response SLA

| Milestone | Target |
|-----------|--------|
| Acknowledgment | 48 hours |
| Initial assessment | 5 business days |
| Patch + advisory | 30 days for most issues; 90 days for complex ones |
| Public disclosure | Coordinated with reporter |

If you don't hear back within 48 hours, follow up. Email gets lost.

---

## Coordinated Disclosure

The standard process:

1. Reporter sends details privately
2. Maintainer confirms/reproduces, assesses severity
3. Maintainer develops and tests patch
4. Reporter verifies the patch resolves the issue
5. Patch ships in a release
6. CVE assigned (if applicable)
7. Security advisory published
8. Reporter credited (unless they prefer anonymity)

90-day hard deadline: if a patch isn't shipped within 90 days of the report, the reporter is free to disclose at their discretion.

---

## Scope

### In Scope

- Key derivation or key handling flaws
- AES-256-GCM misuse (nonce reuse, tag truncation, etc.)
- Authentication bypass (admin token, JWT validation)
- Session management flaws (fixation, replay, improper revocation)
- Path traversal or vault escape
- Audit log tampering that evades `verifyIntegrity()`
- Timing attacks on auth comparisons
- Cryptographic weaknesses in the EMCP format
- Memory disclosure of key material
- Denial of service via resource exhaustion
- Dependency vulnerabilities with direct impact on Ghost Tunnel's security properties

### Out of Scope

- Vulnerabilities requiring physical access to an already-compromised machine
- Issues in Node.js itself (report to the Node.js security team)
- Social engineering attacks
- Findings from automated scanners with no demonstrated impact
- Issues in dependencies with no direct exploitation path through Ghost Tunnel

---

## Security Design Philosophy

Ghost Tunnel is designed around the principle that the vault files are **inert without the running server and the master password**. The security model is documented fully in [docs/SECURITY.md](docs/SECURITY.md). Reports that contradict or break this model are the highest priority.

---

## Hall of Fame

Researchers who responsibly disclosed vulnerabilities and helped make Ghost Tunnel more secure:

*(none yet — be the first)*
