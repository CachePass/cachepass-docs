---
name: feature-development
description: Implement a CachePass feature end to end across the right repositories (contract first, then backend, then webapp), with tests and docs.
---

1. Identify which repos the feature touches and read their `CLAUDE.md`.
2. If it changes data the server stores, decide what is encrypted (default: everything
   about vault content) and update `docs/` (ARCHITECTURE / THREAT_MODEL / CRYPTOGRAPHY).
3. Backend (if needed): migration → repository → service → handler with
   `#[utoipa::path]` → register in `routes/` and `routes/openapi.rs` → integration tests
   (happy path, validation, IDOR, conflicts) → regenerate `openapi.json`.
4. Webapp: Zod schema + endpoint in `src/lib/api/` → store/feature logic with unit tests
   → small components → e2e coverage for the user flow.
5. Run every affected repo's gates; fix failures; update READMEs.
6. Commit per repo with conventional commits.
