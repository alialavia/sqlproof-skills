---
name: sqlproof-scenarios
description: Describe a seeded, reproducible PostgreSQL database state with sqlproof scenarios instead of writing seed SQL. Use whenever the user needs a database holding specific rows — seed a dev or local Supabase database, a preview branch, data for an end-to-end or UI test, a demo, or the state from a bug report ("2 tier 1 customers, 3 tier 2, invoices for the tier 1 ones"). Also use when the task mentions `seed.sql`, seed scripts, fixtures data, `@scenario`, `sqlproof scenarios up/show/export/list`, `*.scenario.yaml` or `*_scenarios.py` files, or the MCP `compile_scenario` / `load_scenario` tools. Covers the Python and YAML forms, references between rows, per-parent child counts, checking without a database, and the safety rules for loading. Pairs with the core `sqlproof` skill. Without this skill, agents hand-write INSERT scripts with hard-coded ids that break when the schema changes, or wipe a database the user cared about to make room for seed data.
---

# Seeded database states with sqlproof scenarios

A **scenario** is a named, seeded database state. Pin the rows the case
depends on; sqlproof generates every other row and column from the
seed, respecting every FK, CHECK, UNIQUE and NOT NULL. The same
scenario, seed and schema load the same rows every time.

**Experimental.** If `import sqlproof.scenarios` fails, or
`sqlproof scenarios` is not a command, the installed sqlproof predates
scenarios; upgrade it.

## When to use a scenario (and when not)

Use one when the user needs a database **holding specific rows**:

- seeding a local or dev database, or a preview branch;
- the starting state for an end-to-end, UI or API test;
- a demo dataset;
- reproducing a bug report ("it breaks once a customer has more than 5
  invoices").

**Not for property tests.** A property test wants *many* generated
datasets (`@sqlproof`, see the core skill and the RLS / RPC / stateful
skills). A scenario is *one* fixed state.

**Instead of a hand-written `seed.sql`.** Hard-coded INSERTs with
literal ids break whenever a column or constraint changes; a scenario
only names what matters and keeps loading as the schema grows.

## The two forms

Python, in `scenarios/<area>_scenarios.py` (any number per file):

```python
from sqlproof.scenarios import scenario


@scenario(seed=42)
def tiered_customers(s):
    alice = s.row("auth.users", name="alice")
    s.row("customers", name="acme", tier=1, owner_id=alice)
    s.table("customers").cohort(2, tier=1).cohort(3, tier=2).fill(20)
    s.table("invoices").per_parent(
        "customers", where={"tier": 1}, count=(5, 10), default=(0, 2)
    )
```

YAML, one per `scenarios/<name>.scenario.yaml` file. Use this form in a
project that does not otherwise use Python (a TypeScript Supabase app).
YAML needs `sqlproof[yaml]`; `*.scenario.json` files need nothing extra.

```yaml
# yaml-language-server: $schema=https://sqlproof.com/schemas/scenario-v1.json
version: 1
name: tiered_customers
seed: 42
tables:
  auth.users:
    - row: alice
  customers:
    - row: acme
      set: {tier: 1, owner_id: {ref: alice}}
    - cohort: 2
      set: {tier: 1}
    - cohort: 3
      set: {tier: 2}
    - fill: 20
  invoices:
    - per_parent:
        parent: customers
        where: {tier: 1}
        count: [5, 10]
        default: [0, 2]
```

| Python | YAML | Rows |
| --- | --- | --- |
| `.cohort(n, col=v)` | `- cohort: n` + `set: {col: v}` | `n` rows with `col` pinned |
| `.fill(n)` | `- fill: n` | `n` rows, nothing pinned |
| `s.row("t", name="x", col=v)` | `- row: x` + `set: {...}` | one row others can reference |
| `s.ref("x")` | `{ref: x}` | the key of row `x` |
| `.per_parent("p", where=..., count=(lo, hi))` | `- per_parent: {parent: p, where: ..., count: [lo, hi]}` | `lo`–`hi` children per matching parent |

`where` is a mapping of values the parent's **cohorts** pin, a reference
(`{ref: acme}`), or omitted for every parent. Parents not matched get
`default` (0 unless given).

## Workflow

1. **Write the scenario** with the fewest pins that express the case.
2. **Check it without a database:**

   ```bash
   sqlproof scenarios show tiered_customers --schema-file supabase/schemas/schema.sql
   ```

   It prints the rows per table and the key of every named row, or the
   first problem (unknown column, a value that breaks a CHECK, a
   reference to an undeclared row, a child table whose parent is
   empty). Fix and re-run until it is clean.
3. **Load it into an empty database:**

   ```bash
   sqlproof scenarios up tiered_customers --dsn "$SQLPROOF_DATABASE_URL"
   ```

   From Python: `report = tiered_customers.load(dsn)`; `report.refs`
   holds the key of every named row.

Without Python in the project:
`uvx --from "sqlproof[yaml]" sqlproof scenarios up tiered_customers --dsn "$DATABASE_URL"`.

With the `sqlproof-mcp` server, call `compile_scenario(scenario,
schema_sql)` for step 2 and `load_scenario(scenario, dsn)` for step 3,
passing the scenario as JSON in the YAML shape above. `list_recipes`
has a `scenario` template.

## Rules

- **Never pass `--truncate-existing` (`truncate_existing=True`) without
  the user's explicit go-ahead for that database.** It deletes every
  row in the tables the schema models. A load refuses tables that
  already hold rows, including rows row-level security hides from you.
  That refusal is the safety net, not an obstacle: report it and ask.
  The MCP `load_scenario` tool has no truncate option at all.
- **Reference rows by name, never by id.** Do not hard-code a generated
  id or UUID in app code, tests or docs; read it from `report.refs` or
  `scenarios show`.
- **Pin only what the case is about.** Every extra pin is a constraint
  to maintain as the schema changes.
- **`auth.users`** takes `fill(n)` and named rows only, no pinned
  columns. Ids derive from the scenario name and seed; emails are
  `sqlproof+<scenario>+<n>@test.invalid`. An FK into `auth.users` you
  do not pin gets one of the scenario's users.
- **Timestamps are fixed.** `now()`-style column defaults become the
  scenario clock (2026-01-01 UTC unless `clock=` is set), and
  `gen_random_uuid()` a seeded UUID, so loads are identical. SQL
  functions that call `now()` at query time still see the real clock.
- **Keep scenarios shareable.** A callable pin (`col=lambda ctx: ...`)
  works in Python but cannot be exported to YAML; prefer literals and
  references. `sqlproof scenarios export NAME` turns a Python scenario
  into YAML.
- **Pin `json` and array columns as strings** in YAML.

## Current limits

- A table sized by `per_parent` cannot also take cohorts, rows or fills,
  and takes one `per_parent`.
- `auth.users` is the only table outside the loaded schema a scenario
  can seed.
- In Python, `s.row(..., name=...)` uses `name` for the row's name, so a
  column called `name` cannot be pinned on a named row; use
  `cohort(1, name=...)`, or YAML, where `row:` and `set: {name: ...}`
  do not collide.

Full guide: <https://sqlproof.com/guides/scenarios/>.
