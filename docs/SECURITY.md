# Security Policy

## Status

CachePass is **pre-1.0 and has not had an independent security audit.** The
design is documented openly ([CRYPTOGRAPHY.md](CRYPTOGRAPHY.md),
[THREAT_MODEL.md](THREAT_MODEL.md)) so it can be reviewed. Until an external
review is complete, do not make production-assurance claims about CachePass and
do not store secrets whose compromise you cannot tolerate.

## Reporting a vulnerability

Please **do not open public issues for security problems.** Until the project
publishes a dedicated security contact, report vulnerabilities privately through a GitHub
security advisory on the affected repository
(`https://github.com/CachePass/<repository>/security/advisories/new`). Include:

- the affected repository and commit;
- reproduction steps or a proof of concept;
- the impact as you understand it.

We aim to acknowledge reports within 7 days and to agree a disclosure timeline
with you. Coordinated disclosure is appreciated; we will credit reporters who
wish to be credited.

## Security assumptions

CachePass's guarantees hold only if:

1. **The code delivered to the browser is honest.** A malicious or
   compromised server, build pipeline or dependency can capture the master
   password. Self-hosting means *you* are the party users must trust.
2. **The user's device and browser are not compromised.** No browser-based
   password manager can defend against malware or hostile extensions.
3. **TLS is correctly configured.** CachePass must be served over HTTPS
   (except on `localhost` during development).
4. **The master password is strong and unique**, and different from the
   account password. Losing it means losing the vault: there is no recovery.
5. **The operator protects server secrets** (`CACHEPASS_SECRET_KEY`, database
   credentials, SMTP credentials) and backups.

## Security policies for contributors

- Vault plaintext, master passwords and vault keys must never reach the
  backend, logs, analytics, browser storage or URLs.
- Never log request bodies, cookies, tokens, passwords or ciphertext.
- Every data-access query must be scoped by the authenticated principal taken
  from the session, never by an id supplied by the client.
- Use only established cryptographic libraries; do not implement primitives
  or invent protocols. Changes to the crypto format require updating
  CRYPTOGRAPHY.md, the version byte and the test vectors.
- Never weaken a security control to make a test or a feature easier.
- New dependencies need justification; run `cargo audit` / `bun audit`.
- Do not commit secrets. `.env` files are ignored; use `.env.example`.

## Security controls overview

| Area | Control |
|---|---|
| Vault confidentiality | Client-side Argon2id + HKDF + XChaCha20-Poly1305 via WASM |
| Account passwords | Argon2id (server), rate limiting, optional TOTP MFA with recovery codes |
| Sessions | Random 256-bit tokens, SHA-256 hashed at rest, HttpOnly/Secure/SameSite=Strict `__Host-` cookies, idle + absolute expiry, revocation |
| CSRF | SameSite=Strict + Origin / Sec-Fetch-Site checks + double-submit token |
| Authorization | Session-derived ownership on every query; foreign ids → 404 |
| Concurrency | Optimistic revisions; 409 on conflict |
| Input | Typed deserialisation, `validator` rules, body size limits, ciphertext structure validation |
| Transport/browser | HSTS, strict CSP, `frame-ancestors 'none'`, `nosniff`, `Referrer-Policy: no-referrer`, `Permissions-Policy` |
| Infrastructure | Non-root containers, internal-only database network, health checks, no debug ports |

## Supported versions

Only the latest commit on `main` of each repository receives security fixes
during the pre-1.0 phase.
