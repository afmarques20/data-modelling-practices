# Layered Architectures: Medallion, Data Vault, and Kimball

Imagine an online shop receives an order. The same order may appear in several useful forms:

1. the original message, kept so it can be replayed;
2. a cleaned record with valid types and a resolved customer;
3. an order-line fact that analysts can safely total;
4. a governed `Net Sales` metric used by dashboards.

Those forms are not unnecessary copies. Each one has a different job.

Medallion, Data Vault, and dimensional modeling are sometimes presented as competing choices. In practice, they usually answer different questions:

- **Medallion:** how does data become progressively cleaner and more useful?
- **Data Vault:** how can integrated source history remain auditable and resilient to change?
- **Kimball dimensional modeling:** how should business-facing analytical data be shaped for understandable, correct analysis?

The useful question is therefore not “which label wins?” Ask instead: “What job does each layer perform, and where does raw data become understandable business data?”

## One flow, several responsibilities

```mermaid
flowchart LR
    S[Operational sources and events] --> R[Replayable raw data]
    R --> I[Cleaned and integrated history]
    I --> D[Dimensional business models]
    D --> M[Semantic metrics]
    M --> C[BI, analytics, data products, AI]
```

The boxes can be tables, views, files, streams, or managed pipelines. Their responsibilities still matter:

1. keep a replayable copy of what the source sent;
2. clean the data and connect records that refer to the same thing;
3. state what each analytical row represents;
4. publish shared dimensions and measures that aggregate correctly;
5. give business metrics one governed meaning.

## Medallion architecture

[Databricks describes medallion](https://docs.databricks.com/aws/en/lakehouse/medallion) as progressive data-quality layers commonly named Bronze, Silver, and Gold.

### Bronze: source fidelity

Bronze is the closest layer to the source. **Source fidelity** means preserving what arrived, even when it is incomplete or awkward.

Typical responsibilities:

- land source records with minimal transformation;
- preserve payload, source identifiers, arrival timestamps, and file/message lineage;
- support replay, audit, and schema-drift investigation;
- quarantine unreadable data without losing evidence.

One Bronze row usually means whatever one source delivery row or message meant. That may not be a useful analytical grain. A table being queryable does not make it a trusted business model.

### Silver: cleaning and integration

Silver turns source-shaped data into dependable reusable data. For example, it may parse `"80.00"` into a number, remove a retried order message, and map two source customer IDs to one customer identity.

Typical responsibilities:

- parse types and standardize codes;
- deduplicate source records;
- apply change data capture (CDC) and delete semantics;
- reconcile identifiers across sources;
- enforce quality rules;
- preserve or construct integrated history.

Silver can contain normalized integration tables, Data Vault structures, cleaned event streams, or other reusable models. It is not required to be one specific schema style.

### Gold: business-facing data products

Gold is designed for a business question or consumer. An order-line star with `net_sales_amount`, Product, Customer, and Date is a typical Gold model.

Typical responsibilities:

- dimensional facts and dimensions;
- governed entity or wide marts where that is the chosen consumption contract;
- aggregates and feature/serving tables;
- documented measures and access policies.

Databricks' current [dimensional-modeling guidance](https://docs.databricks.com/aws/en/ldp/best-practices/dimensional-modeling) places star schemas naturally in Gold. That is an architectural fit, not a rule that every Gold table must be a star.

### Mental model

> Medallion says how trust increases across layers. Kimball says how analytical meaning is organized for consumers.

### What medallion does not decide

The labels do not tell you:

- whether one fact row means an order, order line, payment, or shipment;
- whether a balance can be summed through time;
- whether a customer change needs Type 1 or Type 2 history;
- how facts across domains conform;
- which numerator and denominator define a KPI.

Those remain modeling and governance decisions.

## Optional: Data Vault overview

Data Vault is most relevant when many changing sources must be integrated with strong audit history. If you are learning the basics, remember only this idea: it separates stable business identities, their relationships, and their changing descriptions.

### Hubs

A hub stores a stable business identity, such as customer `C-1042`. It does not try to hold the customer's changing name, segment, or address.

```text
hub_customer
  customer_hk
  customer_bk
  load_timestamp
  record_source
```

### Links

A link records a relationship between business identities. For example, it can say that customer `C-1042` is connected to subscription `S-88`.

```text
link_customer_subscription
  customer_subscription_hk
  customer_hk
  subscription_hk
  load_timestamp
  record_source
```

### Satellites

A satellite stores descriptions that can change, such as the customer's name, segment, or country, together with their history.

```text
sat_customer_profile
  customer_hk
  hashdiff
  effective_timestamp
  load_timestamp
  name
  segment
  country
```

Precise conventions vary across Data Vault methods and implementations. The important idea is the separation: identity, relationships, and changing descriptions are loaded independently, while retaining where each assertion came from.

### Raw Vault and Business Vault

- **Raw Vault** keeps source-derived history in a form that can be traced back to the source.
- **Business Vault** adds reusable business rules, such as deciding that two source records represent the same customer or choosing the preferred value when sources disagree.
- **Information marts** reshape the integrated history for consumers, often as dimensional facts and dimensions.

The [Data Vault Alliance introduction](https://datavaultalliance.com/engineering/data-vault-2-0-an-introduction/) and an [AWS implementation guide](https://aws.amazon.com/blogs/big-data/design-and-build-a-data-vault-model-in-amazon-redshift-from-a-transactional-database/) both describe dimensional information marts as a natural downstream consumption form.

```mermaid
flowchart LR
    A[Sources] --> B[Landing / staging]
    B --> C[Raw Vault<br/>hubs, links, satellites]
    C --> D[Business Vault<br/>integrated rules]
    D --> E[Dimensional marts<br/>facts and dimensions]
    E --> F[Semantic layer and BI]
```

### Mental model

> Data Vault retains and integrates evidence; a dimensional mart turns that evidence into a usable analytical contract.

## How Data Vault and Kimball differ

| Concern | Data Vault strength | Dimensional-model strength |
|---|---|---|
| Source change resilience | Add hubs/links/satellites with limited impact | Requires mart mapping changes |
| Audit and lineage | Load metadata and source-oriented history are structural | Usually handled by pipeline/audit columns |
| Parallel ingestion | Independent structures can load concurrently | Facts often wait for dimension key resolution |
| End-user query simplicity | Low if exposed directly | High through stars and business labels |
| Measure aggregation | Not the primary organizing principle | Central design concern |
| Cross-process usability | Requires a consumption model | Conformed dimensions and facts are explicit |
| Delivery latency | Good for persistent integration; marts still required | Directly serves analytical use cases |

Do not force business users to reconstruct a star by joining hubs, links, and satellites. A technically complete integration layer is not automatically a usable semantic model.

## Common coexistence patterns

### Medallion plus Kimball

```text
Bronze: replayable source fidelity
Silver: cleaned, deduplicated, integrated records
Gold: dimensional facts, dimensions, and selected aggregates
Semantic: governed metrics and consumption contracts
```

This is a strong default when the organization does not need a formal enterprise Data Vault.

### Medallion plus Data Vault plus Kimball

```text
Bronze / staging -> Raw Vault -> Business Vault -> dimensional Gold marts -> semantic layer
```

Use this when traceability, many changing sources, parallel loading, and long-lived integrated history justify the extra structures. It is usually unnecessary for a small project with a few stable sources.

### Dimensional models without a persistent integration layer

Smaller teams may build dimensional marts directly from cleaned source staging. This can be effective when sources and scope are controlled. Preserve replayable inputs, key mappings, tests, and conformance contracts so the approach can evolve.

### Wide marts or semantic graphs

Some dbt-style projects expose wide entity-grained marts, and semantic engines may resolve joins dynamically. These can preserve the same lasting principles — clear grain, governed keys, consistent history, and safe measures — without a physically obvious star. Do not confuse table shape with modeling discipline.

## Architecture decision guide

### Start with the question people need to answer

Ask:

1. Which business processes and decisions must the platform support?
2. What is the atomic grain of each process?
3. Which historical truth must be retained?
4. How many sources and identity conflicts exist?
5. Must the team be able to reconstruct exactly what each source said at an earlier time?
6. What latency, cost, and team skills constrain the design?

### Add layers only when they own a responsibility

| Need | Likely response |
|---|---|
| Replay after faulty transformation | Immutable/replayable landing data |
| Simple cleanup from a few stable sources | Silver integration models may be sufficient |
| Auditable multi-source enterprise history | Consider Raw/Business Vault |
| Usable BI and flexible filtering/grouping | Dimensional marts or equally clear semantic rules |
| Consistent KPIs across tools | Governed semantic/metric layer |

Adding a layer just because an architecture diagram looks complete creates more work and more places for logic to drift. Omitting a necessary responsibility pushes that work into every downstream team.

## Principles that survive technology changes

| Lasting modeling principle | Technology-specific choice |
|---|---|
| Declare grain before facts and dimensions | Whether each star is physically materialized |
| Preserve atomic data somewhere trustworthy | Whether it lives in an RDBMS, object store, or lakehouse table |
| Conform shared business meaning | Whether joins are prebuilt, views, or resolved by a semantic engine |
| Preserve required history | Whether implemented by SQL MERGE, snapshots, streams, or system-versioned tables |
| Make aggregation behavior explicit | Whether aggregates are tables, materialized views, cubes, or cache |
| Keep source lineage and restartability | Exact orchestration and storage technology |

Cheap storage and elastic compute make some implementation choices easier. They do not make mixed grain, double counting, or unclear history correct.

## Common mistakes

- Calling Bronze, Silver, and Gold a data model rather than layer responsibilities.
- Rebranding raw data as Bronze without making it replayable or traceable.
- Building a Raw Vault and exposing it directly to dashboard authors.
- Treating Data Vault business keys as ready-made conformed dimensions without stewardship.
- Building a separate customer definition in every Gold mart.
- Copying source tables through all layers without adding a business contract.
- Materializing every layer even when a view would provide the same ownership and quality boundary.
- Assuming a semantic layer can repair mixed grain underneath it.

## Tradeoffs

More layers can improve isolation, audit, and reuse, but also add latency, cost, deployment dependencies, and places where logic can diverge. A Data Vault can absorb source change elegantly, but requires specialized skill and an intentional mart-generation strategy. Direct dimensional delivery is simpler, but integration and source-history rules must still live somewhere durable.

Use only the layers whose responsibilities you can explain and own.

## Related patterns

- [Dimensional modeling](../01-foundations/dimensional-modeling.md)
- [Conformance and bus architecture](../05-enterprise-modeling/conformance-and-bus-architecture.md)
- [Semantic layers and metrics](semantic-layer-and-metrics.md)
- [Reliable implementation](reliable-implementation.md)

## What to remember

1. Medallion, Data Vault, and Kimball usually solve different layers of the problem.
2. Bronze preserves source evidence; Silver cleans/integrates; Gold commonly exposes business-facing models.
3. Hubs, links, and satellites are strong integration structures, not the normal BI interface.
4. Dimensional marts can sit downstream of either Silver tables or a Business Vault.
5. Physical stars may be replaced by views or semantic models, but grain, history, conformance, and aggregation rules remain.
6. Every layer must have a clear owner and responsibility.
7. For a small team, replayable source data, a dependable cleaning layer, dimensional marts, and governed metrics are often enough; add Data Vault only for a clear need.
