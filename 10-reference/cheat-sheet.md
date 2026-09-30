# Dimensional Modeling Cheat Sheet

Review time: about 10–15 minutes.

## The four-step method

1. **Select the business process.** Model an activity or state, not a department or report.
2. **Declare the grain.** Write: “One row represents ...”
3. **Identify dimensions.** Who, what, where, when, why, and how at that grain?
4. **Identify facts.** Which measurements are true at exactly that grain?

Never choose dimensions or facts before grain.

## Grain

```text
Grain = the exact business meaning of one row.
```

- Atomic grain is the lowest useful detail captured by the process.
- Different business processes or grains normally require different facts.
- Duplicate rows often reveal an incomplete grain, join fanout, retries, or revisions.
- `DISTINCT` is not an explanation.
- Dimensions have grain too: a Type 2 dimension is one row per entity version.

## Star schema

```text
                  dim_user
                      |
dim_date -- fact_learning_event -- dim_course
                      |
              dim_learning_object
```

- **Fact:** measurements and dimension foreign keys at one grain.
- **Dimension:** descriptive context used to filter, group, and label.
- **Star:** dimensions connect directly to a fact.
- **Snowflake:** normalized dimension branches; use sparingly in consumer models.
- Dimensional models optimize analytical understanding; normalized models optimize operational integrity and updates.

## Keys

```text
source natural key       = identity inside one source
durable warehouse key   = one business entity across sources/time
dimension surrogate key = one dimension row/version
```

- Namespace natural keys when sources can collide.
- Type 2 creates a new surrogate key per historical version.
- Facts normally store the surrogate key valid at event time.
- Unknown identity, not applicable, invalid, and inferred member are different states.
- Inferred member: identity known, descriptive attributes pending; give it its own key.

## Core fact-table patterns

| Pattern | One row represents | Write behavior | Best for |
|---|---|---|---|
| Transaction | One event | Mostly insert | Detailed activity and audit |
| Periodic snapshot | One entity/combination per period | Insert every period | Balances, inventory, headcount, recurring state |
| Accumulating snapshot | One lifecycle instance | Insert then update milestones | Orders, applications, cases, enrollments |
| Event factless | One occurrence | Insert | Attendance, eligibility event, assignment |
| Coverage factless | One possible/required combination | Generate by period | What could happen; compare with activity |

An order process may legitimately use transaction, periodic, and accumulating facts together.

## Measures and aggregation

| Class | Rule | Examples |
|---|---|---|
| Additive | Sum across every dimension | revenue amount, quantity, event count |
| Semi-additive | Sum across some dimensions, not all | account balance across accounts but not dates |
| Non-additive | Recompute at query grain | ratio, percentage, average, unit price |

```text
rate = SUM(numerator) / SUM(denominator)
```

Do not average averages or sum balances through time. Distinct counts must name the entity key: durable key for entities, surrogate key for versions.

## Slowly changing dimensions

| Type | Action | Meaning |
|---|---|---|
| 0 | Retain original | Never change the attribute |
| 1 | Overwrite | Show latest/corrected value for all history |
| 2 | Add version row | Preserve event-time history |
| 3 | Add alternate/prior column | Keep a limited alternative perspective |
| 4 | Mini-dimension | Split rapidly changing attribute cluster |
| 5–7 | Hybrid | Offer combinations of current and historical views |

### Type 2 minimum

```text
customer_sk       surrogate version key
customer_bk       source/durable identity mapping
valid_from        inclusive
valid_to          exclusive
is_current        one current version per entity
```

- Intervals must not overlap.
- Resolve fact key using event time during loading.
- Normal queries join the stored surrogate key, not a date range.
- Count durable entity keys when counting customers, not version keys.
- Type 1 can change prior report answers; Type 2 preserves versions.

## Reusable dimension patterns

- **Role-playing:** one physical dimension, several logical roles — order date, ship date, invoice date.
- **Degenerate:** identifier in the fact with no separate attributes — order number, invoice number.
- **Junk:** combinations of low-cardinality flags/statuses.
- **Mini-dimension:** rapidly changing attribute cluster separated from a large base dimension.
- **Outrigger:** dimension linked to another dimension; use carefully.
- **Shrunken:** conformed subset of rows/columns, often for aggregates.
- **Audit:** pipeline run, source, quality, or lineage context.
- **Hot-swappable:** alternative compatible dimension interpretations over the same facts.

## Many-to-many and bridges

First ask whether a lower fact grain can identify one dimension member. If not:

```text
fact -> bridge group -> multiple dimension members
```

- State bridge grain explicitly.
- Weighted allocation: weights per group total 1; additive total is preserved.
- Impact analysis: full fact associated with every member; totals can intentionally overcount.
- Effective-date the bridge when membership changes.
- Hide complexity behind governed views/semantics where possible.

## Hierarchies

- Fixed-depth, stable hierarchy: flatten named levels into the dimension.
- Slightly ragged: flatten only with defensible business rules/placeholders.
- Truly ragged or shared parentage: parent-child or hierarchy bridge.
- Time-varying hierarchy: version membership/paths and require an as-of perspective.
- Avoid generic `level_1`, `level_2` columns when levels have different business meaning.

## Conformance and the bus

- **Conformed dimension:** shared keys, values, attribute meanings, and history across processes.
- **Conformed fact:** same name means the same technical definition, unit, and time basis.
- **Bus matrix:** business processes as rows, shared dimensions as columns.
- **Bus architecture:** deliver process marts incrementally while conforming shared dimensions.
- **Drill-across:** aggregate each fact separately to identical conformed headers, then align results.
- Avoid direct fact-to-fact joins.

## Advanced fact patterns

- **Aggregate fact:** higher-grain performance structure; atomic data remains authoritative.
- **Consolidated fact:** combines processes only when they can share exactly one grain.
- **Header/line:** put header context on line facts; keep or allocate header measures.
- **Allocation:** distribute with a governed rule; preserve original amount when needed.
- **Currency:** store transaction and governed base-currency amounts plus currency identity/rate context.
- **Units:** store standard-unit values and explicit conversion factors.
- **Lag/duration:** anchor calculations to defined milestones and business calendars.
- **Timespan fact:** one state valid for an interval; useful when periodic repetition is wasteful.
- **Fact surrogate key:** optional ETL row identity, not the business grain.

## Late data

```text
event time      = when it happened
processing time = when the platform handled it
```

- Late fact: assign original business date and event-time dimension versions.
- Late dimension with known ID: create an inferred member; complete it later.
- Retroactive Type 2 correction may require interval splits and fact rekeying.
- Rebuild affected snapshots, aggregates, and semantic caches.
- Decide whether history means corrected truth, as-originally-published truth, or both.

## Event and state

```text
individual changes          -> transaction event fact
current state only          -> current table / Type 1 view
state at regular intervals  -> periodic snapshot
latest lifecycle milestones -> accumulating snapshot
continuous valid state      -> timespan model
valid + known-at-the-time   -> bitemporal model
```

Type 2 is not automatically bitemporal. Storage time travel is not automatically business valid-time history.

## Modern layers

```text
Bronze: source fidelity and replay
Silver: clean, deduplicate, integrate
Gold: business-facing dimensional/entity models
Semantic: governed relationships and metrics
```

- Medallion organizes refinement; Kimball organizes analytical meaning.
- Data Vault hubs, links, and satellites can preserve integrated history upstream.
- Dimensional information marts make Vault/Silver history usable.
- A dbt or SAP model may be logical rather than a physical star; semantic rules still apply.

## Reliable loading

- Stable business identity + deterministic winner = safe deduplication.
- Idempotency means retrying accepted input yields the same target state.
- CDC records physical source changes; interpret their business meaning.
- High-water marks need late-arrival/delete safeguards.
- Backfills include facts, dimensions, snapshots, aggregates, and caches.
- Version code/configuration and retain replayable source evidence.
- Reconcile counts, amounts, keys, intervals, lateness, and special-member rates.

## Red flags

- “One big fact for everything.”
- `SELECT DISTINCT` with no duplicate classification.
- Natural-key joins from facts to a Type 2 dimension.
- Facts joined directly to facts.
- Percentage or average stored without components.
- Balances summed across dates.
- Descriptions placed in facts.
- Many snowflaked presentation joins.
- Type 2 on every changing field.
- KPI logic copied into dashboards.
- Null foreign keys.
- `MERGE` assumed to guarantee correctness by itself.

## Five-minute model review

1. Say the fact grain aloud.
2. Check every dimension is single-valued at that grain.
3. Check every measure is true at that grain and classify additivity.
4. Inspect Type 2 joins, durable counts, and effective periods.
5. Find any many-to-many path and demand its allocation rule.
6. Find cross-fact analysis and require drill-across.
7. Explain late data, deletes, retries, and backfills.
8. Locate the governed metric definition.
9. Reconcile one real source example end to end.

## Core memory hooks

```text
Grain                 One row represents ______.
Transaction fact      One row per event.
Periodic snapshot     One row per entity/combination per period.
Accumulating snapshot One row per lifecycle instance.
SCD Type 1            Overwrite; reinterpret history.
SCD Type 2            New version row; preserve history.
Conformed dimension   Shared business context across processes.
Bridge                Controlled analytical many-to-many.
Drill-across           Aggregate facts separately, then align.
Semantic metric       One governed calculation contract.
```

For details, start with [Grain](../01-foundations/grain.md) or use the [modeling decision trees](../09-decision-guides/modeling-workflow-and-decision-trees.md).
