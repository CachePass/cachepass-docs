---
name: security-review
description: Review CachePass changes against the threat model, crypto spec and OWASP Top 10; report verified findings with file:line and fixes.
---

Checklist (verify each against the actual code path before reporting):
1. Zero-knowledge: could any plaintext, key or master password reach the server,
   logs, browser storage or URLs? Are new fields inside the ciphertext?
2. Crypto: matches `docs/CRYPTOGRAPHY.md` (AAD fields/order, KDF bounds, padding,
   random nonces, zeroization, version bytes)? Can a malicious server weaken it?
3. Access control (A01): every query scoped by the session principal; foreign ids → 404.
4. Auth/session (A07): rate limits, enumeration safety, token hashing, rotation,
   expiry, MFA replay, pending-session restrictions.
5. Injection/XSS (A03): static SQL with binds; no `{@html}`; safe links.
6. CSRF/CORS: unsafe methods behind the CSRF middleware; origins allow-listed.
7. Misconfiguration (A05): CSP, headers, cookies, Docker (non-root, read-only,
   no published DB/debug ports), secrets only via env.
8. Logging (A09): security events audited; no secrets logged.
9. Dependencies (A06): `cargo audit`, `bun audit`; justify new dependencies.
Report: severity, file:line, attack scenario, fix. Then fix and add a regression test.
