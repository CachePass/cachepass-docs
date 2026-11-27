# Git workflow

- Each repo has independent history: `git -C cachepass-<name> …`. Never init git at the root.
- Root `docs/`, `README.md`, `CLAUDE.md`, `.claude/` are symlinks: commit changes in `cachepass-docs`.
- Remotes: `https://github.com/CachePass/cachepass-<name>`; default branch `main`.
- Conventional commits (`feat(scope):`, `fix:`, `docs:`, `test:`, `build:`, `chore:`);
  one logical change per commit; contract changes are committed in each affected repo.
- Never force-push, never rewrite pushed history, never overwrite user changes.
- Never commit `.env`, secrets, database dumps, `target/`, `build/` or test output.
- Regenerated artefacts that ARE committed: `cachepass-backend/openapi.json`,
  `cachepass-webapp/src/lib/crypto/wasm/`.
