# sqlproof-skills

Claude Code skills for testing PostgreSQL schemas and SQL behavior with
[sqlproof](https://github.com/alialavia/sqlproof).

## What's included

Five auto-activating skills:

- **`sqlproof`** — core skill. Activates whenever the user asks about
  testing PostgreSQL, RLS policies, RPC functions, or property-based
  testing for databases. Covers when to use sqlproof, the canonical
  fixtures (`proof`, `db`, `supabase_proof`, `supabase_db`), and how
  to bootstrap a new project.
- **`sqlproof-rls-testing`** — pattern for testing an RLS policy.
  Both-directions principle (owner can see + non-owner cannot),
  `as_rls_user` context manager (claims + `SET LOCAL ROLE` so RLS is
  actually enforced), no manual JWT-claim setting.
- **`sqlproof-rpc-testing`** — pattern for testing a SQL function /
  RPC. Property tests with invariants ("never negative", "monotonic",
  oracle-comparison against a Python recomputation).
- **`sqlproof-stateful-testing`** — pattern for bugs that only
  manifest after a sequence of operations (membership churn,
  pagination, accumulation). `SqlProofStateMachine` + `@rule` +
  `@invariant`.
- **`sqlproof-ci-setup`** — wiring sqlproof into GitHub Actions
  CI. The drop-in workflow that uses
  `alialavia/sqlproof/.github/actions/setup-supabase-test-db@v0.x.y`,
  with the auth migration and `storage.buckets` columns Supabase
  consumers need.

All skills auto-activate based on task description. The agent
loads only the relevant skill for the task at hand, not all 700+
lines of `AGENTS.md` from the main repo.

## Pairs with pbt-skills

These skills assume the foundational property-based-testing patterns
(anti-tautology discipline, oracle naming, generator design) live in
the companion `alialavia/pbt-skills` plugin. Install both together
for the best experience.

## Install

In Claude Code:

```
/plugin install alialavia/sqlproof-skills
```

Or declare both as project dependencies in `.claude/settings.json`:

```json
{
  "plugins": [
    "alialavia/pbt-skills",
    "alialavia/sqlproof-skills"
  ]
}
```

## License

MIT. See [LICENSE](./LICENSE).

## Related

- **[alialavia/sqlproof](https://github.com/alialavia/sqlproof)** —
  the underlying library these skills configure agents to use.
- **[alialavia/pbt-skills](https://github.com/alialavia/pbt-skills)** —
  foundational property-based-testing skills.
- **[Trail of Bits skills marketplace](https://github.com/trailofbits/skills)** —
  ships a broader `property-based-testing` skill with smart-contract
  / security focus. Both can coexist if your work spans both contexts.
