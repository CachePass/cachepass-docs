# Development

CachePass is four independent repositories under https://github.com/CachePass.
Clone them side by side and link the shared docs into the workspace root (the
backend's Compose file builds nothing from the others; it only references the
webapp image, and `docker build ../cachepass-webapp` is a convenience):

```sh
mkdir CachePass && cd CachePass
for r in docs backend webapp website; do git clone https://github.com/CachePass/cachepass-$r.git; done
for f in docs README.md CLAUDE.md .claude; do ln -s cachepass-docs/$f $f; done
```

## Prerequisites

| Tool | Version | Needed for |
|---|---|---|
| Rust (rustup) | stable ≥ 1.85, edition 2024 | backend; rebuilding the webapp's WASM engine |
| `wasm32-unknown-unknown` target | — | `rustup target add wasm32-unknown-unknown` (WASM rebuilds only) |
| `wasm-bindgen-cli` | **0.2.130** (must match `crypto/Cargo.toml`) | `cargo install wasm-bindgen-cli --version 0.2.130 --locked` |
| Bun | ≥ 1.4 | webapp, website |
| Docker + Compose | recent | PostgreSQL, Mailpit, production images |
| Chromium for Playwright | via `bunx playwright install chromium` | browser and e2e tests |

## Local stack

```sh
# 1. Backend services
cd cachepass-backend
docker compose -f docker-compose.dev.yml up -d
export DATABASE_URL=postgres://cachepass:cachepass@127.0.0.1:55433/cachepass

# 2. API on :8080 (see the backend README for all variables)
CACHEPASS_PUBLIC_URL=http://localhost:5173 \
CACHEPASS_SECRET_KEY=$(openssl rand -base64 32) \
CACHEPASS_SMTP_HOST=127.0.0.1 CACHEPASS_SMTP_PORT=51025 CACHEPASS_SMTP_TLS=none \
CACHEPASS_BIND_ADDR=127.0.0.1:8080 CACHEPASS_LOG_FORMAT=pretty cargo run

# 3. Webapp on :5173 (proxies /api to :8080)
cd ../cachepass-webapp && bun install && bun run dev

# 4. Website on :5174 (independent)
cd ../cachepass-website && bun install && bun run dev -- --port 5174
```

Verification and reset emails land in Mailpit: http://127.0.0.1:58025.

## Quality gates

| Repo | Commands |
|---|---|
| backend | `cargo fmt --all --check` · `cargo clippy --workspace --all-targets --all-features -- -D warnings` · `cargo test --workspace` (needs `DATABASE_URL`) · `cargo audit` |
| webapp | `bun run check` · `bun run lint` · `bun run test` · `bun run build` · `bun run test:crypto` · `bun run test:e2e` (needs the API with `CACHEPASS_PUBLIC_URL=http://localhost:4173`, see webapp README) |
| website | `bun run check` · `bun run lint` · `bun run test` · `bun run build` |

## Changing a contract

- **API:** change the backend, add integration tests, regenerate
  `cachepass-backend/openapi.json`, then update `cachepass-webapp/src/lib/api/`.
  Commit in each repository separately.
- **Crypto format:** update `docs/CRYPTOGRAPHY.md` first, bump the version byte,
  regenerate vectors (`crypto/src/vectors.rs`), keep decrypting old versions, and
  update `cachepass-backend/crates/cachepass-envelope`. Rebuild the WASM
  (`bun run build:wasm`) and commit the output.

## Cleanup

Rust build output is large. After working: `cargo clean` in `cachepass-backend`
and `cachepass-webapp/crypto`; delete `build/` and `.svelte-kit/output` in the
SvelteKit repos; `docker compose -f docker-compose.dev.yml down` (keeps the volume).
