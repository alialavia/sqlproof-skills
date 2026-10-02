---
name: sqlproof
description: Write property-based tests for PostgreSQL schemas and SQL behavior using sqlproof. Use whenever the user asks to test a Supabase project, RLS policies, RPC functions, SQL functions, database triggers, migrations, or any code that touches PostgreSQL. Also use whenever the task mentions sqlproof, dataset generation respecting FK/CHECK/UNIQUE constraints, the `@sqlproof` decorator, `proof.check`, `proof.client_for_dataset`, the seeded auth.users pool, `supabase_proof`/`supabase_db` fixtures, or testing with savepoint isolation. This skill explains when to use sqlproof, the four canonical fixtures the pytest plugin ships, how to bootstrap a new project, and what NOT to do. For pattern-specific guidance, also load the relevant sibling skill (`sqlproof-rls-testing`, `sqlproof-rpc-testing`, `sqlproof-stateful-testing`, `sqlproof-ci-setup`). Without this skill, generated tests tend to reinvent fixtures the plugin already ships, hand-roll INSERT helpers that defeat property-based testing, or leak postgres internals (manual JWT claims) into test code.
---

# sqlproof — core

Property-based testing for PostgreSQL with sqlproof. This skill
covers the always-on guidance: when to reach for sqlproof, what
the fixtures are, how to bootstrap a project, and the anti-patterns
the agent must avoid.

For pattern-specific guidance, ALSO load the relevant sibling skill:

- **`sqlproof-rls-testing`** — RLS policy tests
- **`sqlproof-rpc-testing`** — SQL function / RPC tests
- **`sqlproof-stateful-testing`** — sequence-dependent bugs
- **`sqlproof-ci-setup`** — wiring into GitHub Actions

This skill also assumes the foundational property-based-testing
patterns are available via the companion `alialavia/pbt-skills`
plugin. Load that first.

## When to use sqlproof

Use sqlproof for any test that touches:

- **RLS policies** — verify users can only see / modify rows they should
- **`public.*` SQL functions / RPCs** — verify outputs for given inputs
- **Triggers** — verify side effects when parent rows change
- **Migrations** — verify new query produces same result as old query
- **Aggregates and reports** — verify DB aggregate matches a Python recomputation

Do **not** use sqlproof for:

- Schema-shape assertions ("does this column exist") — one-line
  `information_schema` query in pgTAP or skip entirely
- Snapshot tests of literal output — use `syrupy` or pytest's snapshot

## Project setup

The user typically runs Supabase locally via `supabase start`. The
local DB lives at `postgresql://postgres:postgres@127.0.0.1:54322/postgres`.

### Install + declare dev deps

```toml
# pyproject.toml
[project.optional-dependencies]
dev = [
  "sqlproof",
  "pytest>=8",
  "hypothesis>=6.100",
  "psycopg[binary]>=3.1",
]

[tool.pytest.ini_options]
addopts = "-ra"
testpaths = ["tests"]
```

Then `pip install -e ".[dev]"` (or `uv sync --extra dev`).

### Tell sqlproof where the DB is

```bash
export SUPABASE_DB_URL='postgresql://postgres:postgres@127.0.0.1:54322/postgres'
```

Resolution chain: `--sqlproof-database-url` flag → `$SQLPROOF_DATABASE_URL`
→ `$SUPABASE_DB_URL`. Without any of these, sqlproof's fixtures skip
cleanly (they don't fail).

## The four fixtures the plugin ships

**There is no `tests/conftest.py` to write.** sqlproof's pytest
plugin auto-registers these:

- **`proof`** (session) — `SqlProof` connected to the DSN
- **`db`** (per-test) — `SqlProofClient` with savepoint isolation
- **`supabase_proof`** (session) — like `proof`, but with the
  deterministic `auth.users` test pool seeded and registered as
  an external table for FK draws. **Use this for any test that
  touches RLS, `auth.uid()`, or RPCs keyed off auth users.**
- **`supabase_db`** (per-test) — `SqlProofClient` backed by `supabase_proof`

Rule: if a test touches auth → take `supabase_db`. Otherwise → take `db`.
Don't define `proof`/`db` yourself unless you need a custom external
table beyond the auth-users pool.

## Property tests over hand-rolled fixtures (the core idiom)

The whole reason to use sqlproof over pgTAP is that it generates
**many valid datasets** for a single test. Edge cases surface that
you'd never think to type.

**Don't write `_insert_user`, `_insert_project`, `_insert_event`
helpers in your tests.** Hand-rolled INSERTs test only the shape
*you* remembered, not the shape your schema actually permits.

**Do use `dataset_strategy` to generate, then assert:**

```python
from hypothesis import given
from hypothesis import strategies as st


@given(data=st.data())
def test_my_invariant(supabase_proof, data):
    dataset = data.draw(supabase_proof.dataset_strategy(
        sizes={"projects": 1, "events": 5},
    ))
    with supabase_proof.client_for_dataset(dataset) as db:
        # ... assertions
```

Each example generates a fresh dataset that respects every FK,
CHECK, UNIQUE (single-column AND composite), and NOT NULL in the
schema.

### Exceptions

Two cases where hand-rolled INSERTs are acceptable:

1. A test that needs a *specific*, fixed shape (e.g. empty-state
   contract returning a zero-payload for an unknown UUID). Use a
   literal `00000000-...-000000000000`, don't generate.
2. A small helper inside a stateful test machine that mutates rows
   between rules. Even there, prefer `proof.client_for_dataset(...)`
   to seed the initial state.

## Anti-patterns the agent must avoid

### ❌ Don't write `tests/conftest.py` with `proof`/`db` fixtures

The plugin ships them. The only reason to override is registering
an *additional* external table beyond `auth.users`.

### ❌ Don't manually set JWT claims

```python
# Wrong — leaks postgres internals:
db.execute(
    "SELECT set_config('request.jwt.claims', %s, true)",
    json.dumps({"sub": user_id, "role": "authenticated"}),
)

# Right — readable, restored on exit, exception-safe, RLS enforced:
from sqlproof.contrib.supabase import as_rls_user
with as_rls_user(db, user_id):
    ...
```

`as_rls_user` also runs `SET LOCAL ROLE authenticated`; the superuser
test connection has BYPASSRLS, so without the role switch policies are
never evaluated. `as_supabase_user` sets the claims only — use it just
for resolving `auth.uid()` without enforcing policies.

### ❌ Don't pass parameters to `db.execute` as a list

`SqlProofClient`'s methods take `*params` (splat), not a single
sequence:

```python
# Wrong — sends ONE parameter (the list) to a query with TWO placeholders:
db.execute(
    "INSERT INTO posts (org_id, author_id) VALUES (%s, %s)",
    [org_id, author_id],
)
# psycopg.errors.SyntaxError: the query has 2 placeholders but 1 parameters

# Right:
db.execute(
    "INSERT INTO posts (org_id, author_id) VALUES (%s, %s)",
    org_id,
    author_id,
)
```

### ❌ Don't access default-bearing columns without putting them in `columns={...}`

The dataset generator **omits columns with DB defaults** from the
returned dataset. If your test reads `dataset["posts"][0]["is_premium"]`,
you need to declare `is_premium` in `columns={...}` explicitly.

### ❌ Don't insert into `auth.users` if the connection lacks permission

Use `supabase_proof` (which seeds once per session) and sample from
the pool. Don't re-seed per test.

### ❌ Don't skip the `:: cast` in raw SQL

```python
db.scalar("SELECT my_function(%s, %s)", uuid_value, integer_value)  # ambiguous
db.scalar("SELECT my_function(%s::uuid, %s::int)", uuid_value, integer_value)  # explicit
```

Postgres function overload resolution requires explicit casts when
the parameter types aren't obvious.

## File and naming conventions

```
project_root/
├── tests/
│   ├── test_rls_<table>.py            # one file per table with RLS
│   ├── test_rpc_<function_name>.py    # one file per public function
│   ├── test_trigger_<trigger_name>.py # one file per trigger
│   └── test_migration_<n>_<desc>.py   # migration safety tests
└── supabase/
    ├── migrations/...
    ├── schemas/...
    └── tests/...                      # pgTAP files (separate suite)
```

Test names: `test_<subject>_<expected_behavior>_<conditions>`. Good:
`test_owner_can_read_their_own_project`. Bad: `test_policy_22`.

## When the agent is unsure

If the user asks for a test you don't have a pattern for, ask ONE
clarifying question about the intended invariant. Don't guess. Once
the invariant is clear, write the test. If still uncertain after
one question, write the smallest possible example test and flag the
area for the user to expand.
