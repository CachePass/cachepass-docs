---
name: code-review
description: Review a CachePass diff for correctness, architecture boundaries, tests and maintainability.
---

1. Boundaries: change stays in the right repo/layer; no cross-repo imports; handlers thin.
2. Correctness: error paths, transactions, revision/concurrency handling, edge cases.
3. Tests: new behaviour covered at the right level; tests can actually fail.
4. Types and validation: no `any`/`unwrap` in prod paths; Zod / domain types at boundaries.
5. Contracts: `openapi.json` regenerated; webapp API mirror updated; docs in sync.
6. Simplicity: no unnecessary dependencies, abstractions or global state.
Hand security-sensitive diffs to the `security-review` skill as well.
