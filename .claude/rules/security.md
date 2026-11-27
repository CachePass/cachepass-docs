# Security rules

Zero-knowledge
- Master password, KEK, vault key and vault plaintext never leave the browser.
  Item type/name/folder/favourite are inside the ciphertext.
- Nothing decrypted or key-related in localStorage/sessionStorage/IndexedDB/cookies/URLs.
- Lock = terminate the crypto worker + clear stores + clear pending clipboard.

Crypto
- Only RustCrypto primitives inside `cachepass-webapp/crypto`. No hand-written
  primitives, no protocol changes without updating CRYPTOGRAPHY.md, version bytes and vectors.
- Random nonces from the CSPRNG; fixed keys/nonces only in `#[cfg(test)]` code.
- Clients must reject KDF parameters outside the documented bounds.

Backend
- Every query scoped by the session's `user_id`/`vault_id`; foreign ids → 404.
- Parameterised SQL with `&'static str` only. Transactions for multi-step writes.
- Argon2id for account passwords; tokens stored as SHA-256 hashes; constant-time compares.
- Rate-limit every credential/email/token endpoint. Keep responses enumeration-safe.
- Never log secrets, bodies, cookies, headers or query strings. Return `AppError`s only.

Web
- Strict CSP (no `unsafe-eval`, no inline handlers, no third-party origins); render
  user data as text, never `{@html}`. HttpOnly + Secure + SameSite=Strict cookies;
  CSRF token + Origin checks on unsafe methods.

Never weaken a control to make a feature or test easier. Never claim audited security.
