# Testing rules

- Run the repo's gates before committing (see root CLAUDE.md). Report real results only.
- Backend: unit tests next to code; integration tests in `tests/` with `#[sqlx::test]`
  (real PostgreSQL). Every endpoint needs happy-path, validation, authorization (IDOR),
  and conflict tests; security-relevant behaviour (CSRF, rate limits, expiry) gets a test.
- Webapp: pure logic → `*.spec.ts` (node); components → `*.svelte.spec.ts` (Chromium);
  crypto → `src/lib/crypto/*.browser.spec.ts` against the spec vectors and the
  independent @noble implementation; flows → `e2e/*.e2e.ts` against a real backend.
- Crypto changes: Rust tests (`cargo test` in `crypto/`), vectors pinned in
  `crypto/src/vectors.rs`, and the envelope crate tests in the backend.
- A test that cannot fail is not a test: when adding a security test, check that it
  fails if the protection is removed.
