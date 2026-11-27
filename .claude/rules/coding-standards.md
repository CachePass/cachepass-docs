# Coding standards

Rust
- Edition 2024, `cargo fmt` (max_width 120), clippy clean with `-D warnings`.
- No `unwrap`/`expect`/`panic!` in production paths (workspace lints deny them);
  tests may use them.
- Explicit errors (`thiserror`, `AppError`); strong types for ids/revisions/blobs;
  avoid needless clones; small functions; doc comments on public items.

TypeScript / Svelte
- Svelte 5 runes only; strict TS; no `any`. `#lib/...` imports with `.js` extensions.
- Validate external data with Zod. Keep components small; extract logic to `features/`.
- No unnecessary global state; stores hold app-wide state only.
- Accessibility: labelled controls, keyboard support, visible focus, reduced motion.
- Internal navigation via `resolve()` (Kit 3 pathnames: `resolve('vault')`).
- Vendored shadcn components in `components/ui/` are regenerated, not hand-edited
  (except documented fixes).
