# CachePass — project-wide instructions for Claude Code

The workspace root is NOT a repository. It holds four independent git repositories
(github.com/CachePass). `docs/`, `README.md`, `CLAUDE.md` and `.claude/` at the root
are symlinks into `cachepass-docs/`: edit and commit them there. Each repo has its own
`CLAUDE.md` with stack-specific rules; read it before working there. Detailed rules
live in `.claude/rules/`.

| Repo | Owns |
|---|---|
| `cachepass-backend/` | API, PostgreSQL schema, auth/sessions, envelope *validation*, Docker Compose deployment |
| `cachepass-webapp/` | UI, API client, **all vault cryptography** (`crypto/` Rust→WASM) |
| `cachepass-website/` | Marketing site only; links to the others via configured URLs |
| `cachepass-docs/` | `docs/` (ARCHITECTURE, SECURITY, THREAT_MODEL, CRYPTOGRAPHY — normative, DEVELOPMENT, ROADMAP), this file, `.claude/` |

## Boundaries (hard rules)
- Work inside the repo the task belongs to; run git commands with `git -C <repo>`.
- Never create a root git repo, a shared package, or a Cargo workspace spanning repos.
- No cross-repo imports. Contracts are documents: `cachepass-backend/openapi.json`
  (API) and `docs/CRYPTOGRAPHY.md` (crypto). Change the contract, then each side.
- Keep `docs/` and each repo's README in sync when behaviour changes.

## Security (summary — see `.claude/rules/security.md`)
- The backend never receives master passwords, vault keys or vault plaintext.
- No plaintext vault data or keys in browser storage, logs, URLs or analytics.
- Use established crypto libraries only; never weaken crypto, auth or CSRF to make
  something easier; never use fixed keys/nonces outside tests.
- Scope every data access by the session's principal; never trust client ownership ids.
- Make no security claims the code and docs do not support. No audit has happened yet.

## Commands
| Repo | Gates (must pass before committing) |
|---|---|
| backend | `cargo fmt --all --check` · `cargo clippy --workspace --all-targets --all-features -- -D warnings` · `cargo test --workspace` (needs `DATABASE_URL`, see `docs/DEVELOPMENT.md`) |
| webapp | `bun run check` · `bun run lint` · `bun run test` · `bun run build` (+ `bun run test:crypto`, `bun run test:e2e`) |
| website | `bun run check` · `bun run lint` · `bun run test` · `bun run build` |

Never claim a test passes unless you ran it. Fix warnings instead of suppressing them.

## Git
Independent history per repo · conventional commits · focused commits · never
force-push · never commit `.env`, secrets or backups · never overwrite user work.

## Cleanup
After building: `cargo clean` (backend, webapp/crypto), delete `build/` and
`.svelte-kit/output`, stop servers and `docker compose down` (never `-v`).
