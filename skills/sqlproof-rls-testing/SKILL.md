---
name: sqlproof-rls-testing
description: Write property-based tests for Supabase / PostgreSQL Row-Level Security (RLS) policies using sqlproof. Use whenever the user asks to test an RLS policy, a `CREATE POLICY` statement, row-level access control, multi-tenant isolation, or anything keyed off `auth.uid()` / `auth.role()` / `auth.jwt()`. Also use when the task mentions cross-org isolation, "can user X see row Y", member-vs-non-member visibility, or `as_rls_user` / `as_supabase_user` context managers. This skill covers the canonical RLS test pattern: both-directions principle (owner can see + non-owner cannot), `as_rls_user` for setting RLS context (claims + `SET LOCAL ROLE`, so superuser BYPASSRLS doesn't skip policies), the `supabase_proof` fixture, and never raw-setting `request.jwt.claims`. Pairs with the core `sqlproof` skill (assumed loaded). Without this skill, generated RLS tests tend to test only the happy path (owner can see) and miss the actual bug class (policies that return TOO MUCH data).
---

# RLS policy tests with sqlproof

## When to write one

Any time the user adds or modifies a `CREATE POLICY` statement, or
when an existing policy needs verification.

## The pattern

```python
"""Test that RLS policies on `<table>` correctly gate access."""

from hypothesis import given
from hypothesis import strategies as st

from sqlproof import SqlProof
from sqlproof.contrib.supabase import as_rls_user


@given(data=st.data())
def test_owner_can_read_their_own_<resource>(
    supabase_proof: SqlProof, data,
) -> None:
    dataset = data.draw(supabase_proof.dataset_strategy(
        sizes={"<resource_table>": 1},
    ))
    with supabase_proof.client_for_dataset(dataset) as db:
        resource = dataset["<resource_table>"][0]
        with as_rls_user(db, resource["user_id"]):
            rows = db.query(
                "SELECT id FROM <resource_table> WHERE id = %s",
                resource["id"],
            )
        assert len(rows) == 1, "owner should see their own resource"


@given(data=st.data())
def test_other_users_cannot_read_<resource>_they_dont_own(
    supabase_proof: SqlProof, data,
) -> None:
    dataset = data.draw(supabase_proof.dataset_strategy(
        sizes={"<resource_table>": 1, "auth.users": 2},
    ))
    with supabase_proof.client_for_dataset(dataset) as db:
        resource = dataset["<resource_table>"][0]
        non_owner = next(
            u for u in dataset["auth.users"] if u["id"] != resource["user_id"]
        )
        with as_rls_user(db, non_owner["id"]):
            rows = db.query(
                "SELECT id FROM <resource_table> WHERE id = %s",
                resource["id"],
            )
        assert rows == [], "non-owner should see no rows"
```

Naming `"auth.users": 2` in `sizes` makes the dataset use exactly two
pool users for every `auth.users` FK and return them as
`dataset["auth.users"]` (each row holds only `id`). Without that entry
there is no `dataset["auth.users"]` key. Requires sqlproof 0.11.2 or
later; on 0.11.1 the key is missing even when named, so read the pool
with `db.query("SELECT id::text AS id FROM auth.users")` instead.

## Critical rules

### Always test BOTH directions

A policy that returns *too much* data is the actual bug class.
Testing only "owner can see" misses cross-tenant leaks. Every RLS
test should ALSO verify "non-owner cannot see."

### Use `as_rls_user(db, user_id)`, not `as_supabase_user`

The test connection is normally the `postgres` superuser, which has
`BYPASSRLS`: as long as queries run under that role, policies are
never evaluated. `as_rls_user` (from `sqlproof.contrib.supabase`)
sets the JWT claims so `auth.uid()` resolves to `user_id` **and**
runs `SET LOCAL ROLE authenticated`, so RLS actually applies. Pass
`role="anon"` (or another role) to test a different Postgres role.

`as_supabase_user` sets only the JWT claims and does not change
role. In an RLS test that makes the outcome independent of the
schema: "non-owner cannot see" fails on a correct policy, and "owner
can see" passes even with RLS disabled. Keep `as_supabase_user` for
code that needs `auth.uid()` resolved without enforcing policies.

Do NOT raw-set `request.jwt.claims` either. `as_rls_user`:
- Sets the JWT claims GUC and the role for the block's duration
- Restores the previous claims and runs `RESET ROLE` on exit
- Is exception-safe (cleanup runs even if assertion fails)
- Needs a transaction (`SET LOCAL`); `client_for_dataset` and
  `supabase_db` already provide one

### Take `supabase_proof` / `supabase_db`, not `proof` / `db`

RLS tests need the seeded `auth.users` pool. The plain fixtures
don't seed it.

### Test names should be sentences

Good: `test_owner_can_read_their_own_project`,
`test_non_owner_cannot_modify_shared_project`.
Bad: `test_policy_22`, `test_rls_works`.

The user reads test names in failure summaries. Make them sentences.

## Multi-role policies

When a policy distinguishes by role (owner / admin / editor / viewer),
test each role separately:

```python
@given(
    data=st.data(),
    role=st.sampled_from(["owner", "admin", "editor", "viewer"]),
)
def test_member_can_read_org_<resource>_by_role(
    supabase_proof: SqlProof, data, role: str,
) -> None:
    dataset = data.draw(supabase_proof.dataset_strategy(
        sizes={"organizations": 1, "org_members": 1, "<resource>": 1},
        columns={"org_members.role": role},
    ))
    with supabase_proof.client_for_dataset(dataset) as db:
        member = dataset["org_members"][0]
        with as_rls_user(db, member["user_id"]):
            rows = db.query("SELECT id FROM <resource> WHERE org_id = %s",
                            member["org_id"])
        # Assert based on the role's expected visibility
        ...
```

## Cross-org isolation tests

For multi-tenant projects, also write a dedicated cross-org test:

```python
@given(data=st.data())
def test_outsider_cannot_read_<resource>_in_another_org(
    supabase_proof: SqlProof, data,
) -> None:
    dataset = data.draw(supabase_proof.dataset_strategy(
        sizes={"organizations": 1, "org_members": 1, "<resource>": 1, "auth.users": 2},
    ))
    with supabase_proof.client_for_dataset(dataset) as db:
        member = dataset["org_members"][0]
        outsiders = [u for u in dataset["auth.users"]
                     if u["id"] != member["user_id"]]
        if not outsiders:
            return  # No outsider available; example invalid
        with as_rls_user(db, outsiders[0]["id"]):
            rows = db.query(...)
        assert rows == [], f"outsider {outsiders[0]['id']} leaked rows"
```

## When the test still passes wrongly

If your RLS test passes but you suspect the policy is broken, check:

1. **Is the connection bypassing RLS?** Make sure the query runs
   inside `as_rls_user`, not `as_supabase_user` (see above).
   `db.scalar("SELECT current_user")` inside the block should be
   `authenticated` (or the `role=` you passed), not `postgres`.
2. **Is `auth.uid()` returning NULL?** Test the GUC propagation:
   `db.scalar("SELECT auth.uid()::text")` inside the
   `as_rls_user` block should match the user_id you passed.
3. **Are you testing the wrong table?** Some "queries on X" actually
   join through Y; RLS on X might be irrelevant.
