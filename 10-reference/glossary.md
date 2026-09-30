# Glossary

Short definitions for recognition. Follow the chapter links for decision guidance and tradeoffs.

## A

**Accumulating snapshot**
A mutable fact with one row per lifecycle instance and columns for major milestones. [Fact-table patterns](../02-fact-tables/fact-table-patterns.md)

**Additive fact**
A measure that can be summed across every dimension applicable to the fact.

**Aggregate fact table**
A higher-grain performance structure derived from atomic facts. It is an optimization, not a replacement for atomic detail.

**Allocation**
A governed rule that distributes a higher-grain amount to lower-grain rows, ideally retaining the original amount and method.

**Atomic grain**
The lowest level of detail captured by a business process and useful to analysis; not necessarily the smallest detail imaginable.

**Audit dimension**
A dimension carrying reusable load, lineage, code-version, source, or data-quality context.

## B

**Bitemporal model**
A model that preserves both valid time (when something was true) and system time (when the platform knew it).

**Bridge table**
A controlled structure between a fact/dimension group and multiple dimension members, often with weights or effective dates. [Bridges](../04-relationships/bridges-and-many-to-many.md)

**Bronze / Silver / Gold**
Medallion-layer names for source fidelity, cleaned/integrated data, and business-facing data. Not fact-table types.

**Bus architecture**
An incremental enterprise approach in which process-specific dimensional models integrate through conformed dimensions and facts.

**Bus matrix**
A planning grid with business processes as rows and conformed dimensions as columns.

**Business key**
A domain identifier for an entity. In practice, distinguish the source natural key from a warehouse-controlled durable identity.

**Business process**
A measurable activity or state in a value chain, such as order lines, payments, attendance, or daily balances.

## C

**Candidate key**
A column set expected to identify one row at the declared grain.

**CDC — change data capture**
A mechanism for extracting source inserts, updates, and deletes. CDC changes still require business interpretation.

**Centipede fact table**
An anti-pattern with excessive tiny dimensions/foreign keys, often created by over-separating flags and codes.

**Conformed dimension**
A shared dimension whose keys, values, attribute meanings, and history are compatible across business processes.

**Conformed fact**
A measure with the same technical definition, units, and time basis wherever it is reused.

**Consolidated fact table**
A fact combining processes that can genuinely be expressed at exactly the same grain.

**Coverage factless fact**
A table of eligible, possible, or required dimensional combinations, used with activity to find what did not happen.

**Current flag**
An SCD Type 2 indicator identifying the current version; there should normally be one per durable entity.

## D

**Data Vault**
An integration/history modeling approach using hubs, links, and satellites, often feeding dimensional information marts.

**Degenerate dimension**
A useful business identifier stored directly in a fact because no remaining descriptive attributes justify a dimension row.

**Dimension**
Descriptive context used to filter, group, label, and navigate facts.

**Dimension grain**
What one dimension row represents, such as one entity, one historical entity version, or one date.

**Drill-across**
Aggregate separate facts to identical conformed headers, then align their results; the safe alternative to direct fact joins.

**Durable key**
A warehouse-controlled identity that remains stable for one business entity across source and historical changes.

## E

**Effective dating**
Recording when a version or relationship is valid, commonly with half-open `[valid_from, valid_to)` intervals.

**Event fact**
A transaction fact in which one row records one business occurrence.

**Event time**
When an event occurred in the business, distinct from ingestion or processing time.

## F

**Fact**
A numeric measurement or occurrence recorded at a declared business-process grain.

**Factless fact**
A fact table with no natural numeric measure; row existence or row count represents the fact.

**Fact surrogate key**
An optional ETL row identifier used for maintenance/restart, not a substitute for business grain.

**Foreign key**
A fact column referencing a dimension row, usually a warehouse surrogate key.

## G

**Gold layer**
The medallion layer for business-facing, consumption-optimized data; dimensional marts commonly live here.

**Grain**
The exact semantic meaning of one row. The central question is “One row represents what?” [Grain](../01-foundations/grain.md)

**Graceful extension**
Adding grain-compatible facts, dimensions, or attributes without changing the meaning of existing rows and queries.

## H

**Hierarchy bridge**
A table representing ancestor/descendant paths in a ragged or shared hierarchy, often with depth and effective dates.

**Hot-swappable dimension**
Alternative, structurally compatible dimension interpretations that can analyze the same facts.

**Hub / Link / Satellite**
Data Vault structures for business identity, relationships, and descriptive/history context respectively.

## I

**Idempotency**
The property that replaying the same accepted input produces the same target state.

**Impact report**
A many-to-many analysis that associates a full fact with every member; useful for reach/impact but intentionally non-additive across members.

**Inferred member**
A unique placeholder dimension row created when an entity's key is known before its attributes arrive.

## J

**Junk dimension**
A dimension containing observed combinations of low-cardinality flags, indicators, and statuses.

## L

**Late-arriving dimension**
A dimension entity/version that becomes available after related facts or after its business-valid period.

**Late-arriving fact**
An event received after the business period in which it occurred has been processed.

**Lag fact**
A duration between defined lifecycle milestones, ideally using a governed calendar and a limited set of anchor calculations.

## M

**Many-to-many relationship**
A relationship in which one fact can legitimately relate to multiple dimension members, requiring lower grain or a bridge.

**Measure**
A numeric field or expression intended for aggregation according to an explicit rule.

**Measure-type dimension**
A dimension identifying which measurement a sparse generic value row contains; useful only in extreme cases because it resembles EAV and complicates calculations.

**Medallion architecture**
A progressive refinement pattern, commonly Bronze -> Silver -> Gold. It does not replace dimensional design.

**Metric**
A governed analytical calculation including components, filters, entity identity, time behavior, units, and owner.

**Mini-dimension**
A separate dimension for a correlated group of rapidly changing or frequently analyzed attributes.

**Mixed grain**
A table whose rows do not share one coherent semantic meaning and level of detail.

## N

**Natural key**
An identifier controlled by a source or business process. It can change, collide, or be reused and should not automatically become a warehouse relationship key.

**Non-additive fact**
A measure such as a ratio, average, or unit price that must be recomputed rather than summed.

## O

**Outrigger dimension**
A dimension referenced by another dimension. Use sparingly because it adds snowflake joins and can complicate Type 2 history.

## P

**Periodic snapshot**
A fact with one row per entity or dimensional combination per standard period.

**Processing time**
When a data platform handled a record, distinct from when the business event occurred.

## R

**Ragged hierarchy**
A hierarchy whose paths have different depths, common in organizations and charts of accounts.

**Role-playing dimension**
One physical dimension exposed in multiple logical roles, such as order date, ship date, and invoice date.

## S

**SCD — slowly changing dimension**
A family of strategies for handling changing dimension attributes. Type 1 overwrites; Type 2 versions; other types preserve fixed/limited/hybrid perspectives.

**Semantic layer**
A governed model of entities, relationships, dimensions, measures, metrics, labels, and access rules over physical data.

**Semi-additive fact**
A fact additive across some dimensions but not all, such as a balance across accounts but not dates.

**Shrunken dimension**
A conformed subset of a base dimension's rows or columns, often matched to aggregate grain.

**Snowflake schema**
A dimensional schema whose dimensions are normalized into additional tables. It can be valid but is often less usable than a flattened star.

**Star schema**
A central fact table joined directly to descriptive dimensions.

**Step dimension**
A dimension describing position within a sequence or session, including current and possible next steps.

**Surrogate key**
A warehouse-controlled dimension-row identifier independent of source keys; in Type 2, it identifies one historical version.

## T

**Transaction fact**
A fact with one row per event at a point in time, normally insert-oriented and sparse.

**Timespan fact**
A fact/state row valid for a continuous interval, with effective and expiration timestamps.

**Type 1**
Overwrite a dimension attribute; all joined history sees the new/corrected value.

**Type 2**
Insert a new dimension row with a new surrogate key and effective interval to preserve history.

## U

**Unknown member**
A shared special dimension row used when identity is genuinely absent or unresolved; different from a unique inferred member.

## V

**Valid time**
The interval during which a statement was true in the business domain.

**Value chain**
A sequence of related business processes, each usually modeled as its own fact and integrated through conformed dimensions.

## W

**Weighting factor**
A bridge allocation value; factors for one bridge group typically total 1 when reports must preserve additive totals.

## Y

**Year-to-date fact**
A period-relative derived value often better calculated from atomic components in the semantic layer than stored as many redundant variants.
