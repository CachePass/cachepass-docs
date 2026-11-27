# CachePass

An open-source, self-hosted, **zero-knowledge** password manager. Vault data is
encrypted and decrypted in the browser by a Rust → WebAssembly engine; the
server stores ciphertext only and never receives your master password.

> **Status: pre-1.0, not independently audited.** The design is documented so it
> can be reviewed. Do not rely on CachePass for secrets you cannot afford to lose
> or expose until an external security review is complete
> ([ROADMAP](docs/ROADMAP.md#security-audit-milestones)).

## Repositories

CachePass is split into four independent repositories under
[github.com/CachePass](https://github.com/CachePass). This one, `cachepass-docs`,
holds the cross-cutting design documents and project-wide contributor
instructions; the other three are the deployable applications:

| Repository | What it is | Stack |
|---|---|---|
| [`cachepass-website/`](https://github.com/CachePass/cachepass-website) | Public marketing / documentation site (static) | SvelteKit 3, Svelte 5, Tailwind v4, shadcn-svelte, Bun |
| [`cachepass-webapp/`](https://github.com/CachePass/cachepass-webapp) | The password manager client + the vault crypto engine (`crypto/`) | SvelteKit 3 SPA, Rust → WASM, Zod, Bun |
| [`cachepass-backend/`](https://github.com/CachePass/cachepass-backend) | API, database, auth/sessions, deployment (Docker Compose, Caddy) | Rust, Axum, Tokio, SQLx, PostgreSQL, Utoipa |
| [`cachepass-docs/`](https://github.com/CachePass/cachepass-docs) (this repo) | [`docs/`](docs/): architecture, security, threat model, cryptography (normative), development, roadmap; project-wide `CLAUDE.md` and `.claude/` | Markdown |

The repositories share no code. Contracts are documents: the API contract is
`cachepass-backend/openapi.json` (generated from the code) and the crypto
contract is [docs/CRYPTOGRAPHY.md](docs/CRYPTOGRAPHY.md).

## How it works

```
Browser                                              Server
───────────────────────────────────────────────      ─────────────────────────────
master password ─NFC─► Argon2id ─► HKDF ─► KEK
                                            │ unwrap
wrapped vault key ◄─────────────────────────┼──────  GET /vault/key-envelope
                                            ▼
                                   vault key (WASM memory only)
                                            │ XChaCha20-Poly1305, AAD = vault id ‖ item id
items ◄── decrypt ──── ciphertext ◄─────────┴──────  GET /vault/sync   (ciphertext only)
```

- The **account password** (sign-in) and the **master password** (encryption)
  are different secrets. Resetting the account password by email cannot decrypt
  the vault. **Losing the master password means losing the vault** — there is no
  recovery backdoor.
- Details: [ARCHITECTURE](docs/ARCHITECTURE.md) · [CRYPTOGRAPHY](docs/CRYPTOGRAPHY.md) ·
  [THREAT_MODEL](docs/THREAT_MODEL.md) · [SECURITY](docs/SECURITY.md)

### What zero-knowledge does *not* protect against

Whoever serves the webapp's JavaScript/WASM can ship code that captures the
master password. A compromised device or browser defeats any browser-based
password manager. When you self-host, you are the party your users trust. See
[THREAT_MODEL §5](docs/THREAT_MODEL.md#5-residual-risks-accepted-documented).

## Features (v0.1)

Logins, secure notes, cards, identities and SSH keys · folders, favourites, trash ·
local search · TOTP codes · password/passphrase generator · strength meter and vault
health report · auto-lock · clipboard auto-clear · TOTP 2FA for sign-in with
recovery codes · session and device management · activity log · optimistic
concurrency and incremental sync · command palette and keyboard shortcuts ·
light/dark themes.

## Local workspace layout

Clone all four repositories into one folder and link the shared files into it, so
the docs and project-wide instructions sit at the workspace root:

```sh
mkdir CachePass && cd CachePass
for r in docs backend webapp website; do git clone https://github.com/CachePass/cachepass-$r.git; done
for f in docs README.md CLAUDE.md .claude; do ln -s cachepass-docs/$f $f; done
```

The workspace folder itself is not a repository.

## Quick start (development)

See [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md). In short:

```sh
cd cachepass-backend && docker compose -f docker-compose.dev.yml up -d && cargo run   # with the env vars from the README
cd cachepass-webapp  && bun install && bun run dev                                   # http://localhost:5173
cd cachepass-website && bun install && bun run dev                                   # marketing site
```

## Self-hosting

```sh
git clone https://github.com/CachePass/cachepass-backend.git
git clone https://github.com/CachePass/cachepass-webapp.git
cd cachepass-backend
cp .env.example .env            # domain, ACME email, secrets, SMTP
docker build -t cachepass-webapp:local ../cachepass-webapp
docker compose up -d --build
```

Full guide including backups, restores and upgrades:
[cachepass-backend README](https://github.com/CachePass/cachepass-backend#self-hosting).

## License

Each repository is licensed AGPL-3.0-only (see its `LICENSE`).
