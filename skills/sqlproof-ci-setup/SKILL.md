---
name: sqlproof-ci-setup
description: Set up sqlproof in a GitHub Actions CI pipeline for a Supabase-shaped (or vanilla Postgres) project. Use whenever the user asks to "add sqlproof to CI", configure GitHub Actions for sqlproof tests, set up `supabase/postgres` as a service container, add the auth migration that makes `auth.uid()` work in tests, or wire `setup-supabase-test-db` composite action into their workflow. Also use when the task mentions CI failures with `auth.uid()` returning NULL, `storage.buckets` missing columns, `plpgsql_check` not installed, the `SQLPROOF_DATABASE_URL` env var, or GitHub Actions service containers for PostgreSQL. This skill provides the canonical drop-in workflow (Shape 1: Supabase + auth + RLS) plus brief variations for vanilla Postgres and Python-version matrices. Pairs with the core `sqlproof` skill. Without this skill, generated CI workflows tend to use `postgres:16` (missing `auth` schema), forget the GoTrue auth migration (causing silent `auth.uid() = NULL`), or miss the `storage.buckets` columns later Supabase migrations add.
---

# Wiring sqlproof into GitHub Actions CI

## The drop-in workflow (Supabase + auth + RLS)

This is the canonical setup for projects whose tests touch RLS,
`auth.uid()`, or anything Supabase-specific. Drop into
`.github/workflows/test.yml`:

```yaml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: supabase/postgres:15.8.1.040
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U postgres -d postgres"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 15
    env:
      SQLPROOF_DATABASE_URL: postgresql://postgres:postgres@127.0.0.1:5432/postgres
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v6
        with:
          python-version: "3.11"
      - uses: alialavia/sqlproof/.github/actions/setup-supabase-test-db@main
        with:
          database-url: ${{ env.SQLPROOF_DATABASE_URL }}
      - run: pip install -e ".[dev]"
      - run: pytest
```

In production, replace `@main` with a tagged release
(e.g. `@v0.2.4` or later) so upstream changes can't surprise the
build.

## What each piece does

**`supabase/postgres:15.8.1.040`** (not `postgres:16`) — ships the
`auth` schema with `auth.users`, the `plpgsql_check` extension, and
the extensions Supabase bundles. Pin to a specific tag.

**`alialavia/sqlproof/.github/actions/setup-supabase-test-db@…`** —
composite action that:
- Installs `plpgsql_check` extension
- Applies [GoTrue's `20220224000811_update_auth_functions` migration](https://github.com/supabase/auth/blob/master/migrations/20220224000811_update_auth_functions.up.sql)
  so `auth.uid()` / `auth.role()` / `auth.email()` accept the modern
  JSON `request.jwt.claims` GUC (PostgREST 8+ format)
- Adds missing columns to `storage.buckets` (`public`,
  `avif_autodetection`, `file_size_limit`, `allowed_mime_types`,
  `owner_id`, `type`) so storage-touching migrations apply cleanly

**Without this step**, `auth.uid()` silently returns NULL in tests
and every RLS test passes for the wrong reason — the test sees no
rows because the policy can't identify the user.

**`SQLPROOF_DATABASE_URL`** — sqlproof's pytest plugin reads this.
Job-level scope means every step sees it.

## Variation 1: vanilla Postgres (no Supabase auth)

For projects that don't use RLS / `auth.uid()`:

```yaml
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_PASSWORD: postgres
    ports:
      - 5432:5432
    options: >-
      --health-cmd "pg_isready -U postgres"
      --health-interval 10s
      --health-timeout 5s
      --health-retries 10
env:
  SQLPROOF_DATABASE_URL: postgresql://postgres:postgres@127.0.0.1:5432/postgres
```

Skip the `setup-supabase-test-db` step entirely.

## Variation 2: Multi-Python-version matrix

Add to the job:

```yaml
strategy:
  matrix:
    python-version: ["3.11", "3.12", "3.13"]
```

The `services.postgres` block boots once per matrix job, so this
just works.

## Variation 3: Use `SUPABASE_DB_URL` for local-dev parity

sqlproof's pytest plugin reads `--sqlproof-database-url` →
`SQLPROOF_DATABASE_URL` → `SUPABASE_DB_URL` (in that order).

If the user runs `supabase start` locally (which exposes Postgres
on `127.0.0.1:54322` and sets `SUPABASE_DB_URL`), they can use
`SUPABASE_DB_URL` in the CI workflow too — same env var both places:

```yaml
env:
  SUPABASE_DB_URL: postgresql://postgres:postgres@127.0.0.1:5432/postgres
```

## Common failure modes

### `auth.uid()` returns NULL

You're either not using `setup-supabase-test-db`, or you're running
against `postgres:16` (which has no `auth` schema). Both are
required.

### Tests skip with "set SQLPROOF_DATABASE_URL..."

The env var isn't reaching the test step. Check it's in the job's
top-level `env:` block, not inside an individual step.

### `Error: ENOENT: no such file or directory, scandir '...github/actions/setup-supabase-test-db'`

You're referencing the action at a ref where it doesn't exist yet
(introduced in sqlproof v0.2.0). Either pin to `@main`, or to
`@v0.2.4` or later.

### `pip install sqlproof` says "no matching distribution"

This was needed when sqlproof was in alpha. As of v0.2.x, plain
`pip install sqlproof` works without `--pre`.

### Service container takes too long to start

Increase `--health-retries` in `services.postgres.options`. The
Supabase image is ~600 MB; cold pulls on a fresh runner can take
~30 seconds before the healthcheck loop begins.

## See also

The full CI/CD guide on [sqlproof.com/guides/ci-cd/](https://sqlproof.com/guides/ci-cd/)
has the same content rendered for non-agent readers, plus
troubleshooting one level deeper.
