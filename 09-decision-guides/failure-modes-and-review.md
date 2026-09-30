# Failure Modes and Architecture Review

Bad dimensional models often announce themselves through query workarounds: unexplained `DISTINCT`, totals that change after joins, different KPI values across dashboards, or filters that work only for current data. Treat those symptoms as design evidence.

## Diagnostic table

| Symptom | Likely root cause | Corrective action |
|---|---|---|
| Totals multiply after adding a dimension | Many-to-many relationship or non-unique dimension join | Verify both grains; use the right dimension version, lower the fact grain, or introduce a governed bridge |
| `DISTINCT` appears throughout transformation and BI SQL | Duplicate source delivery, incomplete business identity, mixed grain, or join fanout | Classify each duplicate and fix the earliest responsible contract |
| Some measures populate only for certain row types | Several business processes forced into one sparse fact | Split processes into coherent fact tables |
| Order amounts repeat on every line | Header measure copied to line grain | Keep a header fact or allocate with a governed rule |
| Historical reports change after a customer update | Type 1 overwrite used for an attribute needing event-time history | Use Type 2 and resolve facts to the valid version |
| Historical facts show today's department | Fact joined to current dimension row using natural key | Store and join the event-time surrogate version key |
| Customer count exceeds known customers | Type 2 surrogate versions counted as entities | Distinct-count the durable entity key |
| Daily balances produce enormous monthly totals | Semi-additive state summed across time | Use period-end, last non-empty, or governed average-balance logic |
| Regional percentage is the average of store percentages | Non-additive ratios aggregated directly | Store/sum numerator and denominator, then divide |
| Dashboards disagree on “active customer” | Business definition rebuilt in each report | Define and test one governed semantic metric |
| Fact table contains long descriptions and status labels | Dimensions not separated from measurements | Move descriptive context to dimensions; retain only legitimate degenerate identifiers |
| Dimension queries require many snowflake joins | Operational normalization copied into presentation | Flatten stable many-to-one descriptive hierarchies |
| One dimension row change produces millions of versions | Rapid attributes put in a large Type 2 base dimension | Reclassify history needs or use a mini-dimension/fact event |
| Facts have null foreign keys and disappear from reports | Missing special dimension members | Use explicit unknown, not-applicable, or inferred members |
| Same source ID refers to different entities | Natural keys collide across systems or are reused | Namespace source keys and maintain durable warehouse identity |
| Payment and shipment facts duplicate each other when joined | Direct fact-to-fact join at incompatible grains | Aggregate separately to conformed headers and drill across |
| Daily and monthly rows coexist with conditional logic | Mixed snapshot grain | Separate facts or enforce a genuinely coherent period-type design |
| Pipeline retry doubles yesterday's events | No stable business identity or idempotent load | Deduplicate deterministically and upsert/replace by declared grain |
| Backfill changes unrelated periods | Business date and processing date confused | Scope by business impact and publish a reconciliation manifest |
| BI tool creates ambiguous paths | Multiple relationships without explicit roles | Use role-playing dimension views/names and one governed path |

## Failure mode 1: mixed grain

### Symptom

- mutually exclusive measure columns;
- repeated values at one level and detailed values at another;
- keys that are unique only after adding `row_type`;
- every query begins with an event-type filter.

### Root cause

The table was designed from available source columns or a report layout rather than one business process and grain sentence.

### Fix

Split the table by process/grain. State “one row represents...” for each output. Integrate only after aggregation through conformed dimensions.

### Red flag

> “It is one big fact so users can find everything in one place.”

Convenience is not achieved when every measure has a different validity rule.

## Failure mode 2: unexplained duplicates and excessive `DISTINCT`

### Symptom

Removing `DISTINCT` changes totals, but nobody can explain which rows are false duplicates.

### Root cause

Possibilities include transport retry, legitimate repeated events, source revisions, SCD version fanout, bridge fanout, or missing grain columns.

### Fix

Trace one concrete duplicate pair. Compare business identity, source revision, and join path. Decide whether to retain, revise, reverse, allocate, or deduplicate. Encode the rule as a test.

`DISTINCT` is valid when the requirement itself asks for a set, such as distinct durable customers. It is not a general data-quality repair.

## Failure mode 3: uncontrolled many-to-many joins

### Symptom

Revenue doubles when filtering by customer segment, employee skill, diagnosis, role, or campaign.

### Root cause

One fact legitimately relates to several dimension members, but the join is treated as one-to-many.

### Fix

First ask whether a more atomic fact would produce one member per row. If not, use a bridge with an explicit grain. For additive allocation, weights per bridge group should total 1. For impact analysis, intentionally repeat the full fact and label totals non-additive.

## Failure mode 4: joining facts directly

### Symptom

Orders × shipments × payments creates a large intermediate table and inflated totals.

### Root cause

Each fact has several rows for the shared key; their join creates a Cartesian multiplication within that key.

### Fix

Aggregate each fact independently to identical conformed dimension headers, then align results. If one shared lower grain genuinely exists, design a new fact at that grain rather than relying on accidental joins.

## Failure mode 5: ratios stored without components

### Symptom

Team-level average pass rate differs depending on grouping order.

### Root cause

Percentages or averages were treated as additive facts.

### Fix

Store `passed_count` and `graded_attempt_count`, or `score_points` and `possible_points`. Sum components at query grain, then divide. For a weighted average, store the weight and weighted component explicitly.

## Failure mode 6: wrong historical join

### Symptom

All past sales move into the customer's current region, or Type 2 joins duplicate a fact across versions.

### Root cause

The fact stores a natural key and joins to every historical version, or joins to the current row.

### Fix

At load time, resolve the dimension surrogate key valid at event time and store it in the fact. Ordinary queries join surrogate key to surrogate key. Provide a separate current-perspective view when required.

## Failure mode 7: natural keys used as warehouse relationships

### Symptom

Source migration breaks joins; IDs collide; key reuse attaches a new entity to old facts.

### Root cause

A source-controlled identifier was treated as permanent enterprise identity and historical version identity.

### Fix

Namespace source keys, establish a durable entity key/crosswalk, and use warehouse surrogate keys for dimensional versions.

## Failure mode 8: unnecessary Type 2 history

### Symptom

Dimensions explode in size, simple counts require complex logic, and users cannot explain why historical versions matter.

### Root cause

Every changed attribute was historized by default.

### Fix

Classify attributes by analytical need. Use Type 1 for corrections/current-only attributes, Type 2 for meaningful historical attribution, Type 0 for fixed originals, and mini-dimensions or facts for rapidly changing analytical state.

## Failure mode 9: excessive snowflaking

### Symptom

Every product query joins product -> subcategory -> category -> division, and different tools reproduce the chain differently.

### Root cause

Operational normalization was retained in a business-facing model to save repeated text or mimic source ownership.

### Fix

Flatten stable, many-to-one descriptive hierarchies into the base dimension. Keep an outrigger only when it has distinct governance/reuse that outweighs query complexity. Use a hierarchy bridge for truly ragged structures, not a snowflake of fixed tables.

## Failure mode 10: semantic logic scattered across dashboards

### Symptom

Finance, sales, and operations publish different revenue for the same month.

### Root cause

The warehouse provides fields but no governed metric contract.

### Fix

Centralize the numerator, denominator, filters, currency policy, time behavior, and entity identity. Version it, test it, and reuse it through the semantic layer.

## Additional high-value anti-patterns

### Designing from one report

A report captures one question and layout. Use it as a requirement source, not the schema blueprint. Model the underlying business process at a durable grain.

### Text in the fact table

Freeform or descriptive text bloats facts and bypasses consistent labeling. Put reusable descriptors in dimensions. A degenerate identifier belongs in the fact only when it has no useful dimension attributes.

### Centipede fact tables

Too many tiny dimensions can make a fact hard to use. Group low-cardinality unrelated flags in a junk dimension; keep genuinely descriptive entities distinct. Do not combine dimensions solely to reduce foreign-key count.

### Optimizing before correctness

Partitions, indexes, clustering, materialized views, and engines can accelerate a wrong answer. Validate grain, relationships, and aggregation first. Add transparent aggregates only after atomic facts reconcile.

### Hidden unknowns

Null or magic default values make missing, not applicable, invalid, and late identities indistinguishable. Use explicit special members and monitor their rates.

### Model says “real time” but publishes inconsistent layers

Low latency is not valuable if facts arrive before dimensions, metrics mix closed and open periods, or retries duplicate data. Define freshness together with completeness and consistency.

## Architecture review checklist

### Business process and grain

- [ ] Each fact models one named business process.
- [ ] Every fact has one precise “one row represents...” statement.
- [ ] Candidate business identity is documented and tested with real data.
- [ ] Header, line, event, state, and lifecycle grains are not mixed.
- [ ] Atomic grain is retained unless a documented limitation or aggregate purpose applies.

### Dimensions and keys

- [ ] Each dimension is single-valued at fact grain or uses a governed bridge.
- [ ] Natural keys are namespaced by source where necessary.
- [ ] Durable entity identity is distinct from Type 2 version identity.
- [ ] Surrogate-key rules are shared across conformed marts.
- [ ] Unknown, not-applicable, invalid, and inferred members have distinct semantics.
- [ ] Role-playing dates and other roles have clear consumer names.

### History and time

- [ ] Every change-tracked attribute has an explicit Type 0/1/2/etc. reason.
- [ ] Type 2 intervals do not overlap; one current row exists per entity.
- [ ] Fact keys are resolved using the correct event-time version.
- [ ] Event time, processing time, and reporting-period time are distinguished.
- [ ] Late facts/dimensions and retroactive corrections have a restatement policy.
- [ ] Current, historical, and as-known perspectives are named clearly.

### Facts and metrics

- [ ] Every measure is valid at fact grain.
- [ ] Additive behavior is documented for every dimension, especially time.
- [ ] Ratios/averages store sufficient additive components.
- [ ] Currency, unit, sign, precision, null, and zero semantics are explicit.
- [ ] Distinct counts use the intended entity key.
- [ ] Header measures are separate or allocated with a governed rule.

### Relationships and enterprise integration

- [ ] Bridge grain, weights, time validity, and impact/allocated behavior are documented.
- [ ] Cross-process queries use conformed dimensions and multipass aggregation.
- [ ] Shared dimensions/facts have a named steward and compatibility contract.
- [ ] The bus matrix shows current and planned process integration.
- [ ] Heterogeneous products share only genuinely common facts/attributes.

### Pipeline reliability

- [ ] Source changes, deletes, reversals, and revisions have explicit rules.
- [ ] Incremental loads are idempotent and safely restartable.
- [ ] Deduplication uses stable business identity and deterministic ordering.
- [ ] Backfills identify every affected fact, snapshot, aggregate, and cache.
- [ ] Raw/source evidence and run lineage support reproducibility.
- [ ] Counts, control totals, referential integrity, lateness, and special-key rates are monitored.

### Usability and governance

- [ ] Business labels and hierarchies are understandable without source expertise.
- [ ] No unnecessary snowflake or ambiguous join path is exposed.
- [ ] Metrics are governed outside individual dashboards.
- [ ] Current platform semantics match warehouse aggregation behavior.
- [ ] Security and retention apply consistently through dimensions and facts.
- [ ] Tradeoffs and rejected alternatives are recorded.

## Final quality review rubric

Score each dimension from 0 to 2:

| Dimension | 0 | 1 | 2 |
|---|---|---|---|
| Coverage | Major requirements missing | Present but shallow | Complete at intended priority |
| Redundancy | Competing explanations | Some avoidable repetition | One canonical home with links |
| Accuracy | Grain/joins/metrics unclear | Mostly correct, edge cases open | Examples and rules internally consistent |
| Practicality | Theory only | Some actionable guidance | Requirement-to-model workflow is executable |
| Clarity | Requires source-book knowledge | Understandable with effort | Standalone and concise |
| Architecture depth | Describes patterns only | Some tradeoffs | Explains why, consequences, and alternatives |
| Modern relevance | Old physical assumptions repeated | Modern terms appended | Principles translated with guardrails |
| Conciseness | Bloated or fragmented | Uneven | High understanding per minute |

A repository is not finished because every file exists. Resolve any zero, investigate ones in Tier 1, validate all internal links, and reread the examples specifically for grain and aggregation errors.

## What to remember

1. Query workarounds are often model diagnostics.
2. Never suppress unexplained duplicates.
3. Historical correctness requires the right dimension version key, not a current natural-key join.
4. Ratios, snapshots, and bridges need explicit aggregation rules.
5. Integrate facts through conformed headers and drill-across, not direct many-to-many joins.
6. Review model semantics, pipeline behavior, and governed metrics together.
