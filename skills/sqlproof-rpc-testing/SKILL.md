---
name: sqlproof-rpc-testing
description: Write property-based tests for PostgreSQL SQL functions and Supabase RPC functions using sqlproof. Use whenever the user asks to test a `CREATE FUNCTION`, a `LANGUAGE sql` or `LANGUAGE plpgsql` function, a Supabase RPC, an RPC endpoint, or any callable database function. Also use when the task mentions function invariants, aggregation correctness, `db.scalar`, `db.query`, "this function should never return X", or comparing DB output to a Python reference model. This skill covers two test shapes: pure-function property tests (invariants like "never negative", "monotonic in argument N") and aggregation tests that compare the DB result to a Python recomputation across generated datasets. Pairs with the core `sqlproof` skill. Without this skill, generated RPC tests tend to assert specific values against hand-crafted fixtures (the pgTAP style), missing the edge cases — NULLs, decimal precision, empty groups, tied window values — that property-based generation surfaces.
---

# SQL function / RPC tests with sqlproof

## When to write one

Any time the user adds or modifies a `CREATE FUNCTION` in `public.*`,
or when an existing function's behavior is in question.

## Two shapes

### Shape 1: Deterministic function with simple inputs

For pure-ish functions (no schema state), use a property test that
asserts an invariant.

```python
"""Property tests for `compute_order_total`."""

from decimal import Decimal

from hypothesis import HealthCheck, given, settings
from hypothesis import strategies as st

from sqlproof.client import SqlProofClient


PROOF_KW = settings(
    max_examples=100,
    deadline=None,
    suppress_health_check=[HealthCheck.function_scoped_fixture],
)


@PROOF_KW
@given(
    subtotal=st.decimals(min_value=Decimal("0"), max_value=Decimal("9999.99"), places=2),
    tier=st.sampled_from(["standard", "silver", "gold", "platinum"]),
)
def test_invoice_total_is_never_negative(
    db: SqlProofClient, subtotal: Decimal, tier: str,
) -> None:
    result = db.scalar(
        "SELECT compute_order_total(%s::numeric, %s)",
        subtotal, tier,
    )
    assert result >= 0


@PROOF_KW
@given(
    subtotal=st.decimals(min_value=Decimal("1"), max_value=Decimal("1000"), places=2),
)
def test_higher_tier_never_costs_more_than_lower_tier(
    db: SqlProofClient, subtotal: Decimal,
) -> None:
    standard = db.scalar("SELECT compute_order_total(%s, 'standard')", subtotal)
    platinum = db.scalar("SELECT compute_order_total(%s, 'platinum')", subtotal)
    assert platinum <= standard, (
        f"platinum ({platinum}) costs more than standard ({standard})"
    )
```

Use `db` (not `supabase_db`) if the function doesn't depend on
auth state.

### Shape 2: Aggregation against generated dataset

For functions that aggregate over rows (`SUM`, `COUNT`,
`get_dashboard_summary`, etc.), generate the dataset and reconcile
the DB result against a Python recomputation.

```python
@given(data=st.data(), event_count=st.integers(min_value=0, max_value=20))
def test_dashboard_summary_event_count_matches_inserted_count(
    supabase_proof, data, event_count: int,
):
    dataset = data.draw(supabase_proof.dataset_strategy(
        sizes={"projects": 1, "events": event_count},
    ))
    with supabase_proof.client_for_dataset(dataset) as db:
        project_id = dataset["projects"][0]["id"]
        payload = db.scalar(
            "SELECT get_dashboard_summary(%s::uuid)", project_id
        )
    assert payload["event_count"] == event_count
```

The Python recomputation is your **oracle** — the independent
source of truth the DB result is compared against. If you can't
articulate an oracle in one sentence, you don't have a property
test; write an example test instead.

### Shape 3 (lower-value): empty-state contract

For "function returns sensible zeros for unknown input", use a
SINGLE example test with a literal unknown UUID:

```python
def test_dashboard_summary_returns_zero_shape_for_unknown_project(db):
    payload = db.scalar(
        "SELECT get_dashboard_summary(%s::uuid)",
        "00000000-0000-0000-0000-000000000000",
    )
    assert payload["totalUsers"] == 0
    assert payload["recentEvents"] == []
```

Don't generate for this — there's only one shape to test.

## Critical rules

### Property tests describe an invariant

"X is never negative." "Higher tier never costs more." "Sum of X
equals sum of Y." If you can't articulate it in one sentence, write
an example test.

### Use `db.scalar(...)` for single-value returns; `db.query(...)` for `RETURNS TABLE`

```python
result = db.scalar("SELECT compute_order_total(...)")  # one value
rows = db.query("SELECT * FROM get_orders_for_user(...)")  # multi-row
```

### Cast inputs explicitly

```python
db.scalar("SELECT my_function(%s::uuid, %s::numeric, %s::text[])", ...)
```

Postgres function overload resolution is strict on parameter types.

### Don't invent properties

If you can't articulate the invariant in one sentence, write an
example test instead. False properties (ones the function doesn't
actually satisfy) waste shrinking time and produce confusing
counterexamples.

### Don't hand-roll the dataset for aggregation tests

Even when you "just need one row." Use `dataset_strategy`; that one
row should still respect every constraint.

## What kinds of bugs this catches

- **Aggregates that drift by one** when there are NULL values.
- **SUMs that lose precision** on `numeric` after a refactor.
- **`ROW_NUMBER() OVER (ORDER BY ...)` that's nondeterministic**
  on tied values.
- **Window functions with wrong `PARTITION BY`** that leak across
  groups.
- **`COALESCE` shadowing** that hides NULL semantics until you hit
  the case.

Property-based tests catch these because they generate hundreds of
valid datasets, including the one that hits the edge case.
