# Roadmap

## MVP (v0.1 — implemented, unaudited)

- Three independent repositories: website, webapp (with Rust→WASM crypto), backend
- Zero-knowledge vault: Argon2id → HKDF → KEK wraps a random vault key; XChaCha20-Poly1305
  items/folders with AAD binding; padded plaintext; versioned envelopes; test vectors
  cross-checked with an independent implementation
- Accounts: registration, email verification, login, logout, logout-all, password
  change/reset, TOTP MFA with recovery codes, sessions and devices, audit log, account deletion
- Vault: logins, notes, cards, identities, SSH keys; folders, favourites, trash/restore/purge,
  duplicate, local search/sort/filter, TOTP codes, generator, strength meter, health report
- Optimistic concurrency, incremental sync with tombstones
- Self-hosting: Docker Compose with Caddy TLS, non-root containers, internal-only database,
  health checks, backup/restore scripts

## Next

- **Security review:** commission an independent audit of the crypto engine, the
  webapp's key handling and the backend's auth/session code (see below)
- Reproducible builds with published hashes of the webapp bundle and WASM;
  Subresource Integrity for all assets
- Vault key (DEK) rotation with full re-encryption; per-item keys to enable sharing
- Rollback / fork detection (client-side signed vault state or revision checkpoints)
- Encrypted export/import (CachePass format; CSV/JSON import from other managers)
- Passkeys / WebAuthn for account login; WebAuthn PRF as an unlock factor
- Email change with re-verification; admin CLI (disable user, reset MFA)
- Shared vaults / organisations with public-key crypto (X25519 key wrapping)
- Attachments (client-encrypted, chunked)
- Browser extension with autofill (signed code delivery also mitigates the
  malicious-server risk for the extension's users)
- Shared rate-limit store for multi-instance deployments
- Breach checking via k-anonymity (opt-in)

## Security audit milestones

1. **Internal review** (done for v0.1): threat model, OWASP Top 10 pass, dependency
   audit, automated tests for CSRF, IDOR, injection, session handling and crypto vectors.
2. **Cryptographic design review** by an external cryptographer: CRYPTOGRAPHY.md,
   key lifecycle, AAD construction, KDF policy.
3. **Implementation audit** of `cachepass-webapp/crypto`, `src/lib/crypto`, the vault
   store, and the backend auth/session/vault modules.
4. **Penetration test** of a reference deployment.
5. Publish reports and fixes; only then make production-assurance statements.
