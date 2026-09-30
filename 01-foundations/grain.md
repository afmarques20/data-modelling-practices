# Grain

Grain is the semantic contract of a table. It answers the most important modeling question:

> **What exactly does one row represent?**

If that sentence is vague, the dimensions, measures, joins, tests, and dashboards built on the table will also be vague. A fast table with an ambiguous grain is still an incorrect model.

## The problem

Source systems rarely arrive with analytical grain already made explicit. An order header, order lines, payments, shipments, and status changes may all be available in one extract. A learning platform may emit starts, completions, assessment attempts, and daily progress. Combining these records because they share identifiers creates tables in which a row can mean several things.

The usual symptoms are familiar:

- totals change after joining a dimension or another fact table;
- analysts add `DISTINCT` without understanding why;
- a “unique” business key produces duplicates;
- some measures repeat across several rows;
- null-heavy columns appear because only some event types have each measure;
- incremental loads cannot decide whether to insert, update, or ignore a record.

These are often grain failures, not SQL failures.

## Mental model

Treat grain as a promise:

> Given the declared identifying dimensions and business identifiers, this table contains one row for one occurrence of the stated business event or state.

A useful formula is:

```text
grain = business process + unit of observation + time or lifecycle qualifier
```

Good declarations are complete sentences:

| Domain | Grain declaration |
|---|---|
| Learning | One row per learner's attempt at one assessment, submitted at one event time |
| SaaS | One row per account-feature usage event |
| E-commerce | One row per order line on a placed order |
| HR | One row per employee at the end of each calendar month |
| Banking | One row per account per business date, after end-of-day processing |

“One row per customer” is incomplete when customers have historical versions. “One row per order” is incomplete if the table contains line-level product keys. “Daily data” says nothing about the entity being observed.

## Declare grain before dimensions and measures

Kimball's dimensional design sequence is intentionally ordered:

1. Select one business process.
2. Declare the grain.
3. Identify dimensions that are valid at that grain.
4. Identify facts that are valid at that grain.

The order prevents wish-list modeling. For example, after declaring “one row per order line,” `product_key` and line quantity fit naturally. A shipment date does not necessarily fit: one order line may be fulfilled by several shipments. Adding it without changing the grain either loses shipments or duplicates the line.

Use these questions in a design workshop:

1. What business event, state, or lifecycle are we observing?
2. What makes two legitimate rows different?
3. At what time precision or reporting period is the observation made?
4. Can the same entity appear more than once? Why?
5. Which source record or combination of records proves the observation happened?
6. Which measures are true for every row at this grain?

Write the answer before drawing the schema.

## Atomic grain

Atomic grain is the lowest level of detail captured by the business process and useful to analysis. It does **not** mean the smallest detail theoretically imaginable.

- If an application records every feature interaction, one interaction is an atomic event.
- If a bank supplies only daily closing balances, account-day is the available atomic balance grain.
- If an HR source publishes one payroll result per employee, pay component, and pay period, that combination is the atomic payroll grain.

Atomic facts are the most flexible foundation because they can usually be rolled up into many summaries. They retain the dimensions available at the moment of measurement and support questions not anticipated when the pipeline was built.

An atomic table can still be physically partitioned, clustered, or exposed through aggregates. Atomicity is a statement about meaning, not storage layout.

### Atomic does not mean “put everything in one table”

Two low-level records may belong to different business processes:

- `course_completion` and `live_session_attendance` are different events;
- `order_line` and `payment_transaction` are different events;
- `account_transaction` and `account_daily_balance` are an event and a sampled state.

Keep them in separate fact tables. They can later be analyzed together through conformed dimensions and a controlled drill-across.

## Transaction, snapshot, and lifecycle grain

The three core fact-table patterns encode different kinds of grain:

| Pattern | Grain shape | Example |
|---|---|---|
| Transaction fact | One row per event | One row per card transaction |
| Periodic snapshot | One row per entity or dimensional combination per period | One row per account per day |
| Accumulating snapshot | One row per lifecycle instance | One row per support ticket, updated as milestones occur |

The pattern is not chosen from table size or refresh frequency. It is chosen from what a row means. See [Fact table patterns](../02-fact-tables/fact-table-patterns.md) for lifecycle and loading behavior.

## Fact grain and dimension grain

Dimensions also have grain:

- a Type 1 customer dimension is commonly one row per customer;
- a Type 2 customer dimension is one row per historical customer version;
- a date dimension is one row per calendar date;
- a product-category dimension is one row per governed category member.

This distinction matters when counting. `COUNT(DISTINCT customer_sk)` on a Type 2 dimension counts versions, not necessarily customers. Count a durable customer key when the question is about entities.

An aggregate fact table declares a higher grain than its atomic source. For example:

> One row per product category, country, and calendar month.

Its dimensions must be valid at that level, often through shrunken conformed dimensions. Do not attach order-line identifiers or customer-level attributes to that aggregate row.

## Worked examples

### Learning analytics: assessment attempts

Requirement: analyze attempts, scores, completion time, and pass rates.

**Declared grain**

> One row represents one submitted attempt by one learner for one assessment.

Candidate structure:

```text
fact_assessment_attempt
  attempt_id               -- degenerate business identifier
  learner_key              -- dimension version resolved at submission time
  assessment_key
  course_key
  submitted_date_key
  submitted_time_key
  attempt_number
  score_points
  possible_points
  duration_seconds
  passed_count
```

`attempt_id` should be unique if the source guarantees it. Otherwise the operational key might be `(source_system, assessment_id, learner_id, attempt_number)`.

A learner can attempt the same assessment repeatedly, so `(learner_key, assessment_key)` is not the grain. A course completion is also not an assessment attempt and belongs in another fact.

### E-commerce: header and line measures

Suppose order 9001 has three lines and a €12 order-level shipping charge.

At order-line grain, copying €12 to each line produces €36 when summed. Valid choices are:

1. keep shipping in an order-header fact with grain “one row per order”;
2. allocate the €12 to lines using a governed rule and store both the allocated amount and allocation method;
3. keep order-level analysis separate and drill across aggregated results.

Changing a column name from `shipping_amount` to `order_shipping_amount` does not fix the grain mismatch.

### HR: monthly headcount

**Declared grain**

> One row represents one employee's employment state at the final instant of one calendar month.

This grain requires rules:

- Is an employee on unpaid leave included?
- Which time zone defines month-end?
- Can an employee hold multiple concurrent assignments?
- Is the observed entity an employee or an employee-assignment?

If concurrent assignments matter, “employee-month” is too coarse. The correct grain may be “employee-assignment-month.”

### Banking: transactions and balances

`fact_account_transaction` may be one row per posted transaction. `fact_account_daily_balance` is one row per account per business date. Joining them row by row multiplies balances by the number of transactions.

For a report of daily transaction amount and closing balance:

1. aggregate transactions to account-day;
2. select the one account-day balance;
3. join the two result sets on conformed account and date keys.

This is a grain-alignment operation, not a direct fact-to-fact join.

## Mixed grain

A table has mixed grain when not every row represents the same kind of observation at the same level of detail.

Common examples:

- order headers and order lines in one fact;
- daily and monthly snapshots in the same table without a period-type dimension and disjoint semantics;
- both user events and account-level subscription state;
- actuals by cost center mixed with budgets by department;
- records from several event types with incompatible dimensionality.

Different record types can share a fact table only when they conform to one declared grain and the measures remain meaningful. A transaction table can contain sale and return rows if both are line-level business events with compatible dimensions and signed measures. “They come from the same source” is not sufficient.

### Never hide mixed grain with nulls

A table with `payment_amount` only on payment rows, `seat_count` only on subscription snapshots, and `feature_name` only on usage events is not a flexible universal fact. It is several processes encoded as null patterns. Queries become conditional, measures become easy to combine incorrectly, and tests no longer describe one contract.

## Changing grain

Grain drift occurs when a pipeline silently changes the meaning of a row:

- a source changes from one record per order to one per order line;
- events once unique by `event_id` begin arriving as revisions;
- a daily snapshot begins emitting several intraday states;
- a subscription model adds multiple products per subscription;
- historical backfills use a different deduplication rule from incremental loads.

Treat a grain change as a schema-contract change. Options include:

- publish a new fact table or versioned model;
- backfill all history to the new grain;
- preserve the old model and build an explicit bridge or allocation;
- add a genuinely new business process instead of overloading the old one.

Do not append the new rows and hope consumers infer the difference.

## How to validate grain

### 1. State the candidate key

Translate the grain sentence into columns. For an account-day balance:

```sql
select
    account_key,
    balance_date_key,
    count(*) as row_count
from fact_account_daily_balance
group by account_key, balance_date_key
having count(*) > 1;
```

Zero results support the hypothesis; they do not prove the business definition. Source corrections, multiple balance types, currencies, or account subledgers may reveal a missing grain component.

### 2. Explain every duplicate

Classify duplicates before deleting them:

- exact transport duplicate;
- later version of the same source event;
- legitimate repeated business event;
- fanout introduced by a join;
- missing grain attribute;
- source correction or reversal.

`SELECT DISTINCT` can suppress evidence and retain the wrong record.

### 3. Test measures against the grain

For every measure, finish this sentence:

> This value was observed or allocated for exactly one ______.

If the blank differs from the table's grain, move, derive, or allocate the measure.

### 4. Test dimensional cardinality

Every fact row should resolve to at most one member for each ordinary dimension role. A fact-to-dimension join that returns several rows often indicates:

- a natural-key join into a Type 2 dimension without an effective-date condition;
- duplicate dimension versions;
- a legitimate many-to-many relationship that needs a bridge;
- the wrong grain.

### 5. Reconcile from atomic to aggregate

Choose a known slice and reconcile:

```text
atomic rows
  -> expected business events
  -> grouped totals
  -> source control total
  -> dashboard metric
```

This catches grain loss between layers.

## Why this works

A declared grain constrains the model:

- dimensions describe the same observation;
- facts measure the same observation;
- keys identify the same observation;
- uniqueness tests become meaningful;
- incremental loads know what constitutes the same record;
- consumers know which aggregations are valid;
- separate business processes can integrate without being physically mixed.

The result is graceful extension: new dimensions or facts can be added when they are true at the existing grain, without changing the meaning of existing queries.

## When to split a table

Create a separate fact table when any of these are true:

- the row represents a different business event or state;
- the identifying dimensions differ materially;
- the time semantics differ, such as event time versus month-end;
- measures would repeat or require pervasive nulls;
- one record type is inserted while another is repeatedly updated;
- users need different aggregation rules.

Keep measures together when they are produced by the same measurement event at exactly the same grain. Splitting those measures arbitrarily creates unnecessary joins.

## When grain alone is not enough

A uniqueness constraint is not a complete model. Grain does not by itself define:

- whether a measure is additive;
- how history is preserved;
- whether a many-to-many relationship needs weighting;
- which business definition governs a metric;
- how late corrections are reconciled.

Use grain as the first constraint, then apply the relevant fact, dimension, key, and relationship patterns.

## Common mistakes

| Mistake | Symptom | Corrective action |
|---|---|---|
| Declaring grain as a topic, such as “sales data” | No stable uniqueness rule | Name the event/state, entity, and time qualifier |
| Designing columns before grain | Measures and dimensions contradict one another | Restart from the business process and grain sentence |
| Mixing header and line facts | Header totals multiply | Separate grains or allocate explicitly |
| Joining facts directly | Many-to-many fanout | Aggregate each fact to common conformed row headers, then drill across |
| Treating ingestion time as business grain | Retries create “new” facts | Preserve event identity; keep ingestion metadata as audit context |
| Assuming a source primary key defines analytical grain | Source revisions or line expansion create duplicates | Validate the business meaning and source lifecycle |
| Deduplicating without a rule | Legitimate events disappear | Classify duplicates and select records using governed precedence |
| Summing a snapshot across dates | Balances inflate | Respect semi-additivity and select the required snapshot |

## Tradeoffs

The lowest useful atomic grain usually increases row count and pipeline cost, but it preserves analytical flexibility and auditability. Higher-grain models can improve performance and usability for stable workloads, but they discard dimensions and make future questions impossible without rebuilding.

The architectural default is:

1. preserve source-fidelity records in an auditable layer;
2. publish atomic facts at a clear business grain;
3. add aggregates for performance, not as a substitute for atomic truth.

Exceptions are reasonable when detailed data is unavailable, prohibited, too sensitive, or economically unjustified. State the limitation explicitly.

## Modern implementation notes — later synthesis

This section maps the timeless grain principle to current engineering practice. The terminology and product guidance here are modern synthesis, not terminology attributed to Kimball.

### SQL and dbt-style pipelines

- Encode the candidate grain in uniqueness tests.
- Test every foreign key for non-nullness and accepted sentinel values.
- Deduplicate with a deterministic precedence rule, such as latest source revision then latest ingestion sequence.
- Keep event time, source modification time, and warehouse load time as separate columns.
- Make incremental `MERGE` keys match the declared observation, not merely the current file's apparent uniqueness.
- Run the same grain tests after backfills and full refreshes as after incremental runs.

A generated hash can implement a composite key, but the normalized input columns still define the grain. Changing trimming, case, null encoding, or hash inputs changes key identity and therefore requires migration planning.

### Lakehouses and columnar warehouses

Cheap storage and fast scans make atomic retention more feasible, but they do not make mixed grain safe. Partitioning and clustering should follow access patterns; they do not define business meaning. Materialized aggregates can accelerate common queries while the atomic fact remains authoritative.

### SAP HANA Cloud and SAP Datasphere

Calculation Views, analytical datasets, and semantic associations must preserve the fact's grain and cardinality. A join declared as many-to-one when the right side is actually many-to-many can produce plausible but inflated results. Validate cardinality with data tests rather than configuration alone, and define exception aggregation for snapshot or ratio measures.

### Streaming and CDC

An event stream can deliver the same logical event more than once. Exactly-once infrastructure claims do not remove the need for a business event identifier, revision policy, and correction semantics. CDC rows describe database changes; they are not automatically business facts. Reconstruct the business observation before publishing the dimensional fact.

## Design-review checklist

- [ ] The grain is written as “one row per …”
- [ ] The business process is singular and named
- [ ] Event, snapshot, or lifecycle semantics are explicit
- [ ] The time precision or reporting period is explicit
- [ ] A candidate key follows from the grain
- [ ] Every dimension is single-valued at the grain, or a bridge is designed
- [ ] Every measure is observed, derived, or allocated at the grain
- [ ] Header and line measures are not silently mixed
- [ ] Other fact tables are combined only after aggregation to a common grain
- [ ] Incremental, retry, correction, and backfill behavior preserve the same contract

## Related patterns

- [Dimensional modeling](dimensional-modeling.md)
- [Keys](keys.md)
- [Measures](measures.md)
- [Fact table patterns](../02-fact-tables/fact-table-patterns.md)
- [Advanced fact designs](../02-fact-tables/advanced-fact-designs.md)
- [Bridges and many-to-many relationships](../04-relationships/bridges-and-many-to-many.md)
- [Conformance and bus architecture](../05-enterprise-modeling/conformance-and-bus-architecture.md)
- [Event, state, and temporal modeling](../06-time-and-change/event-state-and-temporal-modeling.md)
- [Failure modes and review](../09-decision-guides/failure-modes-and-review.md)

## What I should remember

1. Grain is a semantic contract: one row represents one precisely stated observation.
2. Select the business process and declare grain before choosing dimensions or facts.
3. Atomic means the lowest useful detail captured by that process, not “everything in one table.”
4. Different business processes or time semantics normally require different fact tables.
5. A measure or dimension that is not valid at the grain does not belong in the row.
6. Uniqueness tests support a grain declaration; they do not replace business reasoning.
7. Modern platforms reduce storage and compute constraints, but they cannot repair ambiguous meaning.

## Source basis

This chapter synthesizes Kimball and Ross, *The Data Warehouse Toolkit*, 3rd ed., especially Chapters 2–4, 11, 16, and 18. See the repository's [coverage checklist](../COVERAGE.md) and [sources](../SOURCES.md).
