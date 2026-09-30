# Dimensional Modeling Field Guide

> Kimball for a modern data engineer, analytics engineer, or future data architect.

This repository compresses the most durable and useful ideas from Ralph Kimball and Margy Ross's *The Data Warehouse Toolkit, Third Edition* into a practical handbook. It is not a chapter-by-chapter summary. It reorganizes the material around decisions that recur in real projects: choosing a grain, separating business processes, preserving history, controlling aggregation, integrating marts, and turning source data into trustworthy analytical products.

The target is roughly **95% of the book's practical modeling value** in a form that is faster to study and easier to reuse during implementation, design reviews, architecture discussions, and interviews.

## The rule that prevents most modeling failures

Before choosing columns, write one precise sentence:

> **One row represents ...**

Then use Kimball's four-step design process in order:

1. **Choose one business process.** What real-world activity or state are we measuring?
2. **Declare the grain.** What exactly does one row represent?
3. **Identify the dimensions.** Which descriptive contexts apply at that exact grain?
4. **Identify the facts.** Which numeric observations are valid at that exact grain?

A plausible-looking table with ambiguous grain is still a broken model.

## Recommended reading order

### Must master

1. [Grain](01-foundations/grain.md)
2. [Dimensional modeling](01-foundations/dimensional-modeling.md)
3. [Keys](01-foundations/keys.md)
4. [Fact-table patterns](02-fact-tables/fact-table-patterns.md)
5. [Measures and additivity](01-foundations/measures.md)
6. [Slowly changing dimensions](03-dimensions/slowly-changing-dimensions.md)
7. [Conformance and bus architecture](05-enterprise-modeling/conformance-and-bus-architecture.md)
8. [Bridges and many-to-many relationships](04-relationships/bridges-and-many-to-many.md)
9. [Hierarchies](04-relationships/hierarchies.md)
10. [Late-arriving data](06-time-and-change/late-arriving-data.md)
11. [Modeling workflow and decision trees](09-decision-guides/modeling-workflow-and-decision-trees.md)

### Know well

- [Advanced fact designs](02-fact-tables/advanced-fact-designs.md)
- [Reusable dimension patterns](03-dimensions/dimension-patterns.md)
- [Dates, times, and calendars](03-dimensions/date-time-calendars.md)
- [Multiple processes and heterogeneous models](05-enterprise-modeling/multi-process-and-heterogeneous-models.md)
- [Event, state, and temporal modeling](06-time-and-change/event-state-and-temporal-modeling.md)
- [Semantic layers and governed metrics](07-modern-architecture/semantic-layer-and-metrics.md)
- [Reliable incremental implementation](07-modern-architecture/reliable-implementation.md)

### Recognize

- Hybrid SCD types, outriggers, hot-swappable dimensions, and audit dimensions
- Shrunken dimensions, aggregate navigation, timespan facts, and complex snapshot variants
- [Medallion, Data Vault, and dimensional marts](07-modern-architecture/layered-architectures.md)
- Which implementation details were shaped by older warehouse technology and which modeling rules remain timeless

## Choose a route

| Goal | Route |
|---|---|
| Design a model now | [Decision trees](09-decision-guides/modeling-workflow-and-decision-trees.md) -> relevant pattern -> [review checklist](09-decision-guides/failure-modes-and-review.md#architecture-review-checklist) |
| Fix duplicates or wrong totals | [Grain](01-foundations/grain.md) -> [Measures](01-foundations/measures.md) -> [Failure modes](09-decision-guides/failure-modes-and-review.md) |
| Preserve history | [Keys](01-foundations/keys.md) -> [SCDs](03-dimensions/slowly-changing-dimensions.md) -> [Late data](06-time-and-change/late-arriving-data.md) |
| Integrate domains | [Conformance and bus architecture](05-enterprise-modeling/conformance-and-bus-architecture.md) -> [Multiple processes](05-enterprise-modeling/multi-process-and-heterogeneous-models.md) |
| Translate to a modern platform | [Layered architectures](07-modern-architecture/layered-architectures.md) -> [Semantic layer](07-modern-architecture/semantic-layer-and-metrics.md) -> [Reliable implementation](07-modern-architecture/reliable-implementation.md) |
| Prepare for an interview | [Cheat sheet](10-reference/cheat-sheet.md) -> [Interview questions](10-reference/architect-interview-questions.md) |

## Handbook map

### 1. Foundations

- [Grain](01-foundations/grain.md) — the row-level contract
- [Dimensional modeling](01-foundations/dimensional-modeling.md) — facts, dimensions, stars, snowflakes, and normalized operational models
- [Keys](01-foundations/keys.md) — natural, durable, surrogate, and special members
- [Measures](01-foundations/measures.md) — additive, semi-additive, non-additive, and safe metric construction

### 2. Fact tables

- [Fact-table patterns](02-fact-tables/fact-table-patterns.md) — transaction, periodic snapshot, accumulating snapshot, and factless facts
- [Advanced fact designs](02-fact-tables/advanced-fact-designs.md) — aggregates, consolidation, header allocation, currencies, units, lags, and timespans

### 3. Dimensions

- [Slowly changing dimensions](03-dimensions/slowly-changing-dimensions.md)
- [Dimension patterns](03-dimensions/dimension-patterns.md)
- [Dates, times, and calendars](03-dimensions/date-time-calendars.md)

### 4. Relationships

- [Bridges and many-to-many relationships](04-relationships/bridges-and-many-to-many.md)
- [Hierarchies](04-relationships/hierarchies.md)

### 5. Enterprise modeling

- [Conformance and bus architecture](05-enterprise-modeling/conformance-and-bus-architecture.md)
- [Multiple processes and heterogeneous models](05-enterprise-modeling/multi-process-and-heterogeneous-models.md)

### 6. Time and change

- [Late-arriving data](06-time-and-change/late-arriving-data.md)
- [Event, state, and temporal modeling](06-time-and-change/event-state-and-temporal-modeling.md)

### 7. Modern architecture

- [Layered architectures](07-modern-architecture/layered-architectures.md) — medallion, Data Vault, and dimensional marts
- [Semantic layers and metrics](07-modern-architecture/semantic-layer-and-metrics.md)
- [Reliable implementation](07-modern-architecture/reliable-implementation.md) — CDC, idempotency, MERGE, deduplication, backfills, and schema evolution

### 8. Practical designs

- [Learning analytics](08-practical-designs/learning-analytics.md)
- [Domain case studies](08-practical-designs/domain-case-studies.md) — e-commerce, SaaS, HR, and banking

### 9. Decisions and review

- [Modeling workflow and decision trees](09-decision-guides/modeling-workflow-and-decision-trees.md)
- [Failure modes and architecture review](09-decision-guides/failure-modes-and-review.md)

### 10. Reference

- [Ten-minute cheat sheet](10-reference/cheat-sheet.md)
- [Glossary](10-reference/glossary.md)
- [Architect interview questions](10-reference/architect-interview-questions.md)

For traceability, see the [coverage checklist](COVERAGE.md) and [sources](SOURCES.md).

## How examples are structured

Each major pattern answers the same questions:

- **The problem:** what requirement or failure creates the need?
- **Mental model:** how should you remember it?
- **Grain:** what does one row represent?
- **Model:** which tables, keys, facts, and relationships are involved?
- **Use / avoid:** where does the pattern fit, and where does it fail?
- **Tradeoffs:** what changes in correctness, complexity, usability, or performance?
- **Modern implementation:** what matters in SQL, dbt-style pipelines, lakehouses, SAP Datasphere, or HANA?

SQL is portable pseudocode unless platform-specific behavior matters.

## Scope and source discipline

The primary source is *The Data Warehouse Toolkit*, third edition. The handbook paraphrases and synthesizes; it does not reproduce the book. Official material from dbt Labs, Databricks, Microsoft, SAP, AWS, and Data Vault practitioners is used selectively for current architecture and platform notes.

Technology changes. Grain, business meaning, historical correctness, and aggregation behavior remain architectural concerns even when storage and compute become cheaper.
