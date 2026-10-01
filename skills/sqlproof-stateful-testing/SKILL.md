---
name: sqlproof-stateful-testing
description: Write stateful (sequence-dependent) tests for PostgreSQL using sqlproof's SqlProofStateMachine. Use whenever the user asks to test a bug that only manifests after a sequence of operations, membership churn (join/leave repeated), pagination across mutations, accumulator state, or any "works on first call but fails after N operations" pattern. Also use when the task mentions `SqlProofStateMachine`, `@rule`, `@invariant`, Hypothesis stateful testing, or `run_state_machine`. State machines are slower than property tests; use them only when the bug REQUIRES a sequence. For one-shot assertions, use the `sqlproof-rls-testing` or `sqlproof-rpc-testing` skill instead. This skill covers the canonical pattern: subclass SqlProofStateMachine, override `on_setup` (not `__init__`), define `@rule` decorators for mutations and `@invariant` decorators for the consistency check, use `self.enter(cm)` for context managers across rules. Pairs with the core `sqlproof` skill. Without this skill, generated stateful tests tend to incorrectly override `__init__`, miss the `self.enter` pattern for JWT claims, or run state machines for tests that don't need sequences.
---

# Stateful tests with sqlproof

## When to write one

Stateful tests are for bugs that only manifest after a **sequence**
of operations:

- Membership churn (join, leave, re-join, see what visibility looks like)
- Pagination across mutations (insert rows between page fetches)
- Accumulator state (running totals that drift)
- Trigger-induced side effects after specific sequences
- RLS policies that misbehave after role changes

If your test doesn't need a sequence — one INSERT, one SELECT — use
a property test (`@given`) instead. State machines have setup
overhead per example and run slower.

## The pattern

```python
"""Stateful test: project membership and visibility."""

from uuid import uuid4

from hypothesis import HealthCheck, settings
from hypothesis import strategies as st
from hypothesis.stateful import invariant, rule

from sqlproof import SqlProof
from sqlproof.contrib.supabase import as_rls_user
from sqlproof.testing import SqlProofStateMachine


class MembershipMachine(SqlProofStateMachine):
    def on_setup(self) -> None:
        # Pull a real user from the seeded auth.users pool.
        rows = self.db.query(
            r"SELECT id::text FROM auth.users WHERE email LIKE %s ESCAPE '\\' LIMIT 4",
            r"sqlproof\\_%@test.invalid",
        )
        self.user_id = rows[0]["id"]
        self.projects: list[str] = [str(uuid4()) for _ in range(3)]
        self.member_of: set[str] = set()

    @rule(idx=st.integers(0, 2))
    def join_project(self, idx: int) -> None:
        project_id = self.projects[idx]
        self.db.execute(
            "INSERT INTO project_members (project_id, user_id, role) "
            "VALUES (%s, %s, 'viewer') ON CONFLICT DO NOTHING",
            project_id, self.user_id,
        )
        self.member_of.add(project_id)

    @rule(idx=st.integers(0, 2))
    def leave_project(self, idx: int) -> None:
        project_id = self.projects[idx]
        self.db.execute(
            "DELETE FROM project_members WHERE project_id = %s AND user_id = %s",
            project_id, self.user_id,
        )
        self.member_of.discard(project_id)

    @invariant()
    def user_only_sees_projects_they_are_member_of(self) -> None:
        # Rules mutate as the test superuser; only the visibility check
        # runs as the user. `as_rls_user` also does `SET LOCAL ROLE
        # authenticated` — without it the superuser's BYPASSRLS means
        # the policy is never evaluated.
        with as_rls_user(self.db, self.user_id):
            visible = {row["id"] for row in self.db.query("SELECT id FROM projects")}
        assert visible == self.member_of, (
            f"visible {visible} != expected {self.member_of}"
        )


def test_membership_visibility_invariant(supabase_proof: SqlProof) -> None:
    supabase_proof.run_state_machine(MembershipMachine)
```

## Critical rules

### Override `on_setup`, NOT `__init__`

```python
# Wrong:
class MyMachine(SqlProofStateMachine):
    def __init__(self):
        super().__init__()
        self.foo = ...

# Right:
class MyMachine(SqlProofStateMachine):
    def on_setup(self) -> None:
        # `self.db` is already set up by the base class
        self.foo = ...
```

sqlproof manages `__init__`; overriding it breaks the lifecycle.

### Use `self.enter(cm)` for context managers across rules

If you need a context manager active for the WHOLE example (JWT
claims, savepoints, mocked clocks), call `self.enter(cm)` in
`on_setup`. The cleanup happens automatically when the example
ends.

```python
self.enter(as_rls_user(self.db, self.user_id))
# Now every rule's query runs as that user, with RLS enforced
```

Only do this when the rules themselves should go through the
policies (e.g. testing that a user can INSERT/DELETE their own
membership). If rules set up state the user couldn't create
directly, keep them on the superuser connection and wrap just the
invariant's query in `as_rls_user`, as in the pattern above. Use
`as_supabase_user` (claims only, no role switch) only when you need
`auth.uid()` resolved without enforcing policies.

### Run via `proof.run_state_machine(MachineClass)`

NOT `run_state_machine_as_test` directly — `proof.run_state_machine`
handles the dataset / client binding correctly.

```python
def test_my_machine(supabase_proof: SqlProof) -> None:
    supabase_proof.run_state_machine(MyMachine)
```

### State machines are SLOWER — use them only when needed

A state machine has setup overhead per example. If your assertion
doesn't depend on a *sequence* of operations, write a property test
(`@given`) instead.

## Tuning

```python
def test_membership_visibility_invariant(supabase_proof: SqlProof) -> None:
    supabase_proof.run_state_machine(
        MembershipMachine,
        settings=settings(
            max_examples=15,           # how many independent sequences
            stateful_step_count=10,    # steps per sequence
            deadline=None,
            suppress_health_check=[HealthCheck.function_scoped_fixture],
        ),
    )
```

`stateful_step_count` is the depth of each sequence;
`max_examples` is how many independent sequences run.

## Tracking model state in Python

The invariant compares DB state to a Python model. Keep the model in
`self.<name>` updates inside each rule:

```python
@rule(...)
def do_thing(self):
    self.db.execute(...)
    self.model_state.update(...)  # mirror in the model

@invariant()
def db_matches_model(self):
    db_state = self.db.query(...)
    assert db_state == self.model_state
```

If updating the Python model accurately is hard, that's often a sign
the production code is too complex — a useful test smell to flag to
the user.
