# Late-Arriving Data

Late data is a time-alignment problem, not merely a slow-pipeline problem. A record is late when it becomes available after the warehouse has already processed the business time to which it belongs.

The governing rule is:

> Model the business event at the time it happened, while separately recording when the platform learned about it.

## The problem

A course completion from Monday arrives on Thursday. A customer event arrives before the customer's profile. A bank posts a back-valued transaction after the daily balance was published. An HR correction says an employee actually moved departments last month.

If the load simply uses current dimension values and today's date, the data becomes operationally convenient but historically false. If it silently rewrites old outputs, previously published results become irreproducible. A robust design makes lateness, correction, and restatement policies explicit.

## Mental model: two clocks, sometimes more

At minimum, distinguish:

| Clock | Meaning | Example |
|---|---|---|
| Event time | When the business event or state was valid | Course completed at 2026-03-02 14:12 UTC |
| Processing time | When this pipeline handled it | Loaded on 2026-03-05 01:00 UTC |

Useful operational timestamps often include source commit time, extraction time, ingestion time, transformation run time, and warehouse publication time. These are lineage fields, not substitutes for event time.

```text
business event      source commits       platform ingests      mart publishes
     t0 ---------------- t1 ------------------- t2 ------------------ t3
     |<-------------------------- observed lateness ----------------->|
```

Use event time for historical dimension resolution and business-period attribution. Use processing timestamps for observability, service-level monitoring, replay, and audit.

## Late-arriving facts

A late-arriving fact describes an event that occurred earlier than its arrival in the analytical pipeline.

### Grain

> **One row represents one business event at its original event time, regardless of when it was loaded.**

Example: one row in `fact_course_completion` represents one user's completion attempt for one course enrollment.

### Resolve dimensions as of event time

Suppose user `U-42` moved from the Retail division to Enterprise on March 1. A February 27 completion arrives on March 4. The fact should normally point to the Retail version of the Type 2 user dimension because that version was valid at the event time.

```sql
select user_sk
from dim_user
where source_system = :source_system
  and user_bk = :user_bk
  and :event_ts >= valid_from
  and :event_ts < valid_to;
```

Use half-open effective intervals — `[valid_from, valid_to)` — so adjacent versions do not overlap at a boundary. Once the lookup returns the surrogate key, the fact stores it. Normal analytical joins use that key; they do not repeat the date-range lookup on every query.

### Choose the business date deliberately

For the late completion, `completion_date_key` comes from the completion timestamp, not the load date. Keep `loaded_at` or an audit dimension separately.

```text
fact_course_completion
  completion_id
  completion_date_key  -> 20260302
  user_key             -> version valid on 2026-03-02
  loaded_at            -> 2026-03-05T01:00:00Z
  pipeline_run_key     -> audit context
```

### Effects on snapshots

Late events can change previously published state. A back-valued bank transaction may affect:

- the transaction fact for its original posting date;
- the account's closing balance on that date;
- every later balance if the source supplies deltas rather than authoritative closings;
- monthly averages and downstream aggregates.

The snapshot policy must say whether to:

1. restate every affected closed period;
2. post an adjustment in the current period;
3. publish both originally reported and restated values;
4. defer to a source-provided authoritative balance.

There is no universally correct policy. Finance, regulatory, and operational users may require different views, but those views must be named and governed.

### Load pattern

An incremental job needs a lookback or change feed, not only `source_timestamp > max_loaded_timestamp`.

```sql
-- Pseudocode: deduplicate by stable event identity before merging.
merge into fact_course_completion as target
using staged_latest_event as source
  on target.source_system = source.source_system
 and target.completion_id = source.completion_id
when matched and source.revision > target.source_revision then
  update set ...
when not matched then
  insert (...);
```

`MERGE` is only a mechanism. Correctness comes from a stable business identity, deterministic revision ordering, historical dimension lookup, and a rule for source deletes or reversals.

## Late-arriving dimensions

A late-arriving dimension occurs when a fact references an entity whose descriptive row is not yet available, or whose corrected history arrives after facts were keyed.

### Unknown member versus inferred member

These solve different problems:

| Pattern | Meaning | Key behavior | Later action |
|---|---|---|---|
| Unknown member | The identity is genuinely absent or cannot be resolved | Many unrelated facts may share one sentinel surrogate key | Investigate or leave unknown |
| Not applicable member | This dimension does not apply to the fact | Shared, explicit sentinel key | No reconciliation expected |
| Inferred member | The natural key is known, but attributes have not arrived | Create a unique surrogate key for that natural key | Fill attributes later, usually as Type 1 |

If a completion contains user ID `U-99` but the user feed is late, do not map it to the same generic unknown user used for every missing ID. Create an inferred row:

```text
user_sk        78123
user_bk        U-99
user_name      Unknown pending source
department     Unknown pending source
is_inferred    true
valid_from     earliest defensible time
valid_to       9999-12-31
is_current     true
```

The fact immediately points to `78123`. When the user row arrives, populate that same version with a Type 1 completion if no earlier history is required. This avoids rekeying every fact that arrived first.

### Grain of an inferred row

> **One inferred dimension row represents one specifically identified business entity whose descriptive attributes are pending.**

It is not one row per missing event and not one shared row for all unknown entities.

### When the arriving row reveals history

Suppose the eventual payload says `U-99` belonged to Department A until February 15 and Department B afterward. A single inferred current row is insufficient for facts on both sides of that date.

Possible repair:

1. construct non-overlapping Type 2 versions;
2. resolve affected facts by their event time;
3. update their surrogate foreign keys when the business requires corrected historical truth;
4. rebuild affected aggregates and semantic caches;
5. retain lineage showing the restatement run.

This is a controlled historical repair, not a routine Type 1 fill.

## Retroactive dimension changes

A source may report today that an attribute became valid last month. The warehouse then knows both:

- **valid time:** when the statement was true in the business;
- **system time:** when the warehouse learned and recorded it.

Three common policies are:

### Corrected business history

Insert or split Type 2 versions at the retroactive boundary and rekey affected facts. Use this when reports are meant to reflect the best current understanding of what was true then.

### As-originally-known history

Do not rekey old facts; retain what the warehouse knew when it published them. Use this when reproducing prior regulatory or operational reports matters more than retrospective correction.

### Both views

Preserve system-time history or a publication snapshot as well as corrected valid-time history. This approaches bitemporal modeling and costs more in storage, processing, and consumer education. See [Event, state, and temporal modeling](event-state-and-temporal-modeling.md).

Document the policy per subject area. “Historical truth” is ambiguous until the organization chooses which clock it means.

## Reconciliation workflow

Late-data handling should be observable and repeatable.

```mermaid
flowchart LR
    A[Land source record unchanged] --> B[Identify business key and event time]
    B --> C{Dimension member available?}
    C -- Yes --> D[Resolve version valid at event time]
    C -- No, identity known --> E[Create or reuse inferred member]
    C -- Identity absent --> F[Use explicit unknown member]
    D --> G[Upsert fact idempotently]
    E --> G
    F --> G
    G --> H[Record lateness and audit context]
    H --> I[Reconcile counts, amounts, and affected periods]
    I --> J{Restatement required?}
    J -- Yes --> K[Repair facts, snapshots, aggregates, and caches]
    J -- No --> L[Close exception with policy reason]
```

Maintain an exception queue or reconciliation table with:

- source system and business identifier;
- event time and first-seen time;
- reason: late fact, missing dimension, retroactive correction, delete, or reversal;
- chosen remediation;
- affected partitions and downstream models;
- reconciliation status and audit run.

## Tests that expose late-data defects

### Referential completeness

Every fact foreign key should resolve, including special members. Null foreign keys make joins silently drop rows.

### Inferred-member aging

Alert when inferred members remain incomplete beyond the expected source delay. Track their facts and business impact.

### Effective-period integrity

For each durable entity, Type 2 ranges should not overlap. Decide whether gaps are allowed.

### Event-to-version consistency

For sampled facts, verify that the stored dimension surrogate key was valid at the fact's event time. Exceptions may be legitimate only when an “as originally known” policy applies.

### Closed-period drift

Compare period totals before and after late processing. Require an explained reconciliation rather than allowing silent changes.

### Idempotent replay

Reprocessing the same landed records should produce the same facts, dimensions, and totals. Test this explicitly.

## Common mistakes

| Mistake | Consequence | Better response |
|---|---|---|
| Use load date as business date | Events move into the wrong reporting period | Keep event and processing clocks separately |
| Resolve every late fact to the current dimension row | Historical facts acquire future attributes | Look up the Type 2 version valid at event time |
| Send every missing natural key to one unknown member | Entity identity is lost and facts require mass rekeying | Create an inferred member when identity is known |
| Create a new inferred member for every retry | Duplicate dimension rows and inconsistent fact keys | Enforce uniqueness on source namespace plus natural key |
| Treat all late changes as Type 1 | Historical meaning silently changes | Apply the subject area's Type 1/2 and restatement policy |
| Refresh only the atomic fact | Snapshots, aggregates, and metric caches disagree | Trace and rebuild every affected derivative |
| Use a fixed lookback without reconciliation | Records later than the window disappear | Combine CDC/watermarks with exception and completeness checks |
| Hide lateness | Consumers cannot reproduce published results | Expose audit timestamps and publication/restatement status |

## Tradeoffs

| Choice | Benefit | Cost |
|---|---|---|
| Larger incremental lookback | Captures more delayed data | More scanning and repeated work |
| Inferred members | Facts load immediately without losing identity | Temporary incomplete descriptions and reconciliation state |
| Historical fact rekeying | Corrected event-time truth | Expensive updates and changed prior outputs |
| Immutable facts plus adjustments | Strong audit trail | More complex consumer logic |
| Bitemporal history | Reproduce both valid and known-at-the-time views | More rows, keys, tests, and semantic complexity |

## Modern implementation notes

- Keep an immutable or replayable landing layer so historical repairs do not depend on the current source state.
- CDC reduces extraction delay but does not eliminate out-of-order arrival, missing parents, or retroactive corrections.
- dbt snapshots can preserve mutable source history, but a snapshot does not choose the fact's business grain or restatement policy for you.
- In lakehouse tables, partition replacement can make historical repair efficient; always derive the affected partition set from business dates, not only load dates.
- In SAP HANA or Datasphere, time-dependent dimensions and validity associations can express effective dating, but the pipeline still needs deterministic event-time key resolution.
- Publish data freshness and restatement metadata alongside metrics when users make operational decisions from incomplete periods.

## Related patterns

- [Keys](../01-foundations/keys.md)
- [Slowly changing dimensions](../03-dimensions/slowly-changing-dimensions.md)
- [Fact-table patterns](../02-fact-tables/fact-table-patterns.md)
- [Reliable implementation](../07-modern-architecture/reliable-implementation.md)
- [Event, state, and temporal modeling](event-state-and-temporal-modeling.md)

## What to remember

1. Preserve event time and processing time as different facts about the record.
2. Resolve a late fact to the dimension version valid when the event occurred.
3. Use a unique inferred member when identity is known but attributes are late; use a shared unknown member only when identity truly is not known.
4. Late events can require snapshot, aggregate, and cache restatement — not just one atomic insert.
5. Decide whether history means corrected business truth, originally published truth, or both.
6. Make replay idempotent and reconciliation visible.
