# Conformance and the Enterprise Bus Architecture

Independent stars are easy to build and hard to reconcile. If sales, subscriptions, learning, support, and finance each define customer, product, date, and revenue differently, the platform contains several fast reporting silos rather than an integrated analytical system.

The Kimball bus architecture provides an incremental alternative: deliver one business process at a time, but make those processes fit together through governed conformed dimensions and facts.

## The problem

Teams often begin with local requirements:

- Sales needs an account and product model.
- Learning needs user, course, and completion data.
- Finance needs legal entity, cost center, and fiscal period.
- Product analytics needs account, user, feature, and event.

Without an enterprise contract, each team invents keys, labels, hierarchies, and metric equations. Cross-process reports then depend on manual mappings and arguments over whose number is correct.

## Mental model

**The bus matrix is the blueprint. Conformed dimensions are the shared connectors. Fact tables are replaceable process modules.**

Like a hardware bus, a new process can connect without redesigning every existing process, provided it honors the common contracts.

## Grain

Conformance never removes the need to declare grain:

- Each fact row has one process-specific measurement grain.
- Each conformed dimension has one atomic member grain or an explicitly shrunken rollup grain.
- Each bus-matrix row represents one business process, not one department or report.
- Each bus-matrix column represents one reusable business dimension at its governed base meaning.

## Conformed dimensions

### Definition

Dimensions conform when their shared attributes have compatible keys, names, definitions, domain values, and behavior, so those attributes can align results from different fact tables.

The strongest form is one physical dimension reused by several stars. Physical sharing is not required: synchronized copies across platforms can conform if their contract is identical.

### Example: conformed user

    dim_user
    ├── user_key                  surrogate version key
    ├── durable_user_key          stable enterprise identity
    ├── source_user_id
    ├── user_type
    ├── country_code
    ├── organization_unit
    ├── employment_status
    └── SCD metadata ...

The same governed user meaning can be referenced by:

- `fact_learning_event` at one row per user learning event;
- `fact_feature_usage` at one row per user feature event;
- `fact_user_month` at one row per user per month;
- `fact_support_case` at one row per case.

The fact grains differ. The user identity and shared attributes do not.

### What must conform

- member identity and key mapping;
- shared attribute names and business definitions;
- data types and permitted domain values;
- hierarchy and SCD rules for shared attributes;
- unknown, not-applicable, and error members;
- security classification and stewardship;
- refresh expectations where synchronized copies exist.

Two columns named `customer_segment` are not conformed if one uses current CRM segment and another uses event-time marketing segment. Conversely, dimensions can conform even if one contains extra process-specific attributes; only the common attributes can be used safely for cross-process alignment.

## Conformed facts

Facts conform when the same label means the same equation, dimensional context, unit, sign convention, and timing rule everywhere.

For example, `net_revenue` is not conformed unless teams agree on:

- inclusion or exclusion of tax, discounts, credits, and refunds;
- transaction, recognition, or settlement date;
- transaction versus reporting currency;
- gross versus net sign handling;
- treatment of cancelled or test transactions.

If two measures differ, name them differently. A misleading common label is worse than visible disagreement.

Facts from different processes do not need the same physical column or row grain to be conformed. They need an explicit common definition at the grain where comparison or addition is valid.

## Shrunken conformed dimensions

Different processes can operate at different grains while sharing rollups.

Example:

- actual learning events: user-course-day;
- workforce plan: organization-course-category-month.

The plan cannot use the atomic user or day dimension. It can use governed shrunken dimensions:

> One month row represents one reporting month, containing a strict subset of the daily date dimension's rollup attributes.

> One course-category row represents one governed course category, containing shared category attributes from the atomic course dimension.

The shrunken dimension's common attributes must have identical definitions and values. Its key is separate because its grain is different.

Two forms exist:

- **Attribute/rollup subset:** brand from product, month from date.
- **Row subset:** one business line's rows from a corporate dimension at the same grain.

For a row subset, the associated fact must be limited to the same population. Otherwise dimension foreign keys will not resolve and reports will silently omit data.

## The bus architecture

### Why this works

The enterprise is decomposed by observable business process: order placed, shipment delivered, learning completed, subscription invoiced, support case resolved. Each process can be delivered incrementally, usually as one or more atomic fact tables, while reusing dimensions already governed for the enterprise.

This balances two needs:

- **Local delivery:** a team can ship a valuable process slice without waiting for an enterprise-wide “perfect model.”
- **Global integration:** future slices can align because common dimensions and facts are deliberate contracts.

The architecture is logical and technology-independent. Facts may live in different schemas or platforms; conformance is what makes integration possible.

## The bus matrix

Rows are business processes. Columns are conformed dimensions. An X means the process uses that dimension; a qualified marker such as `M` can indicate a shrunken month role.

### Example: learning and SaaS business

| Business process | Date | User | Account | Course | Learning object | Session | Product | Subscription | Employee | Geography |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Learning object activity | X | X | X | X | X |  |  |  |  | X |
| Course enrollment | X | X | X | X |  |  |  |  |  | X |
| Course completion | X | X | X | X |  |  |  |  |  | X |
| Live session attendance | X | X | X | X |  | X |  |  | X | X |
| Assessment attempt | X | X | X | X | X |  |  |  |  | X |
| Product feature usage | X | X | X |  |  |  | X |  |  | X |
| Subscription lifecycle | X |  | X |  |  |  | X | X |  | X |
| Subscription month snapshot | M |  | X |  |  |  | X | X |  | X |
| Payment transaction | X |  | X |  |  |  | X | X |  | X |
| Support case | X | X | X |  |  |  | X |  | X | X |

The matrix exposes useful architecture questions:

- Does `User` mean the same learner and application user across both domains?
- Does `Account` include internal tenants, prospects, and paying customers?
- Which date roles exist in each process?
- Can Course and Product share a broader offering hierarchy, or are they genuinely separate dimensions?
- At which common grain can learning, usage, subscription, and payment metrics be compared?

The correct answer may be limited conformance, not forced universality.

### From enterprise matrix to implementation detail

The enterprise matrix stays readable. A detailed implementation matrix can expand one process row into exact fact tables, fact types, grain statements, measures, date roles, and source systems.

| Fact table | Type | Grain | Main measures |
|---|---|---|---|
| `fact_learning_event` | Transaction | One user interaction with one learning object at one event timestamp | seconds spent, completion increment |
| `fact_course_enrollment` | Accumulating snapshot | One user enrollment in one course offering | days to start, days to complete |
| `fact_user_learning_month` | Periodic snapshot | One user-course per reporting month | month-end progress, cumulative hours |

Do not overload the executive bus matrix with every fact and attribute. Maintain both views for different audiences.

## How to build a bus matrix

1. Identify the organization's value chain and observable business processes.
2. Name process rows with verbs and business objects, not department names.
3. Declare a candidate grain for each process.
4. List reusable dimensions at their most atomic governed meaning.
5. Mark process-dimension intersections.
6. Annotate roles or shrunken grains where necessary.
7. Scan each row: are all descriptive contexts for that process present?
8. Scan each column: where must the dimension be shared and governed?
9. Identify conformed facts and unit conversions needed for cross-process analysis.
10. Prioritize process rows by value, feasibility, reuse, and source readiness.

## Data governance and ownership

Conformance is an organizational agreement expressed in data, not a naming convention imposed by architects.

For each shared dimension, establish:

- an accountable business owner;
- definitions for member identity and merge/split rules;
- attribute stewards and SCD behavior;
- accepted source precedence;
- domain values and hierarchies;
- data-quality thresholds;
- schema and semantic versioning rules;
- consumer notification and migration policy.

IT can facilitate and implement. It cannot unilaterally resolve whether two customer definitions are equivalent or which revenue equation the enterprise should use.

Start with a valuable common subset if full agreement is impossible. One governed product category or enterprise account identifier can enable real integration. Limited honest conformance is better than broad fictional conformance.

## Using the matrix as an architecture tool

The bus matrix supports more than schema design:

- **Roadmap:** each process row is a manageable delivery candidate.
- **Reuse plan:** heavily used columns deserve early governance and robust pipelines.
- **Impact analysis:** a dimension contract change reveals affected processes.
- **Data quality:** shared dimensions expose conflicting source identities and labels.
- **Security:** sensitive dimensions show where access policies propagate.
- **Team coordination:** independent teams can work asynchronously against stable contracts.
- **Semantic governance:** common facts and attributes become reusable metric inputs.

## When to use it

- Several business processes must support cross-process analysis.
- Teams need incremental delivery without creating isolated marts.
- Shared entities such as customer, product, employee, user, date, or geography recur.
- A platform spans several warehouses, lakehouses, semantic models, or SAP analytical spaces.

## When not to force it

Do not force a single enterprise dimension when populations and meanings truly do not overlap and there is no cross-domain analytical need. A conglomerate may have unrelated product or customer worlds. Preserve separate domains and conform only a small corporate rollup if that is the real requirement.

Do not wait for every attribute across the enterprise to be standardized before delivering the first process. Govern the smallest useful common contract, then extend it gracefully.

## Common mistakes

- Using departments such as Finance or Marketing as matrix rows instead of business processes.
- Using individual reports as rows.
- Creating a generic `Person` or `Location` column that hides incompatible populations and attributes.
- Making each hierarchy level a separate matrix column.
- Marking two dimensions conformed because their column names match.
- Sharing natural keys without reconciling source collisions or key reuse.
- Allowing local teams to extend shared attributes with conflicting semantics.
- Forcing a fact table to combine several processes just because they share dimensions.
- Treating the bus matrix as a one-time presentation rather than a living architecture artifact.
- Defining dimensions centrally but allowing every dashboard to redefine the facts.

## Tradeoffs

| Benefit | Cost |
|---|---|
| Cross-process consistency | Governance and negotiation effort |
| Faster later delivery through reuse | More discipline in early projects |
| Independent, incremental process teams | Contract versioning and coordination |
| Simpler drill-across analysis | Shared identity resolution and data quality work |
| Source-system independence | Mapping and stewardship pipelines |

Conformance can feel slower at the beginning because disagreements become visible. That visibility is architectural value: the disagreement existed already and would otherwise surface in production reports.

## Modern implementation notes

- **Data contracts:** publish dimension grain, keys, shared attributes, SCD behavior, freshness, and quality tests as a versioned contract.
- **dbt-style projects:** build shared dimensions once or from one governed package; use relationship and accepted-value tests in every consuming mart.
- **Lakehouses:** a conformed Gold layer can expose shared dimensions over cleaned Silver entities. Medallion layer names alone do not create conformance.
- **Semantic layers:** centrally govern conformed facts as metrics, including date role, unit, filters, and valid dimensionality. A semantic metric cannot repair incompatible physical grains.
- **SAP Datasphere:** shared dimensions and analytical datasets can be reused across spaces through governed models. Ensure associations, hierarchy definitions, and analytical semantics remain consistent when replicated or exposed remotely.
- **SAP HANA:** calculation views can implement shared dimensions and star joins, but identical view names are not enough; key mapping, cardinality, and attribute semantics must conform.
- **Master data management:** MDM can improve source identity and reference data, but a dimensional conformed dimension still needs analytical SCD and unknown-member rules.

## Architectural consequences

- **Scalability:** atomic process facts can scale independently while dimensions provide common access paths.
- **Maintainability:** one governed definition reduces repeated transformation logic.
- **Historical correctness:** shared SCD rules prevent one process from reporting as-was while another silently reports as-is.
- **Interoperability:** common keys and attributes make data products composable across tools.
- **Governance:** ownership moves from dashboard-level reconciliation to explicit enterprise contracts.
- **Query simplicity:** consumers align separately aggregated results on shared attributes rather than engineering bespoke source-to-source joins.

## What to remember

1. Bus-matrix rows are business processes; columns are conformed dimensions.
2. Conformed dimensions share keys or compatible attributes, definitions, domain values, and history rules.
3. Conformed facts share equations, dimensional context, timing, units, and names.
4. Shrunken dimensions preserve conformance when a process operates at a higher or subset grain.
5. Build one process at a time, but make every process honor shared contracts.
6. Conformance requires business governance; technology only implements the agreement.
7. The payoff is safe cross-process analysis and faster reuse, not one giant enterprise fact table.

## Related patterns

- [Grain](../01-foundations/grain.md)
- [Keys](../01-foundations/keys.md)
- [Dimension patterns](../03-dimensions/dimension-patterns.md)
- [Dates, times, and calendars](../03-dimensions/date-time-calendars.md)
- [Multi-process and heterogeneous models](multi-process-and-heterogeneous-models.md)
- [Semantic layers and governed metrics](../07-modern-architecture/semantic-layer-and-metrics.md)
