# Contributing to The Ghost Tunnel

This is a security project. The bar is high by design.

---

## Before You Touch Anything

Read the full codebase. All of it.  
Read `docs/ARCHITECTURE.md`, `docs/CRYPTOGRAPHY.md`, and `docs/SECURITY.md`.  
If you haven't, your PR will be closed without review.

Security code is not a place for drive-by contributions. Every line that touches crypto, key handling, auth, or session management will be scrutinized. If you don't understand what you changed and why, don't submit it.

---

## What Gets Accepted

- **Bug fixes** — must include a regression test demonstrating the bug before and the fix after
- **Security fixes** — disclose privately first (see [SECURITY.md](SECURITY.md)), then submit the patch
- **Performance improvements** — must not weaken any security property; must include benchmarks
- **New features** — open an issue first; unsolicited feature PRs for large changes get closed
- **Documentation** — factual corrections and improvements are always welcome; editorial changes are not

---

## What Gets Rejected

- Anything that weakens the threat model
- Padding the feature set without a real use case
- Code that doesn't understand the crypto it's touching
- "I replaced X with Y because Y is more popular" without a concrete security or correctness justification
- Test coverage theater (tests that test nothing meaningful)
- Formatting-only PRs

---

## Code Standards

### Style

- Node.js / plain JavaScript (ES2022+, no transpilation)
- `'use strict'` at the top of every module
- 2-space indent, single quotes, semicolons
- Keep lines under 120 characters
- No external dependencies unless strictly necessary and security-audited
- Run `npm test` before submitting — all tests must pass

### Crypto Rules

- **Never** roll your own primitives. Use Node.js `crypto` or `@noble/hashes`.
- **Never** use `Math.random()` for anything security-related. `crypto.randomBytes()` only.
- **Never** use `==` for token/key comparison. `crypto.timingSafeEqual()` only.
- **Never** log key material, tokens, or passwords — not even partially.
- **Never** store key material in a JS object property that gets JSON-serialized.
- All IVs/nonces must be randomly generated per operation — no counters, no timestamps.
- If you add a new key usage, add a new HKDF domain label. No key reuse across contexts.

### Error Handling

- Crypto failures must not leak oracle information in error messages.
- Auth failures must return identical responses regardless of which check failed (timing + content).
- Never `console.log` inside production paths. Use `process.stderr.write` with the `[Ghost Tunnel]` prefix.

### Tests

- Every new code path needs a test.
- Security tests: include both the happy path and the attack path.
- Crypto tests: test known-answer vectors where possible.
- Don't mock crypto primitives in tests — test the real thing.

---

## Submitting a Pull Request

1. Fork the repo and create a branch: `fix/short-description` or `feat/short-description`
2. Make your changes. Keep commits atomic and well-described.
3. Run the full test suite: `npm test`
4. Run `node --check src/**/*.js scripts/*.js` — no syntax errors
5. Open the PR against `main`. Describe:
   - What the bug/problem is
   - What your fix/change does
   - Why it's the right approach
   - What tests cover it
6. PRs with no description get closed.

---

## Security Vulnerabilities

Do not open a public issue for a security vulnerability.  
Follow the process in [SECURITY.md](SECURITY.md).

---

## Skill Level

This project doesn't handhold. If something isn't documented and you can't figure it out from the source, that's intentional — the source code is the ground truth. Read it.

Questions that are answered in the docs or source will be closed without response.
