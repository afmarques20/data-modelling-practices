# Coverage and Quality Checklist

This is both a source map and a completion contract. A checked row means the concept has a canonical explanation, a grain-correct example, source verification, and a technical review — not merely that the term appears.

Page references use printed book pages from *The Data Warehouse Toolkit, Third Edition*. For the supplied PDF, add 36 to printed page numbers in the main text.

## Tier 1 — must master

| Done | Concept | Canonical chapter | Primary book anchors |
|---|---|---|---|
| [ ] | Grain: atomic, mixed, changing, aggregate, dimensional | [Grain](01-foundations/grain.md) | Ch. 2 pp. 39–40; Ch. 3 pp. 71–72; Ch. 11 pp. 300–301 |
| [ ] | Facts, dimensions, stars, snowflakes | [Dimensional modeling](01-foundations/dimensional-modeling.md) | Ch. 1 pp. 7–17; Ch. 2 pp. 40–50 |
| [ ] | Natural, durable, surrogate, and source keys | [Keys](01-foundations/keys.md) | Ch. 2 pp. 46–47; Ch. 3 pp. 98–102 |
| [ ] | Transaction, periodic, and accumulating facts | [Fact-table patterns](02-fact-tables/fact-table-patterns.md) | Ch. 2 pp. 43–45; Ch. 4 pp. 113–121 |
| [ ] | Additive, semi-additive, non-additive measures | [Measures](01-foundations/measures.md) | Ch. 1 pp. 10–12; Ch. 2 p. 42 |
| [ ] | SCD Types 1 and 2; recognition of Types 0, 3–7 | [SCDs](03-dimensions/slowly-changing-dimensions.md) | Ch. 2 pp. 53–56; Ch. 5 pp. 147–164 |
| [ ] | Conformed dimensions/facts and enterprise integration | [Conformance](05-enterprise-modeling/conformance-and-bus-architecture.md) | Ch. 2 pp. 50–53; Ch. 4 pp. 122–138 |
| [ ] | Bus architecture and bus matrix | [Conformance](05-enterprise-modeling/conformance-and-bus-architecture.md) | Ch. 4 pp. 123–129 |
| [ ] | Multivalued dimensions, bridges, weighting | [Bridges](04-relationships/bridges-and-many-to-many.md) | Ch. 2 p. 63; Ch. 8 pp. 245–248; Ch. 10 pp. 287–290 |
| [ ] | Fixed, ragged, parent-child, recursive hierarchies | [Hierarchies](04-relationships/hierarchies.md) | Ch. 2 pp. 56–58; Ch. 7 pp. 214–223; Ch. 9 pp. 271–273 |
| [ ] | Late facts and late dimensions | [Late-arriving data](06-time-and-change/late-arriving-data.md) | Ch. 2 pp. 62, 67; Ch. 19 pp. 477–479 |

## Tier 2 and Tier 3 patterns

| Done | Concept | Canonical chapter | Primary book anchors |
|---|---|---|---|
| [ ] | Factless, aggregate, consolidated facts | [Advanced fact designs](02-fact-tables/advanced-fact-designs.md) | Ch. 2 pp. 44–45; Chs. 3, 7, 13, 15–16 |
| [ ] | Header/line, allocation, currencies, units | [Advanced fact designs](02-fact-tables/advanced-fact-designs.md) | Ch. 2 pp. 59–61; Ch. 6 pp. 181–197 |
| [ ] | Lag, duration, timespan, changing facts, snapshot edges | [Advanced fact designs](02-fact-tables/advanced-fact-designs.md) | Ch. 2 pp. 59–62; Ch. 16 pp. 393–395 |
| [ ] | Role-playing, degenerate, junk, mini, outrigger dimensions | [Dimension patterns](03-dimensions/dimension-patterns.md) | Ch. 2 pp. 47–50, 55; Chs. 3, 5, 6, 10 |
| [ ] | Shrunken, hot-swappable, audit, measure-type dimensions | [Dimension patterns](03-dimensions/dimension-patterns.md) | Ch. 2 pp. 51, 65–67; Chs. 6, 10, 14, 16 |
| [ ] | Very large, rapid, heterogeneous, internationalized dimensions | [Dimension patterns](03-dimensions/dimension-patterns.md) | Chs. 8, 10, 12, 16 |
| [ ] | Date, time, fiscal, holiday, week, time-zone calendars | [Dates and calendars](03-dimensions/date-time-calendars.md) | Ch. 2 pp. 48–49; Chs. 3, 7, 12 |
| [ ] | Multiple processes, drill-across, fact-to-fact risk | [Multiple processes](05-enterprise-modeling/multi-process-and-heterogeneous-models.md) | Ch. 2 pp. 51, 61; Chs. 4, 8 |
| [ ] | Heterogeneous products/processes | [Multiple processes](05-enterprise-modeling/multi-process-and-heterogeneous-models.md) | Ch. 2 pp. 67–68; Chs. 10, 14, 16 |

## Modern synthesis and application

These topics are later synthesis; they must not be presented as terminology from the 2013 book.

| Done | Concept | Canonical chapter | Source basis |
|---|---|---|---|
| [ ] | Event, state, valid time, system time, bitemporality | [Event, state, and temporal](06-time-and-change/event-state-and-temporal-modeling.md) | Kimball fact/SCD patterns plus modern temporal vocabulary |
| [ ] | Medallion and dimensional Gold models | [Layered architectures](07-modern-architecture/layered-architectures.md) | Databricks documentation |
| [ ] | Hubs, links, satellites; Vault-to-mart coexistence | [Layered architectures](07-modern-architecture/layered-architectures.md) | Data Vault Alliance and AWS |
| [ ] | Semantic models and governed metrics | [Semantic layer](07-modern-architecture/semantic-layer-and-metrics.md) | dbt, Microsoft, SAP documentation |
| [ ] | CDC, idempotency, MERGE, dedupe, backfills, schema evolution | [Reliable implementation](07-modern-architecture/reliable-implementation.md) | Kimball Chs. 19–20 plus current platform practices |
| [ ] | Learning analytics | [Learning analytics](08-practical-designs/learning-analytics.md) | Applied synthesis |
| [ ] | E-commerce, SaaS, HR, and banking | [Domain cases](08-practical-designs/domain-case-studies.md) | Applied synthesis from case-study patterns |

## Important concepts added after the source review

- [ ] Four-step design process and business-process orientation
- [ ] Value chains and graceful model extension
- [ ] Special unknown, missing, not-applicable, and inferred members
- [ ] Drill-across instead of direct fact-to-fact joins
- [ ] Fact surrogate keys for ETL operability
- [ ] Behavior cohorts, step dimensions, and error/audit dimensions
- [ ] Data profiling, deduplication, restart/recovery, lineage, and versioning
- [ ] Timeless modeling rules versus technology-contingent physical optimizations

## Final review record

| Review | Pass criteria | Status |
|---|---|---|
| Coverage | Every requirement has one canonical home | Pending |
| Redundancy | Full explanations are not repeated across chapters | Pending |
| Accuracy | Grains, keys, joins, histories, and aggregation rules are consistent | Pending |
| Practicality | A practitioner can reason from requirement to model | Pending |
| Clarity | Concepts stand alone without prior reading of the book | Pending |
| Architecture depth | Major patterns explain consequences and tradeoffs | Pending |
| Modern relevance | Current mappings are useful and clearly labeled as synthesis | Pending |
| Conciseness | Secondary variants are compressed without hiding decision-critical nuance | Pending |

