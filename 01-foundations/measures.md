# Measures and Aggregation Behavior

A measure is useful only when its aggregation behavior is understood. Many wrong dashboards use correct rows and valid joins but apply the wrong arithmetic.

## The problem

SQL makes `SUM` and `AVG` easy. It does not know whether they make business sense.

- Revenue can usually be summed across products, customers, and dates.
- An account balance can be summed across accounts for one date, but not across daily snapshots.
- A conversion rate cannot be added across channels.
- Distinct customers by month cannot be summed to obtain distinct customers for a year.
- The average of store-level averages is wrong when stores have different numbers of orders.

The database can return a precise number for every one of these calculations. Measure design determines whether the number is meaningful.

## Mental model

Every measure needs a contract:

```text
measure meaning
+ fact grain
+ valid aggregation dimensions
+ time behavior
+ unit and currency
+ null / zero semantics
= trustworthy metric input
```

Ask:

> **What exactly was measured for one row, and along which dimensions may it be combined?**

## Grain

A measure must be observed, derived, or deliberately allocated at the table's declared grain.

For an e-commerce fact:

> **One row represents one product line on one placed order.**

`line_quantity` and `line_net_amount` fit. The whole order's shipping charge does not fit unless it is allocated to lines using a governed rule.

For a banking snapshot:

> **One row represents one account in one currency at the close of one business date.**

`closing_balance` fits. Its valid time aggregation is constrained by the snapshot semantics.

Before choosing `SUM`, `AVG`, or another function, verify the [grain](grain.md).

## The three aggregation classes

### Additive measures

An additive measure can be summed across every relevant dimension of the fact.

Typical examples:

- sales quantity;
- gross, discount, net, tax, and cost amounts in a consistent currency;
- learning duration captured per activity event;
- transaction amount;
- bytes processed;
- event count stored as `1`.

For an order-line fact, line net amount can normally be summed by product, customer, channel, or date.

**Important condition:** additivity assumes each row represents a distinct observation and all values use compatible units. A fully additive amount in euros is not directly additive with an amount in dollars.

### Semi-additive measures

A semi-additive measure can be summed across some dimensions but not all. Time is the most common exception.

Examples:

- bank account balance;
- inventory on hand;
- open support-case count;
- active subscription seats;
- month-end headcount;
- SaaS monthly recurring revenue captured as periodic state.

For an account-day snapshot:

- summing balances across accounts on 2026-09-30 is meaningful;
- summing one account's balance across every day in September is usually not.

Useful time behaviors include:

- last non-empty value in the period;
- period-end value;
- period-start value;
- minimum or maximum state;
- average daily balance;
- weighted average over time.

These are different metrics and need different names.

### Non-additive measures

A non-additive measure cannot be meaningfully summed across ordinary dimensions.

Examples:

- ratios and percentages;
- unit prices;
- averages;
- rates;
- medians and percentiles;
- distinct counts;
- index values.

These measures often can be recomputed from additive components or require a specialized aggregation.

## Comparison

| Measure | Example grain | Across non-time dimensions | Across time | Correct strategy |
|---|---|---|---|---|
| Net revenue | One order line | Sum | Sum | `SUM(net_amount)` |
| Learning seconds | One activity event | Sum | Sum | `SUM(duration_seconds)` |
| Closing balance | One account-day | Sum across accounts for one date | Usually not sum | Last, average, min, or max as defined |
| Inventory on hand | One product-location-day | Sum across products/locations for one date | Usually not sum | Select snapshot or average state |
| Conversion rate | One reporting slice | Do not sum | Do not sum | `SUM(conversions) / SUM(opportunities)` |
| Average order value | One reporting slice | Do not average averages | Do not average averages | `SUM(revenue) / COUNT(DISTINCT order)` |
| Distinct learners | One reporting slice | Recompute | Recompute | `COUNT(DISTINCT durable_learner_key)` |
| Median salary | One employee observation | Recompute | Recompute | Calculate percentile over target population |

“Non-additive” does not mean “useless in a fact table.” It means the semantic layer must use a valid aggregation rule.

## Store components, derive ratios

### The dangerous design

Suppose two SaaS channels report:

| Channel | Trials | Paid conversions | Conversion rate |
|---|---:|---:|---:|
| Partner | 10 | 5 | 50% |
| Organic | 1,000 | 100 | 10% |

Averaging 50% and 10% gives 30%. The combined conversion rate is:

```text
(5 + 100) / (10 + 1,000) = 10.40%
```

The correct model stores or makes available:

- `trial_count`;
- `paid_conversion_count`.

The rate is calculated after aggregation:

```sql
sum(paid_conversion_count)
/ nullif(sum(trial_count), 0)
```

### Averages

Store additive components whenever possible:

```text
average assessment score
= sum(score_points) / count(scored_attempts)

weighted average unit price
= sum(quantity * unit_price) / sum(quantity)

average handling time
= sum(handling_seconds) / sum(handled_case_count)
```

Do not take `AVG(average_score)` over groups unless every group has equal weight by definition.

### Percentages and shares

A stored row-level percentage can be legitimate if it describes that row, but it should not be given default `SUM` behavior. Keep numerator and denominator when consumers need aggregation at other levels.

For pass rate:

- `passed_attempt_count` = 1 or 0;
- `scored_attempt_count` = 1 for eligible attempts;
- pass rate = sums divided at query time.

The denominator definition — submitted attempts, first attempts, latest attempts, or unique learners — is a business rule and must be named.

## Distinct counts

Distinct counts are non-additive because the same entity can appear in several groups or periods.

If 80 learners were active in January and 90 in February, the two months do not imply 170 distinct learners. Some people may be present in both.

### Choose the identity being counted

- `COUNT(DISTINCT learner_key)` on a Type 2 dimension may count profile versions.
- `COUNT(DISTINCT durable_learner_key)` counts durable entities.
- `COUNT(DISTINCT enrollment_id)` counts enrollment lifecycles.
- `COUNT(DISTINCT account_id)` counts accounts, not customers.

The key must match the metric's entity definition.

### Strategies

- calculate exact distinct counts at the requested query grain;
- create a factless table at a lower reusable grain, such as one user-feature-day;
- use a platform's mergeable approximate sketch for very large interactive workloads;
- precompute distinct counts only for fixed dimensional combinations and prevent invalid summation.

An ordinary integer “monthly distinct users” is not additive into quarters. Approximation technology improves performance, not semantics.

## Flow versus stock

A useful distinction:

- **Flow** measures activity during an interval: sales, payments, hours learned, hires during month.
- **Stock** measures state at a point or boundary: balance, inventory, headcount, open tickets.

Flows are often additive across time. Stocks are usually semi-additive across time.

An account snapshot can include both:

| Column | Meaning | Time behavior |
|---|---|---|
| `closing_balance` | State at day end | Do not sum across days |
| `debit_amount_during_day` | Flow during day | Sum across days |
| `credit_amount_during_day` | Flow during day | Sum across days |
| `transaction_count_during_day` | Flow during day | Sum across days |

Name the time semantics. A generic `amount` column invites misuse.

## Worked examples

### Learning analytics

**Grain**

> One row represents one completed learning activity event by one learner.

Good atomic measures:

- `duration_seconds`;
- `completion_count = 1`;
- `score_points` and `possible_points` when the event is scorable.

Derived metrics:

```text
learning hours = SUM(duration_seconds) / 3600
completion rate = SUM(completion_count) / SUM(assigned_activity_count)
normalized score = SUM(score_points) / SUM(possible_points)
```

The assignment denominator may come from a separate coverage fact. Aggregate activity and coverage to matching conformed dimensions before combining them.

### E-commerce

**Grain**

> One row represents one product line on one placed order.

Store:

- quantity;
- gross amount;
- discount amount;
- net amount;
- tax amount;
- cost amount.

Derive:

```text
gross margin amount = SUM(net_amount) - SUM(cost_amount)
gross margin percent = gross margin amount / SUM(net_amount)
average order value = SUM(net_amount) / COUNT(DISTINCT order_number)
```

Keeping additive components makes pricing, discount, and margin logic auditable.

### SaaS subscription state

**Grain**

> One row represents one subscription at the close of one calendar month.

`licensed_seat_count` and `monthly_recurring_revenue` can be summed across subscriptions for the same month. Summing twelve monthly MRR values answers “subscription-month value,” not annual recurring revenue. ARR may be defined as period-end MRR multiplied by 12; it is a point-in-time run-rate metric, not the sum of monthly snapshots.

### HR headcount

**Grain**

> One row represents one employee assignment at month-end.

`headcount_count = 1` and `full_time_equivalent` are additive across departments at a fixed month-end if assignments and allocation rules prevent double counting. They are not additive across months. Hires and terminations during a month are flows and belong in an event fact or clearly named flow columns.

### Banking

**Grain**

> One row represents one account in one currency at the close of one business date.

For “total deposits at September 30,” sum balances across eligible deposit accounts on that date. For “average daily deposits in September,” first produce one total balance per day, then average those daily totals according to the governed calendar and missing-day rule.

## Zero, null, and missing rows

These states are different:

- **Zero:** the measure is known and equals zero.
- **Null:** the measure is unknown, not applicable, or not yet available; document which.
- **Missing row:** the business event or snapshot row is absent.

In an event fact, no row often means no event. In a dense periodic snapshot, a missing account-day may mean a failed load rather than zero balance. Do not use `COALESCE(value, 0)` until the business meaning supports it.

Null additive facts are ignored by `SUM` in most SQL engines, while all-null groups may return null. Metric definitions must specify intended behavior.

## Units, currencies, and signs

Additivity requires compatible measurement bases:

- seconds cannot be summed with minutes without conversion;
- euros cannot be summed with dollars without a governed exchange-rate basis;
- quantities in items and kilograms need explicit unit semantics;
- debit/credit or revenue/refund signs need a consistent convention.

Store an original amount and a standardized amount when both auditability and cross-entity aggregation matter. Keep the original currency/unit key and the conversion basis. See [Advanced fact designs](../02-fact-tables/advanced-fact-designs.md).

## Derived and stored measures

### Derive when

- the formula is stable and inexpensive;
- additive components are available;
- the value depends on the query period or filter context;
- storing it would create reconciliation risk.

Examples: margin percentage, year-to-date revenue, rolling average, conversion rate.

### Store when

- the measure was observed by the source and must be audited;
- a governed complex calculation must be frozen as of processing;
- recomputation would require unavailable historical inputs;
- performance benefit is material and reconciliation is implemented.

If storing a derived measure, document formula version, rounding, and refresh/restatement behavior. Prefer storing its components as well.

## Why this works

Classifying measures prevents an aggregation engine from inventing business meaning. Additive components preserve flexibility: they can be combined into ratios, rates, margins, and averages at any valid dimensional level.

The model also separates two responsibilities:

- the fact table records measurements at a known grain;
- the semantic layer exposes governed formulas and context-sensitive aggregation.

Neither layer can compensate for an invalid grain or fanout join.

## When to use explicit measures

- Users need a governed numerical observation, count, state, or component.
- The value is valid for every applicable row at the declared grain.
- Its unit, sign, null behavior, and valid aggregation are known.
- It can be reconciled to a trusted source or control total.

## When not to store a measure

- It is a report-specific percentage easily recomputed from additive components.
- It repeats a header-level value on lower-grain rows without allocation.
- It is a current entity attribute rather than an observation at fact grain.
- Its formula depends on arbitrary user filter context, such as year-to-date.
- Its source and business meaning cannot be distinguished from similarly named values.

## Common mistakes

| Mistake | Symptom | Corrective action |
|---|---|---|
| Summing a balance across dates | Inflated stock values | Use period-end, average, min, or max behavior |
| Averaging percentages | Small groups receive the same weight as large groups | Sum numerators and denominators, then divide |
| Summing distinct counts | Repeated entities are counted several times | Recompute at the requested dimensional grain |
| Repeating a header amount on lines | Amount multiplies by line count | Keep header grain or allocate explicitly |
| Mixing currencies or units | Plausible but meaningless total | Standardize with governed conversion and retain original basis |
| Treating null as zero | Data failure appears as valid absence | Preserve and classify missingness |
| Using `DISTINCT` after fanout | Some duplicates disappear, amounts may remain wrong | Fix grain and relationship cardinality |
| Naming every value `amount` or `count` | Time and business semantics disappear | Use precise names and metric metadata |
| Storing only a ratio | Correct aggregation becomes impossible | Store numerator and denominator components |

## Tradeoffs

Atomic components increase column count and may require semantic formulas, but they are reusable and auditable. Precomputed metrics can be faster and easier for one report, but they often become invalid at another grain or after a definition change.

Exact distinct counts and percentiles can be computationally expensive. Aggregates and sketches improve responsiveness but constrain dimensions or introduce approximation. Preserve an authoritative path to exact results for reconciliation where the business risk requires it.

## Modern implementation notes — later synthesis

The additive, semi-additive, and non-additive classification is Kimball-derived. The implementation mappings below are modern synthesis.

### Semantic layer

Define each governed metric with:

- source fact and grain;
- expression;
- required filters;
- entity key for distinct counts;
- valid dimensions;
- time grain and time aggregation;
- currency or unit;
- null and missing-row behavior.

For example:

```yaml
metric: monthly_completion_rate
fact: fact_learning_activity
numerator: sum(completion_count)
denominator: sum(eligible_activity_count)
time_grain: month
format: percentage
```

Syntax differs by product. The contract should not.

### SQL

Prefer safe division:

```sql
sum(numerator) / nullif(sum(denominator), 0)
```

Aggregate facts to their own grain before joining them. Use window functions for period-end or rolling behavior only after the base rows are correctly grouped.

### dbt-style tests

Test:

- nonnegative or accepted ranges where business-valid;
- component reconciliation, such as `gross - discount = net`;
- counts against known control totals;
- one snapshot row per entity-period;
- currency/unit completeness;
- denominator eligibility rules.

A test such as “not null” is useful but does not prove that a measure is additive.

### SAP HANA Cloud and SAP Datasphere

Set measure aggregation deliberately:

- `SUM` for additive components;
- exception aggregation such as last value by date for eligible snapshot metrics;
- calculated measures for ratios after aggregation;
- currency and unit conversion with governed reference data and dates.

Validate behavior across drill-down and roll-up levels. A formula that is correct at detail level may be wrong if the engine aggregates its result rather than its components.

### Approximate measures

Mergeable sketches for distinct counts or quantiles can support large interactive workloads. Label them as approximate, record error guarantees, and ensure aggregation uses compatible sketch states rather than summed displayed counts.

## Measure-design checklist

- [ ] The fact grain is written explicitly
- [ ] Each measure is valid at that grain
- [ ] Additive dimensions and exceptions are documented
- [ ] Stock and flow semantics are named
- [ ] Ratios retain numerator and denominator
- [ ] Average metrics retain sum and count or other required weights
- [ ] Distinct counts name the entity key
- [ ] Currency, unit, sign, precision, and rounding are governed
- [ ] Zero, null, and missing-row meanings are distinct
- [ ] Stored derivations have formula version and reconciliation rules
- [ ] Semantic aggregation is tested at several drill levels

## Related patterns

- [Grain](grain.md)
- [Dimensional modeling](dimensional-modeling.md)
- [Keys](keys.md)
- [Fact table patterns](../02-fact-tables/fact-table-patterns.md)
- [Advanced fact designs](../02-fact-tables/advanced-fact-designs.md)
- [Conformance and bus architecture](../05-enterprise-modeling/conformance-and-bus-architecture.md)
- [Semantic layer and metrics](../07-modern-architecture/semantic-layer-and-metrics.md)
- [Failure modes and review](../09-decision-guides/failure-modes-and-review.md)

## What I should remember

1. A measure is not trustworthy until its grain, unit, and aggregation behavior are explicit.
2. Additive measures sum across all relevant dimensions; semi-additive measures have exceptions, usually time.
3. Ratios, averages, percentages, and distinct counts must be recomputed at the requested query grain.
4. Store additive components such as numerator and denominator instead of only a precomputed ratio.
5. Stock measures describe state at a point; flow measures describe activity during an interval.
6. Zero, null, and a missing row are different business states.
7. A semantic layer can govern aggregation, but it cannot repair mixed grain or fanout.

## Source basis

This chapter synthesizes Kimball and Ross, *The Data Warehouse Toolkit*, 3rd ed., especially Chapters 2, 4, 7, and 10, with recurring measure examples across the case studies. See the [coverage checklist](../COVERAGE.md) and [sources](../SOURCES.md).
