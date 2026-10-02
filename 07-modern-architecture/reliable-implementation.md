# Reliable Dimensional Implementation

A well-designed star can still produce a wrong dashboard if the loading pipeline duplicates events, misses deletes, or links a sale to the wrong customer version.

Consider a payment service that sends payment `P-42` twice because the first delivery timed out. A reliable pipeline recognizes both messages as the same payment and publishes one fact row. If the job fails halfway through and runs again, the result is still one row. This safe-to-retry behavior is called **idempotency**.

More generally, a reliable implementation gives the same explainable result when the same accepted input is processed again. It can also trace each published row back to source data and the transformation rules that produced it.

Kimball's extract-transform-load (ETL) chapters emphasize profiling, change data capture (CDC), cleansing, deduplication, surrogate-key management, late-data handling, restart/recovery, lineage, and version control. Modern extract-load-transform (ELT) tools change where this work runs; they do not remove the responsibilities.

## Start by writing the reliability contract

For each model, answer these questions before choosing an incremental loading technique:

```text
business process        What is measured?
grain                   What does one row represent?
business identity       How is the same event/entity recognized on replay?
source ordering         Which change wins?
history rule            Type 1, Type 2, event, snapshot, or interval?
incremental boundary    Which records are reconsidered?
delete/correction rule  How are reversals and removals represented?
reconciliation          Which counts and amounts must balance?
restart unit            What can be safely rerun?
publication rule        When is output complete enough for consumers?
```

Without these answers, an “incremental model” may be fast but nobody knows what should happen after a retry, late record, correction, or delete.

## Idempotency

An operation is idempotent when applying the same accepted input again produces the same target state.

### Mental model

> Retry must be safe.

Idempotency does not mean every job only inserts rows. It means the pipeline can recognize the same input and apply one predictable rule, preventing duplicate or conflicting results.

### Transaction fact example

For payment events, a source namespace plus payment event ID may be the business identity:

```sql
unique_key = (source_system, payment_event_id)
```

Before loading:

1. deduplicate transport copies;
2. order revisions using a source sequence or commit timestamp;
3. choose one deterministic winner;
4. resolve dimensional keys;
5. insert or update according to correction policy.

Do not use ingestion timestamp alone as identity. A replay has a new ingestion timestamp but is still the same business event.

For example, these messages should normally become one fact row:

| `payment_event_id` | `ingested_at` | `amount` | Meaning |
|---|---|---:|---|
| `P-42` | 10:01 | 80.00 | First delivery |
| `P-42` | 10:04 | 80.00 | Retry of the same payment |

The stable event ID tells the pipeline they are the same event. The changing ingestion time only tells us when each copy arrived.

### Snapshot example

For account-day balances:

```sql
unique_key = (account_durable_key, snapshot_date_key, balance_type, currency_key)
```

Rebuilding one date partition should replace or merge exactly that declared population, then reconcile expected accounts and totals.

## Change data capture

CDC reports inserts, updates, and deletes from source storage. It tells us what physically changed in a source table; it does not automatically tell us what the change means to the business.

```mermaid
flowchart LR
    A[Source row changes] --> B[CDC log]
    B --> C[Ordered, deduplicated changes]
    C --> D{Business interpretation}
    D --> E[Type 1 or Type 2 dimension]
    D --> F[Transaction/reversal fact]
    D --> G[Current state]
    E --> H[Dimensional marts]
    F --> H
    G --> H
```

An update to an order row might mean:

- correction of a typo;
- a business status transition;
- accumulation of a milestone;
- physical maintenance with no analytical meaning.

Decide what a source change means before choosing how to store it analytically.

### CDC completeness questions

- Does the feed include deletes and before-images?
- Is ordering guaranteed within a key or partition?
- Can records arrive more than once or out of order?
- How long are logs retained?
- What happens when the consumer checkpoint is lost?
- How is initial history aligned with ongoing changes?
- Can schema changes alter the payload silently?

Retain a restart checkpoint only after the target transaction and reconciliation succeed.

## Incremental loading patterns

An incremental load processes only data that may have changed instead of rebuilding everything. The right pattern depends on how late data, corrections, and deletes behave.

### High-water mark

Read rows after the last successfully processed source position. A source commit sequence that only moves forward is safer than a business timestamp that users or applications can edit.

Risk: late or backdated records may fall behind the watermark. Add an overlap window and deterministic deduplication, or use CDC with reconciliation.

### Sliding lookback

Reprocess a recent window, such as the last seven business dates, on every run.

Risk: a fixed window silently misses anything later than the window. Monitor lateness distribution and maintain an exception/backfill path.

### Partition replacement

Rebuild a complete affected slice, such as all facts for one date, from replayable input.

Benefit: deterministic and often simpler than row-level mutation. Risk: the partition date must match business impact; one late event may affect later balance partitions too.

### Key-based `MERGE`

Insert new rows and update existing rows by a stable identity. This combined operation is often called an **upsert**.

```sql
merge into dim_customer as d
using staged_customer as s
  on d.source_system = s.source_system
 and d.customer_bk = s.customer_bk
 and d.is_current = true
when matched and d.type1_hash <> s.type1_hash then
  update set ...
when not matched then
  insert (...);
```

This simplified example is not a complete Type 2 implementation. A single `MERGE` statement does not by itself guarantee correct close-and-insert ordering, non-overlapping effective periods, or concurrency safety.

### Append plus latest view

Retain every source revision and expose the latest accepted version through a view or compaction job.

Benefit: excellent traceability. Cost: consumers must not accidentally query all revisions, and storage/compaction rules must be governed.

## Deduplication

Two rows that look alike are not necessarily duplicates. First classify why both rows exist:

| Case | Same business event? | Response |
|---|---|---|
| Message retry with same event ID | Yes | Keep one deterministically |
| Updated revision of same source row | Same row, new assertion | Apply history/correction policy |
| Two legitimate purchases with identical values | No | Preserve both; identity is incomplete |
| Join fanout | Source rows may be unique | Fix relationship or pre-aggregate |
| Reversal | Separate business event | Preserve with signed/linked semantics |

A common staging pattern is:

```sql
select *
from (
  select
    s.*,
    row_number() over (
      partition by source_system, event_id
      order by source_revision desc, source_commit_ts desc, ingestion_id desc
    ) as rn
  from staged_events s
) ranked
where rn = 1;
```

Every ordering column needs a defined meaning. If two candidates can still tie, isolate them for investigation instead of choosing a different winner on different runs.

## Dimension loading

### Type 1

1. resolve source and durable identity;
2. compare Type 1 attributes or a deterministic hash;
3. overwrite changed values;
4. record audit metadata;
5. invalidate downstream aggregates/caches whose labels or grouping changed.

Type 1 changes reinterpret all facts joined to the row. That is the intended behavior only when history has no analytical value or a correction should apply everywhere.

### Type 2

1. resolve the current version by durable/business identity;
2. compare Type 2 attributes;
3. close the old interval at the new version's valid start;
4. insert a new row with a new surrogate key;
5. guarantee one current row and non-overlapping intervals;
6. resolve facts by event time.

If a change arrives retroactively, the pipeline may need to split an existing interval and rekey affected facts. See [Late-arriving data](../06-time-and-change/late-arriving-data.md).

### Unknown and inferred members

Create stable special rows before facts load. Use non-null foreign keys:

```text
-1 Unknown identity
-2 Not applicable
-3 Source key missing
-4 Invalid or rejected key
```

Use a unique inferred member rather than a shared sentinel when the natural key is known but its descriptive data is late.

### Surrogate-key lookup service

Whether implemented as SQL joins, cached mappings, or managed transforms, key resolution should have one governed contract:

```text
(source namespace, natural key, event time) -> dimension surrogate key
```

Do not generate independent surrogate keys for the same conformed dimension in each mart.

## Fact loading

### Validate dimensions before measures

For every fact row:

- confirm its business process and grain;
- resolve every mandatory dimension to a real or special member;
- reject or quarantine impossible relationships;
- classify measures and units;
- enforce event identity;
- attach audit lineage.

### Reversals and corrections

Choose one policy per process:

- **update in place** for a source-authoritative correction;
- **append reversal and replacement** for an accounting-style audit trail;
- **soft delete** with explicit status when consumers need exclusion plus traceability;
- **partition rebuild** when deterministic replay is the operational standard.

Never treat a missing record in a partial extract as a delete without proof that the extract is complete.

### Fact-table surrogate keys

An optional fact surrogate key can simplify ETL row identification, restart, or decomposing an update into delete/insert. It is not the business grain and should not be used to hide duplicate business events.

## Backfills

A **backfill** deliberately rebuilds past data, perhaps after fixing a bug or adding a column. Because it can change published history, treat it like a production migration rather than an ordinary retry.

### Backfill plan

1. state the reason and affected business dates/entities;
2. pin source snapshot and transformation-code version;
3. estimate downstream partitions and semantic caches affected;
4. run in an isolated target or shadow tables when risk is high;
5. reconcile rows, additive totals, distinct entities, and exception counts;
6. atomically publish or swap validated output;
7. retain the run manifest and rollback path.

### Reproducibility manifest

```text
pipeline_run_id
code_commit
configuration_version
source_extract_ids / table versions
watermark range
business-date range
schema version
row counts and control totals
quality-test results
published_at
```

A full refresh is not inherently more correct than incremental loading. It can reproduce today's source state while erasing historical states the source no longer holds. Replayability depends on retained evidence and deterministic rules.

## Schema evolution

Classify a source change before propagating it:

| Change | Typical response |
|---|---|
| New nullable descriptive attribute | Add to dimension or staging after semantic review |
| New measure at existing grain | Add only if valid for every relevant row; backfill/null policy required |
| Identifier meaning changes | New mapping/version; do not silently reuse old key |
| One source row becomes many | Grain change; create/version the model and backfill deliberately |
| Enum gains a value | Preserve unknown code, update decode and tests |
| Column removed | Keep contract temporarily or version consumers |
| Timestamp precision/time zone changes | Normalize explicitly and reassess uniqueness/windows |

Schema-on-read does not remove the need for a contract. It only delays when incompatible assumptions cause a failure.

## Source-system changes

Migrations create special key and history risks:

- natural keys may be renumbered or reused;
- old and new systems may run in parallel;
- attribute definitions may change;
- history may be truncated during conversion;
- the new source may emit different grain.

Namespace natural keys with source identity, maintain durable crosswalks for the same business entity, and define a cutover boundary. Do not join facts from two systems merely because their IDs look alike.

## Data-quality and audit models

An audit dimension can attach reusable load context to facts:

```text
dim_audit
  audit_key
  pipeline_run_id
  code_version
  source_extract_id
  quality_status
  inferred_member_count
  loaded_at
```

An error-event fact can record rejected or suspicious rows by source, rule, field, severity, and time. This turns pipeline quality into analyzable data rather than lost log messages.

Monitor:

- source-to-staging and staging-to-target row counts;
- control totals for money and quantities;
- duplicate business identities;
- null/special-key rates;
- Type 2 overlap and current-row uniqueness;
- inferred-member age;
- lateness distribution;
- referential integrity;
- closed-period drift;
- pipeline duration and freshness.

## Deployment, restart, and atomic publication

A pipeline should restart from a known checkpoint without double-applying work. Common approaches:

- write into staging then transact target changes;
- build new partitions/tables and swap only after validation;
- checkpoint source positions after successful target commit;
- record task-level state and dependencies;
- make cleanup safe for abandoned runs;
- separate “processed” from “published.”

Partial publication is often worse than a delayed publication because facts, dimensions, and aggregates can temporarily disagree.

## Optional: platform-specific notes

### dbt-style transformations

- Encode grain tests with `unique` or custom composite-key tests.
- Test relationships and accepted special members.
- Use incremental predicates as performance choices, not correctness definitions.
- Treat `unique_key` and late-arrival windows as explicit contracts.
- [dbt snapshots](https://docs.getdbt.com/docs/build/snapshots) provide Type 2-like source history, but downstream dimensions still need conformed keys and event-time fact lookup.

### Lakehouse tables

- Use ACID table formats for atomic merge/replace behavior.
- Compact small files and cluster by real access patterns after semantic correctness.
- Retain source versions long enough for the required replay window.
- Do not mistake storage time travel for a complete business temporal model.

### SAP HANA Cloud and Datasphere

- Push heavy transformations when it improves operations, but keep business rules versioned and testable.
- In calculation views, declare cardinality only when data actually satisfies it; an optimistic cardinality hint can yield wrong results.
- Keep semantic measure aggregation aligned with the fact's grain.
- For replication flows and remote sources, document latency, delete propagation, and snapshot consistency.
- Separate technical source IDs from conformed/durable entity identity.

## Common mistakes

- Increment only on maximum event timestamp.
- Use `MERGE` without a unique, stable business identity.
- Deduplicate by every column and accidentally delete legitimate repeated events.
- Generate surrogate keys independently in parallel marts.
- Apply Type 1 to every source update because it is easier.
- Advance a CDC checkpoint before targets and controls commit.
- Backfill facts without rebuilding snapshots, aggregates, or metric caches.
- Assume a full refresh can reconstruct history not retained by the source.
- Let schema drift silently change grain or units.
- Publish partially updated facts and dimensions.

## Related patterns

- [Grain](../01-foundations/grain.md)
- [Keys](../01-foundations/keys.md)
- [Slowly changing dimensions](../03-dimensions/slowly-changing-dimensions.md)
- [Late-arriving data](../06-time-and-change/late-arriving-data.md)
- [Layered architectures](layered-architectures.md)

## Beginner review checklist

- [ ] Can a retried event load without creating a second fact row?
- [ ] Is there a stable identity for each event or entity?
- [ ] Are late records, deletes, corrections, and reversals handled explicitly?
- [ ] Can the team rebuild a past period from retained input?
- [ ] Are fact rows linked to the dimension version valid at event time?
- [ ] Do counts and important monetary totals reconcile before publication?
- [ ] Are incomplete facts, dimensions, and aggregates prevented from appearing together?

## What to remember

1. Idempotency depends on stable business identity and deterministic ordering, not a particular command.
2. CDC captures physical changes; the model must interpret their business meaning.
3. Incremental boundaries need late-data and delete strategies plus reconciliation.
4. Preserve replayable source evidence and the exact code/configuration used for publication.
5. Backfills are governed migrations that include every affected derivative.
6. Schema evolution can be a grain change in disguise.
7. Restart, lineage, and quality controls are part of the analytical architecture.
