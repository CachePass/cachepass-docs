# CachePass Threat Model

Scope: CachePass v0.1, the self-hosted web deployment (reverse proxy, webapp,
backend API, PostgreSQL). Method: asset- and boundary-driven, STRIDE per
boundary. Status: initial model, maintained by the project, **not externally
reviewed**.

## 1. Assets

| ID | Asset | Where it lives | Sensitivity |
|---|---|---|---|
| A1 | Master password | User memory; briefly in browser memory during unlock/change | Critical |
| A2 | Vault key (DEK), KEK | WASM memory while unlocked; KEK only during derivation | Critical |
| A3 | Vault plaintext (items, folder names, TOTP seeds) | Browser memory while unlocked | Critical |
| A4 | Vault ciphertext + wrapped vault key | PostgreSQL, backups, network | High (offline-guessable with A1 attempts) |
| A5 | Account password | Browser during login; server only as Argon2id hash | High |
| A6 | Session tokens, CSRF tokens | HttpOnly cookies; server stores SHA-256 hashes | High |
| A7 | Email verification / password reset tokens | Email; server stores SHA-256 hashes | High (reset → account takeover, not vault access) |
| A8 | Account MFA secret, recovery codes | Server (secret encrypted with server key; codes hashed) | High |
| A9 | Metadata: email, item count, sizes, timestamps, IPs, user agents, audit log | PostgreSQL | Medium |
| A10 | Server secret key (`CACHEPASS_SECRET_KEY`), DB credentials | Server environment | High |
| A11 | Integrity of delivered webapp code (JS/WASM) | Webapp container / reverse proxy | Critical: controls A1–A3 |

## 2. Threat actors

| Actor | Capability |
|---|---|
| T1 Remote unauthenticated attacker | Sends arbitrary HTTP requests; credential stuffing; enumeration |
| T2 Authenticated malicious user | Valid account; attempts IDOR, resource exhaustion |
| T3 Network attacker | Observes or modifies traffic if TLS is absent or misconfigured |
| T4 Database thief | Obtains a DB dump or backup (stolen disk, leaked backup, SQL injection) |
| T5 Malicious or compromised server operator | Full control of the backend, DB and the files served to browsers |
| T6 Malicious web content / XSS | Script injected into the webapp origin |
| T7 Local attacker / malware on the user's device | Reads memory, keystrokes, clipboard, extensions |
| T8 Supply-chain attacker | Compromises a dependency (npm, crates.io, container image) |

## 3. Trust boundaries and attack surfaces

```
[User device / browser] ──TLS── [Reverse proxy] ──┬── [Webapp static files]
        │  B1                          B2         └── [Backend API] ──B3── [PostgreSQL]
        │                                                 │
   B0: WASM ↔ JS                                       B4: SMTP relay
```

| Boundary | Surface |
|---|---|
| B0 WASM ↔ JS | Key handles, plaintext buffers crossing into JS |
| B1 Browser ↔ internet | All `/api/v1/*` endpoints, cookies, static asset delivery |
| B2 Proxy ↔ services | Forwarded headers (`X-Forwarded-For`), TLS termination |
| B3 API ↔ database | SQL queries, credentials, migrations |
| B4 API ↔ SMTP | Verification and reset emails |

## 4. Threats and mitigations

### 4.1 Confidentiality of vault data

| Threat | Actor | Mitigation |
|---|---|---|
| Server reads vault contents | T5, T4 | Client-side XChaCha20-Poly1305 encryption; server stores ciphertext only; master password never sent ([CRYPTOGRAPHY.md](CRYPTOGRAPHY.md)) |
| Offline guessing of master password from dump | T4, T5 | Argon2id (64 MiB, t = 3 by default; client-enforced minimums); unique per-vault salt; guidance on strong master passwords |
| Server swaps or reuses ciphertexts | T5 | AAD binds vault id, item/folder id and object type |
| Server downgrades KDF parameters | T5 | Client rejects out-of-bounds parameters; parameters bound into wrap AAD |
| Plaintext leaked to browser storage | T7, T6 | No decrypted data or keys in localStorage/sessionStorage/IndexedDB/cookies; only non-sensitive UI preferences are persisted |
| Plaintext leaked via logs/analytics | T5 | Server never has plaintext; no analytics or third-party scripts in the webapp; server logs exclude bodies, tokens and passwords |
| Clipboard exposure | T7 | Copy only on explicit user action; clipboard cleared after 30 s (best effort: only if unchanged and the page still has focus) |
| Unattended unlocked vault | T7 | Auto-lock on idle (default 15 min), manual lock, lock on logout; in-flight requests that finish after a lock are discarded (lock generation check) |

### 4.2 Authentication and sessions

| Threat | Actor | Mitigation |
|---|---|---|
| Credential stuffing / brute force | T1 | Per-IP (IPv6 per /64) and per-email/per-account rate limits on login, registration, reset, MFA; Argon2id password hashing; optional TOTP MFA; MFA attempts reserved atomically (no parallel-guess bypass) |
| User enumeration | T1 | Identical responses for register/forgot-password/resend regardless of existence (vault-id conflicts are checked before the email); dummy Argon2 verification for unknown emails |
| Session theft via XSS | T6 | Session cookie is HttpOnly, Secure, SameSite=Strict, `__Host-` prefixed; tokens never in localStorage |
| CSRF | T1 | SameSite=Strict, Origin/`Sec-Fetch-Site` validation, double-submit CSRF token with constant-time comparison on every unsafe method |
| Stolen session reused indefinitely | T1, T7 | Idle (30 min) and absolute (7 day) expiry; session list with per-session and per-device revocation; logout-all; password change revokes other sessions |
| Session tokens in a DB dump | T4 | Only SHA-256 hashes stored |
| Password reset abuse | T1 | Single-use, 30-minute, hashed tokens; reset revokes all sessions; **reset cannot decrypt the vault** |
| MFA bypass | T1 | TOTP verified with ±1 step window and replay prevention (last used step stored); recovery codes hashed and single-use; MFA required again after password reset |

### 4.3 Authorization

| Threat | Actor | Mitigation |
|---|---|---|
| IDOR on items/folders/sessions/devices | T2 | Every query is scoped by the authenticated user's vault/user id taken from the session — never from the request; foreign ids return 404 (no existence oracle) |
| Lost updates / race conditions | T2 | Optimistic concurrency via `revision`; conflicting writes return 409 with the current revision |
| Resource exhaustion | T1, T2 | Global body limit (1 MiB), per-object ciphertext limits, item count limit per vault (10 000), pagination caps, bounded concurrent Argon2 hashing (503 when saturated), reset tokens validated before hashing |
| Vault destruction with a stolen session | T1, T7 | Replacing the key envelope requires the account password (+ 2FA code); deletions are soft by default (trash) |

### 4.4 Injection and web security

| Threat | Actor | Mitigation |
|---|---|---|
| SQL injection | T1, T2 | Parameterised SQLx queries only; no string-built SQL |
| XSS | T6 | Svelte auto-escaping; no `{@html}` with user data; strict CSP (`script-src 'self' 'wasm-unsafe-eval'` + hashes, no inline handlers, no third-party origins); decrypted values are rendered as text only |
| Clickjacking | T1 | `frame-ancestors 'none'`, `X-Frame-Options: DENY` |
| Mixed content / downgrade | T3 | TLS via reverse proxy; HSTS; `Secure` cookies; webapp refuses non-secure contexts |
| Information disclosure in errors | T1 | Uniform error envelope; database errors and internals logged server-side only, never returned |

### 4.5 Infrastructure

| Threat | Actor | Mitigation |
|---|---|---|
| Exposed database | T1 | PostgreSQL only on an internal Docker network, no published port |
| Container breakout impact | T1 | Non-root containers, read-only root filesystem where possible, `no-new-privileges` |
| Secret leakage via repo | T8 | `.env` ignored; `.env.example` contains no secrets; secret scanning in CI |
| Dependency compromise | T8 | Lockfiles committed; `cargo audit` and `bun audit`; minimal dependency set |
| Spoofed client IP | T1 | Forwarded headers trusted only when `CACHEPASS_TRUST_PROXY=true` (proxy on the internal network) |

## 5. Residual risks (accepted, documented)

1. **Malicious code delivery (T5, T8).** Whoever controls the server or the
   webapp build can ship JavaScript/WASM that captures the master password on
   the next unlock. Zero-knowledge storage does **not** protect against this.
   Partial mitigations: self-hosting (you are the operator), reproducible
   builds and published hashes (roadmap), a future browser extension or
   desktop client with signed code.
2. **Compromised device or browser (T7).** Malware, malicious extensions or
   keyloggers defeat all client-side protections.
3. **Weak master password.** An attacker holding a DB dump can mount offline
   guessing; Argon2id slows this down but does not prevent it.
4. **Rollback.** The server can serve older authentic ciphertexts (for example
   restoring a deleted item, or reverting an edit). No client-side
   anti-rollback state exists yet.
5. **Metadata.** Email address, item count, approximate item sizes, timestamps,
   IP addresses and user agents are visible to the operator.
6. **JS memory.** Decrypted strings cannot be reliably zeroized in JavaScript.
7. **No DEK rotation.** Old wrapped keys plus old master passwords keep working
   against current data (see CRYPTOGRAPHY.md §8.1).
8. **Single-instance rate limiting.** Limits are held in process memory; they
   reset on restart and are not shared across multiple API replicas.
9. **Targeted login lockout.** The per-email login limit lets anyone temporarily
   delay a specific user's sign-in (10 attempts / 15 min). Accepted to slow online guessing.
10. **Edge proxy runs as root in its container** (to bind :80/:443) with every other
   capability dropped, `no-new-privileges` and a read-only root filesystem.
11. **`chrono` is flagged unmaintained** (RUSTSEC-2026-0543, informational, no known
   vulnerability). Migration to `time` is planned.
12. **Email as reset channel.** Whoever controls the user's mailbox can reset
   the *account* password (but cannot decrypt the vault). MFA mitigates this.

## 6. Review cadence

Update this model whenever an endpoint, a stored field, a cryptographic
operation or a deployment component is added or changed. An external security
review is required before any production-assurance claim (see
[ROADMAP.md](ROADMAP.md)).
