# Architecture rules

- Four independent repos: backend, webapp, website (applications) and docs (design
  docs + contributor instructions, symlinked into the workspace root). No shared code,
  packages or workspaces. The workspace root is not a repo.
- Backend: modular monolith. `handlers` (HTTP only) → `service` (rules, transactions)
  → `repository` (SQL). Value types in `domain/`, adapters in `infrastructure/`.
  No microservices, no new crates unless a module is pure and independently testable.
- Webapp: static SPA (no SSR, no Node server). Crypto only through the worker-hosted
  WASM engine (`src/lib/crypto/engine.ts`). Decrypted state only in `stores/vault.svelte.ts`.
  Pure logic in `features/*`, presentation in `components/*`.
- Website: prerendered static site; never depends on the other repos.
- Contracts: API = `cachepass-backend/openapi.json` (generated; a test enforces
  freshness) mirrored by `cachepass-webapp/src/lib/api/`. Crypto = `docs/CRYPTOGRAPHY.md`,
  implemented in `cachepass-webapp/crypto`, validated by `cachepass-backend/crates/cachepass-envelope`.
- Deployment topology: Caddy (TLS) → webapp + API on one origin; PostgreSQL on an
  internal network only.
