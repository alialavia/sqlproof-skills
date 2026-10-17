---
name: sqlproof
description: Write property-based tests for PostgreSQL schemas and SQL behavior using sqlproof. Use whenever the user asks to test a Supabase project, RLS policies, RPC functions, SQL functions, database triggers, migrations, or any code that touches PostgreSQL. Also use whenever the task mentions sqlproof, dataset generation respecting FK/CHECK/UNIQUE constraints, the `@sqlproof` decorator, `proof.check`, `proof.client_for_dataset`, the seeded auth.users pool, `supabase_proof`/`supabase_db` fixtures, or testing with savepoint isolation. This skill explains when to use sqlproof, the four canonical fixtures the pytest plugin ships, how to bootstrap a new project, and what NOT to do. For pattern-specific guidance, also load the relevant sibling skill (`sqlproof-rls-testing`, `sqlproof-rpc-testing`, `sqlproof-stateful-testing`, `sqlproof-ci-setup`, `sqlproof-scenarios`). Without this skill, generated tests tend to reinvent fixtures the plugin already ships, hand-roll INSERT helpers that defeat property-based testing, or leak postgres internals (manual JWT claims) into test code.
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
- **`sqlproof-scenarios`** — a seeded, reproducible database state
  (seed data, e2e test setup, bug repros) instead of seed SQL

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

For a database that should *hold specific rows* (seed data, an e2e test's
starting state, a bug repro), use a scenario: load `sqlproof-scenarios`.

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

## What the generators don't produce (check before you trust a pass)

`dataset_strategy` draws from deliberately practical windows, not
from everything PostgreSQL accepts. A property whose bug lives outside
those windows **passes on every run** — the search never gets there.
This is the most common way property tests miss bugs (the
`pbt-skills` core skill, Step 5.5), and with a schema-driven generator
it is invisible unless you know the windows.

| Type | Generated by default | Not generated |
|---|---|---|
| `date` | 2000-01-01 to 2054-10-04 | earlier/later dates, `infinity` |
| `timestamp`, `timestamptz` | 2000-01-01 to 2031-09-09; `timestamptz` in UTC | other years, non-UTC offsets, `infinity` |
| `interval` | 0 to ~115 days | negative intervals, months/years |
| `numeric` (no precision) | ±1,000,000, 2 places | larger magnitudes, other scales |
| `real`, `double precision` | ±1e6 unless a CHECK bounds it | NaN, ±Infinity, extremes |
| arrays | at most 5 elements, one dimension | long arrays (unless a CHECK asks) |
| `bytea` | at most 32 bytes | larger values |
| ranges | non-empty, both bounds finite, `[)` | empty ranges, unbounded bounds |
| `json`/`jsonb` | small nested documents (≤10 leaves) | large or deep documents |
| columns with a `DEFAULT` | omitted (the DB fills them) | any other value |

NULL: nullable columns get `NULL` sometimes, not always — a property
about NULL handling needs a `columns=` override that forces it.

So, for every property: name the inputs most likely to break it
(boundaries, negatives, empties, NULL, type extremes, equal values),
check the table above, and **override any column whose trigger region
is outside the default** with `columns={"table.column": strategy}`:

```python
from datetime import date, timedelta
from decimal import Decimal
from hypothesis import given, strategies as st
from sqlproof import Range

NO_NUL = st.characters(blacklist_characters="\x00")  # Postgres text can't hold NUL

WIDE = {
    "events.day": st.dates(min_value=date(1, 1, 1), max_value=date(9999, 12, 31)),
    "jobs.timeout": st.timedeltas(min_value=-timedelta(days=3650),
                                  max_value=timedelta(days=3650))
                      .map(lambda d: timedelta(seconds=round(d.total_seconds()))),
    "bookings.span": st.one_of(
        st.just(Range(empty=True)),
        st.builds(lambda lo, hi: Range(lo, hi, "[)"),
                  st.none() | st.integers(-10, 0), st.none() | st.integers(1, 10)),
    ),
    "posts.tags": st.lists(st.none() | st.text(alphabet=NO_NUL, max_size=3), max_size=20),
    "invoices.amount": st.one_of(            # numeric(10,2): pin both extremes
        st.sampled_from([Decimal("-99999999.99"), Decimal("99999999.99")]),
        st.decimals(-10**8 + 1, 10**8 - 1, places=2),
    ),
    "profiles.bio": st.none(),               # force the NULL path
}

@given(data=st.data())
def test_report_handles_full_domain(proof, data):
    dataset = data.draw(proof.dataset_strategy(sizes={"events": 5}, columns=WIDE))
    ...
```

Rules that go with it:

- **Don't `assume()` away awkward rows.** Filtering out duplicates,
  empty groups, NULLs or zero amounts because the assertion gets harder
  usually removes the exact case the SQL gets wrong. Make the Python
  oracle handle them instead.
- **Pin narrow triggers with `@example`** (a threshold value, the type
  minimum, an empty table via `sizes={"t": 0}`). Database tests run
  ~100 examples; a single boundary value won't be hit by chance.
- **One failing property is not the end.** Report it, then finish the
  remaining properties without narrowing their generators around the
  known failure.
- **Known sqlproof issue:** until
  [alialavia/sqlproof#167](https://github.com/alialavia/sqlproof/issues/167)
  is fixed, `interval` values with fractional seconds are read back
  wrong (`0.5 s` → `5 µs`), so the recipe above generates whole
  seconds. Don't compare sub-second intervals read from the database.

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
