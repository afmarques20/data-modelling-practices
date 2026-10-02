# Semantic Layers and Governed Metrics

Suppose two dashboards show a different number of “active learners.” One counts anyone who logged in. The other counts learners who completed at least one activity. Both queries run correctly, but the phrase has two meanings.

A **semantic layer** prevents that kind of disagreement by giving business terms, relationships, and calculations a shared definition. A dimensional warehouse organizes trustworthy rows; the semantic layer explains how people and tools should use those rows safely.

```mermaid
flowchart LR
    P[Physical data model<br/>facts, dimensions, history] --> S[Semantic model<br/>relationships and aggregation behavior]
    S --> G[Governed metrics<br/>definitions and policies]
    G --> C[Dashboards, notebooks,<br/>applications, AI]
```

Both layers are necessary. A semantic layer cannot repair a fact table with mixed grain. A perfect star schema, on its own, cannot stop five dashboards from defining “active learner” five different ways.

## The problem

Without shared definitions, every report has to decide for itself:

- joins and relationship direction;
- current versus historical dimension logic;
- currency conversion;
- distinct-count entity identity;
- ratios and weighted averages;
- fiscal calendars and time comparisons;
- filters such as test accounts, canceled orders, or internal employees.

Two queries can both be valid SQL and still answer different questions. A governed metric makes the chosen meaning explicit and reusable.

## Mental model

> The dimensional model defines what the data means at row level. The semantic layer defines how people may safely ask questions of it.

## Three levels of meaning

The terms are easier to separate with one example:

| Level | Question it answers | Learning example |
|---|---|---|
| Physical fact | What does one stored row mean? | One submitted assessment attempt |
| Semantic model | How may tables be joined and fields grouped? | Attempts join to the learner and course valid for that attempt |
| Metric | How is a business result calculated? | Passed attempts divided by graded attempts |

The underlying fact still has its declared grain:

> One row in `fact_assessment_attempt` represents one submitted learner attempt for one assessment.

A semantic metric also states the groupings for which its calculation is valid. For example:

```yaml
metric: assessment_pass_rate
label: Assessment pass rate
type: ratio
numerator: sum(passed_count)
denominator: sum(graded_attempt_count)
time_dimension: submitted_date
valid_dimensions:
  - course
  - assessment
  - learner_department_as_of_attempt
filters:
  - is_practice_attempt = false
format: percentage
```

Store countable components such as `passed_count` and `graded_attempt_count` in the fact. Add them for the requested group, then divide. Averaging percentages from individual rows or subgroups is usually wrong.

For example, one course with 9 passes out of 10 attempts and another with 1 pass out of 2 attempts have a combined pass rate of `10 / 12 = 83.3%`. Averaging `90%` and `50%` would incorrectly produce `70%`.

## What a semantic definition should specify

The following checklist is detailed. Beginners can start with the metric's name, source fact, calculation, time field, allowed filters, and owner, then add the remaining rules as the model grows.

### Entities and keys

- Which key identifies a business entity across source systems?
- Does a surrogate key identify a Type 2 version?
- Which relationships are one-to-many, one-to-one, or many-to-many?
- Which join paths are safe and which require a bridge or pre-aggregation?

### Dimensions

- User-facing labels and descriptions
- Hierarchies and sort order
- Role-playing dates such as order, ship, and invoice date
- Current versus event-time historical attributes
- Fiscal/calendar semantics and time zones
- Unknown and not-applicable member presentation

### Measures

- Base expression and data type
- Additive dimensions and forbidden aggregations
- Currency or unit basis
- Null and zero behavior
- Whether distinct count uses a durable entity key or version key

### Metrics

- Business name and purpose
- Numerator, denominator, and filters
- Time window and comparison behavior
- Grain and valid slice dimensions
- Inclusion/exclusion policy
- Owner, version, tests, and change process

### Security

- Row-level restrictions
- Sensitive attributes and masking
- Aggregation thresholds
- Entitlement behavior through bridges or organizational hierarchies

## Example: active learners

“Monthly active learner” can mean at least four things:

1. a user who logged in;
2. a user who generated any learning event;
3. a user with at least one minute of learning activity;
4. a user enrolled in an active course.

A governed definition might be:

> Distinct durable users with at least one non-test learning activity event during the calendar month, attributed to the user's department valid at event time.

That definition settles:

- fact source: `fact_learning_activity`;
- entity key: `user_durable_key`, not Type 2 `user_sk`;
- date role: activity event date;
- exclusions: test activity;
- historical attribution: event-time department;
- time window: calendar month in the reporting time zone.

The SQL now implements an agreed definition instead of quietly inventing one.

## Conformed dimensions and semantic consistency

Dimensional models integrate processes through shared, consistently defined dimensions and facts. These are called **conformed dimensions and facts**. A semantic layer carries that agreement into business-facing names and calculations:

- `Customer` means the same governed entity across sales, support, and payments;
- `Net Revenue` uses the same components and currency policy across tools;
- `Order Date` and `Ship Date` are explicit roles, not a generic ambiguous date;
- metric joins respect fact grain rather than automatically joining raw fact tables.

Matching column names are not enough. Two `customer_id` columns are not conformed if they identify different populations, use different keys, or handle history differently.

## Safe cross-process metrics

Suppose a dashboard needs orders, shipments, and payments by customer-month. Do not build an uncontrolled fact-to-fact join. A semantic planner or mart should:

1. aggregate each fact independently to conformed `customer` and `month` headers;
2. align the aggregated results;
3. calculate cross-process metrics from those aligned values.

This pattern is called **drill-across**: summarize separate processes to the same shared headings, then combine the summaries. See [Multiple processes](../05-enterprise-modeling/multi-process-and-heterogeneous-models.md).

## Optional: platform mappings

The products below use different vocabulary, but the same questions remain: What does one row mean? Which joins are safe? How should each measure aggregate? Where is a metric defined once?

### dbt

[dbt semantic models](https://docs.getdbt.com/docs/build/semantic-models) describe entities, dimensions, and measures connected in a semantic graph. The [dbt Semantic Layer](https://docs.getdbt.com/docs/use-dbt-semantic-layer/dbt-sl) centralizes metric definitions and join logic for downstream tools.

Do not map dbt vocabulary mechanically onto physical Kimball tables. A dbt entity can describe a join role, while a dbt mart may be a wide entity-grained table rather than a star. The transferable rules are clear grain, unique entity keys, controlled joins, and governed aggregation.

### Power BI and Microsoft Fabric

Microsoft's [Power BI star-schema guidance](https://learn.microsoft.com/en-us/power-bi/guidance/star-schema) maps dimensions to filtering/grouping and facts to summarization, with explicit support for surrogate keys, role-playing dimensions, SCDs, factless facts, and consistent fact grain.

In a Power BI semantic model:

- prefer one-to-many relationships from dimensions to facts;
- keep measures centralized rather than recreating them in reports;
- mark role-playing dates with clear table/relationship names;
- prevent default summation of semi-additive balances;
- create upstream Type 2 history rather than expecting the BI layer to reconstruct it.

### SAP Datasphere

SAP Datasphere's [semantic modeling](https://help.sap.com/docs/SAP_DATASPHERE/c8a54ee704e94e15926551293243fd1d/5c1e3d4a49554fcd8fcf199d664d1109.html) distinguishes fact entities with measures from dimension entities with descriptive attributes and supports associations, hierarchies, and analytic models.

Useful mapping:

| Dimensional idea | Datasphere expression |
|---|---|
| Fact process and grain | Fact entity plus documented key/semantic usage |
| Dimension | Dimension entity associated to facts |
| Measure | Measure with explicit aggregation behavior |
| Hierarchy | Hierarchy on dimension semantics |
| Type 2-like validity | Time-dependent dimension/association where appropriate |
| Consumer model | Analytic model exposing selected measures and dimensions |

This may be logical rather than a physically materialized star. The architect still owns grain, cardinality, history, and aggregation correctness.

### SAP HANA calculation views

HANA calculation views can use a [star join](https://help.sap.com/docs/hana-cloud-database/sap-hana-cloud-sap-hana-database-modeling-guide-for-sap-web-ide-full-stack/create-calculation-views-with-star-joins) to associate shared dimension views with a fact/cube source. Model cardinality accurately, separate measures from attributes, and verify exception aggregation for balances and ratios. Engine optimization is not a substitute for a valid dimensional relationship.

### Lakehouse engines

Facts and dimensions may be Delta/Iceberg tables, materialized views, or virtual models. A semantic layer can calculate metrics over them, but the contracts should be version-controlled and tested along with transformations.

## Metric design patterns

### Additive metric

```text
gross_revenue = SUM(gross_revenue_amount_base_currency)
```

Safe across dimensions at or above the fact grain, assuming currency and duplicate rules are governed.

### Ratio metric

```text
completion_rate = SUM(completion_count) / NULLIF(SUM(eligible_count), 0)
```

Aggregate components first. Define eligibility explicitly.

### Semi-additive metric

```text
ending_balance = balance from the latest valid snapshot in the selected period
```

Summing across accounts on one date can be valid; summing daily balances across dates is not. The metric needs a time-selection rule such as last non-empty, period end, or average daily balance.

### Distinct entity metric

```text
active_customers = COUNT(DISTINCT customer_durable_key)
```

Using a Type 2 surrogate key counts versions. Using a source key may collide across systems. The governed entity key is part of the metric definition.

### Non-additive percentile or median

These must be recomputed from sufficient detail or a mergeable sketch appropriate to the required accuracy. Do not average subgroup medians.

## Change management

A metric is a shared interface used by people, dashboards, and software. A quiet definition change can alter many reports at once, so treat changes deliberately:

1. name an owner and approver;
2. store the definition in version control;
3. test base facts, joins, and expected examples;
4. assess downstream impact;
5. version or announce breaking semantic changes;
6. retain an explanation of why the definition changed;
7. compare new and old outputs before release.

The objective is not to freeze definitions forever. It is to make evolution governed and reproducible.

## Optional: AI and natural-language analytics

Generative interfaces amplify semantic-model quality. An AI assistant can translate a request into a query, but it still needs:

- a precise catalog of entities and metrics;
- safe relationship paths;
- synonyms and descriptions;
- temporal and currency semantics;
- access controls;
- examples of valid questions and expected results.

Without those contracts, natural-language convenience increases the speed at which ambiguous metrics spread.

## Common mistakes

- Defining the same KPI separately in each dashboard.
- Treating a measure name as a complete business definition.
- Averaging stored averages or percentages.
- Letting tools infer many-to-many joins without allocation rules.
- Counting Type 2 versions when the business asks for entities.
- Exposing both current and historical attributes without naming the perspective.
- Building semantic models directly on raw source tables with unstable keys.
- Assuming a semantic layer removes the need for data-quality and grain tests.
- Hiding metric changes rather than versioning or communicating them.

## Tradeoffs

Central semantics reduce duplication and improve consistency, but create a governed dependency that needs ownership, deployment discipline, and performance engineering. Tool-specific models can use native capabilities deeply; tool-neutral contracts improve reuse but may support a lowest common denominator. Many organizations use a central metric specification plus thin platform adaptations.

## Related patterns

- [Measures and additivity](../01-foundations/measures.md)
- [Conformance and bus architecture](../05-enterprise-modeling/conformance-and-bus-architecture.md)
- [Layered architectures](layered-architectures.md)
- [Learning analytics design](../08-practical-designs/learning-analytics.md)

## Beginner review checklist

- [ ] Does every important metric have one plain-language definition?
- [ ] Are its source fact, time field, filters, and entity key named?
- [ ] Are ratios calculated from summed components rather than averaged percentages?
- [ ] Are balances and other snapshots protected from being summed through time?
- [ ] Are facts summarized separately before cross-process results are combined?
- [ ] Can users tell whether an attribute means “at event time” or “current”?
- [ ] Is one owner responsible for approving definition changes?

## What to remember

1. The physical model governs row meaning; the semantic layer governs safe analytical use.
2. A metric definition includes grain, entity identity, time, filters, aggregation, currency/unit policy, and owner.
3. Centralize reusable definitions so dashboards do not implement competing KPIs.
4. Aggregate facts separately before cross-process analysis.
5. Platform semantics may be logical or physical; grain and cardinality rules still apply.
6. AI analytics makes semantic governance more important, not less.
