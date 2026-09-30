# Data Architect Interview Questions

Use these as explanation drills, not facts to memorize. A strong answer begins with the business question and grain, then explains alternatives, failure modes, and architectural consequences.

## Foundations

### 1. What is grain, and why declare it first?

A strong answer:

- defines grain as what exactly one row represents;
- explains that dimensions and facts are only valid relative to that row meaning;
- gives an atomic example;
- describes mixed-grain symptoms such as repeated measures and unexplained duplicates;
- says grain is a semantic contract, not merely a unique constraint.

### 2. Why not expose a normalized OLTP schema directly for analytics?

Discuss different workloads: operational schemas optimize small writes, integrity, and current state; dimensional schemas optimize understandable filtering, grouping, aggregation, and history. Normalization is not “bad,” and dimensional models do not replace operational systems.

### 3. What distinguishes a fact from a dimension attribute?

Facts are generally measurements at a process grain; dimension attributes provide descriptive context. Numeric data can be either: unit price may be a fact when calculated/aggregated and a band attribute when used for grouping. Explain intent, grain, and change behavior.

### 4. Why use surrogate keys when a source ID is unique?

The source ID may change, collide with another source, be reused, or identify an entity rather than a historical version. A surrogate decouples facts from source keys and supports SCD Type 2. Also distinguish the durable entity key.

### 5. What makes a star schema extensible?

At stable grain, new dimensions, facts, or descriptive attributes can often be added without invalidating existing queries. Changing grain or the definition of a conformed key is not a graceful extension.

## Fact-table design

### 6. Compare transaction, periodic snapshot, and accumulating snapshot facts.

State all three grains and write behaviors:

- one row per event; insert-oriented;
- one row per entity/combination per period; periodic inserts;
- one row per lifecycle instance; insert then milestone updates.

Explain that they may coexist for the same value chain.

### 7. When would you choose a factless fact?

Give both cases: event occurrence such as attendance, and coverage/eligibility such as which learner was expected to attend. The row count is the measure; anti-join coverage to activity to find absences.

### 8. How do you model an order with header discounts and line products?

Prefer line grain for product analysis. Repeat header dimensional context, not header numeric amounts. Keep header measures in a header fact or allocate them with a governed, reconciled rule. Preserve order number as a degenerate dimension.

### 9. When is a consolidated fact table safe?

Only when the processes can be expressed at exactly the same grain, dimensions, units, and compatible fact definitions. Otherwise keep facts separate and drill across.

### 10. Why can a fact surrogate key be useful, and what does it not solve?

It can support ETL restart, physical row identity, or delete/insert handling. It does not declare business grain and must not hide duplicated events.

## Measures

### 11. Explain additive, semi-additive, and non-additive facts.

Use revenue, balance, and percentage examples. Name the dimensions across which addition is invalid, especially time for balances.

### 12. Why should a warehouse usually store ratio components?

Ratios must be recomputed after grouping: `SUM(numerator) / SUM(denominator)`. Averaging row or subgroup percentages weights them incorrectly unless equal denominators are intended.

### 13. How would you model average daily balance?

Retain daily account balance snapshots, then define average over valid account-days, including rules for missing/closed days and currency. Do not sum balances through time or average pre-aggregated account averages without weights.

### 14. How do you define a distinct-customer metric with an SCD Type 2 dimension?

Distinct-count a durable entity key, not the version surrogate key. Define source integration, filters, and time perspective; a customer can have several Type 2 rows.

## Dimension history

### 15. Compare SCD Type 1 and Type 2.

Type 1 overwrites and reinterprets all joined history. Type 2 creates a new version with a surrogate key and effective interval, preserving event-time attribution. Choose per attribute based on analytical value, not because one is more sophisticated.

### 16. How does a fact join to an SCD Type 2 dimension?

During loading, find the version valid at the fact's event time and store its surrogate key. Ordinary queries join that key directly. Avoid natural-key joins to every version or current-only joins unless current perspective is explicitly requested.

### 17. What can go wrong with Type 2?

Overlapping intervals, several current rows, version counts mistaken for entity counts, rapid-change row explosion, late corrections, source-key reuse, and attributes historized without business need.

### 18. What are Types 3, 4, 6, and 7 for?

Recognition-depth answer: Type 3 stores a limited alternate/prior value; Type 4 separates a mini-dimension; Types 6/7 combine historical and current perspectives. Explain why Type 1/2 cover most everyday needs.

### 19. When is a mini-dimension appropriate?

For a correlated set of rapidly changing or frequently filtered attributes whose Type 2 changes would explode a large base dimension. Both base and mini keys normally sit on the fact. Band continuous values deliberately.

## Relationships and hierarchies

### 20. How do you model an account with several owners?

First ask whether a lower event grain identifies one owner. For account-month balances that relate to all owners, use an account-holder group/bridge. Use weights totaling 1 for allocated reporting or explicitly label unweighted impact reporting. Effective-date membership when it changes.

### 21. Why are many-to-many joins dangerous?

They multiply fact rows and destroy additivity. A bridge controls membership but does not automatically make a measure allocatable. Grain, weighting, and time perspective must be explicit.

### 22. When should a hierarchy be flattened?

Flatten stable, named, fixed-depth many-to-one levels for usability. Use a parent-child/hierarchy bridge for truly ragged, shared, or time-varying structures. Avoid generic level names that erase business semantics.

### 23. How would you model a chart of accounts that changes over time?

Preserve account identity, version or effective-date parent/path relationships, and provide an as-of date. A hierarchy bridge can store ancestor/descendant paths, depth, and allocation for shared ownership. Clarify whether reports use current or historical hierarchy.

## Enterprise integration

### 24. What is a conformed dimension?

More than a table with the same name: shared key mapping, attribute meanings, value domains, history policy, and governance across processes. Identical, row-subset, and attribute-subset dimensions can conform under controlled rules.

### 25. Explain the Kimball bus matrix.

Rows are business processes; columns are conformed dimensions. It is an integration blueprint, scope/roadmap, and governance tool. It enables incremental delivery without independent, incompatible marts.

### 26. How do you analyze orders and payments together without joining facts directly?

Aggregate each fact to common conformed headers such as customer-month-currency, then join the result sets. Explain the fanout risk of direct fact joins and when a new consolidated fact could be justified.

### 27. What is a shrunken dimension?

A row or attribute subset of a conformed base dimension, often used at aggregate grain or for a naturally higher-level process. It must retain compatible keys/values and not introduce a competing definition.

### 28. How do you model heterogeneous products?

Put genuinely common attributes and facts in a supertype/core model; use subtype-specific dimensions/facts for incompatible details. Avoid a giant sparse union. Provide conformed cross-product analysis at the intersection only.

## Time and late data

### 29. Distinguish event time and processing time.

Event time is when the business occurrence happened; processing time is when the platform handled it. Use event time for business-period attribution and Type 2 lookup, processing time for audit/freshness/replay.

### 30. What is the difference between an unknown and an inferred member?

Unknown means identity is absent/unresolved and may be shared. Inferred means a specific natural key is known before attributes; create one unique placeholder and complete it later, avoiding mass fact rekeying.

### 31. How do you handle a late fact?

Deduplicate it, preserve original event date, resolve dimension versions at event time, upsert idempotently, and identify snapshots/aggregates/closed periods requiring restatement. Reconcile the change visibly.

### 32. How do you handle a retroactive dimension correction?

Choose corrected-history, as-originally-known, or both. Corrected Type 2 history may require splitting intervals and rekeying affected facts. Both perspectives may justify bitemporal modeling.

### 33. Is an SCD Type 2 dimension bitemporal?

Not automatically. It often preserves valid-time versions but not the history of when the warehouse learned or superseded each assertion. Bitemporal requires both axes and retained prior beliefs.

## Modern architecture

### 34. How does dimensional modeling fit medallion architecture?

Bronze preserves source fidelity, Silver cleans/integrates, and dimensional models commonly serve Gold. Medallion assigns layer responsibilities; Kimball supplies analytical semantics. Neither term replaces the other.

### 35. Can Data Vault and Kimball coexist?

Yes: sources -> staging -> Raw Vault -> Business Vault -> dimensional information marts -> semantic layer. Vault emphasizes auditable, change-resilient integration; dimensional marts emphasize consumer usability and aggregation.

### 36. Must a modern dimensional model be a physical star?

No. Views, wide marts, HANA/Datasphere semantic models, or a semantic graph may express the contract. But grain, keys, history, cardinality, conformance, and measure behavior must remain explicit.

### 37. What belongs in a semantic metric definition?

Business purpose, base facts, numerator/denominator, aggregation, filters, entity identity, time dimension/window, currency/unit, valid dimensions, owner, tests, and change/version policy.

### 38. Why can't a semantic layer fix a bad warehouse model?

It can centralize joins and calculations, but cannot recover lost atomic detail, unmix grains, reconstruct overwritten history, or determine an unstated allocation rule.

## Data engineering and operability

### 39. What makes an incremental load idempotent?

Stable business identity, deterministic source ordering, explicit correction/delete semantics, atomic publication, and checkpoints advanced only after target success. `MERGE` alone is not the guarantee.

### 40. How do you distinguish duplicates from legitimate repeated events?

Use business identity and source revision, not equal values. Classify transport retry, revision, reversal, repeated event, and join fanout. Quarantine ambiguous ties rather than choose nondeterministically.

### 41. What must a safe backfill include?

Pinned source/code/configuration, exact business scope, shadow validation where needed, atomic facts, dimensions, snapshots, aggregates, semantic caches, reconciliations, run manifest, and rollback/publish strategy.

### 42. How can schema evolution break grain?

A source changing from one row per order to one row per line, adding multiple products per subscription, or changing timestamp precision can invalidate uniqueness and measure semantics. Treat it as a model contract change, not a nullable-column change.

## Scenario drills

### Scenario A: learning platform

Requirement: course starts, completions, live attendance, assessment attempts, learning hours, and department history.

Explain:

- which processes become separate facts and why;
- grain of every fact;
- user SCD strategy;
- pass-rate components;
- monthly active user entity key;
- instructor/skill many-to-many handling;
- late offline attendance;
- bus matrix and semantic metrics.

Compare your answer with [Learning analytics](../08-practical-designs/learning-analytics.md).

### Scenario B: e-commerce

Requirement: order margin, shipment performance, payments, refunds, and customer history.

Explain:

- order-line grain and header allocation;
- why shipments/payments are separate processes;
- role-playing dates;
- product/customer Type 2 choices;
- currency conversion;
- drill-across for order-to-cash metrics.

### Scenario C: SaaS

Requirement: feature adoption, active accounts, subscriptions, seats, MRR, and churn.

Explain:

- event facts versus account/subscription snapshots;
- definition of “active”;
- subscription lifecycle and amendments;
- MRR state and currency;
- durable account identity;
- late events and bot/test exclusions.

### Scenario D: HR

Requirement: current organization, historical organization, monthly headcount, skills, and compensation.

Explain:

- employee versus assignment grain;
- SCD Type 2 and current-perspective view;
- headcount semi-additivity;
- organization hierarchy as-of reporting;
- employee-skill bridge;
- privacy/security consequences.

### Scenario E: banking

Requirement: transactions, end-of-day balance, household exposure, and multi-currency reporting.

Explain:

- separate transaction and account-day grains;
- semi-additive balances;
- account-holder/household weighting or impact analysis;
- source and base currency facts;
- late back-valued transactions and snapshot restatement.

## What interviewers are usually testing

- Can you make grain explicit before talking about tools?
- Can you detect fanout and aggregation errors?
- Can you separate business history from pipeline timestamps?
- Can you explain when patterns are inappropriate, not just define them?
- Can you integrate domains without a monolithic model?
- Can you translate timeless semantics into the platform at hand?
- Can you design for replay, reconciliation, and governance as well as queries?

Use the [cheat sheet](cheat-sheet.md) for rapid review and the [architecture checklist](../09-decision-guides/failure-modes-and-review.md#architecture-review-checklist) for a full design critique.
