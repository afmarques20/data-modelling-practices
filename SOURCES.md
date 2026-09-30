# Sources and Interpretation Notes

## Primary source

- Ralph Kimball and Margy Ross, *The Data Warehouse Toolkit: The Definitive Guide to Dimensional Modeling*, 3rd ed., Wiley, 2013.

This handbook is an original educational synthesis. It paraphrases concepts, reorganizes them around modeling decisions, and uses new examples. It does not reproduce the book or attempt to preserve its chapter order.

Chapter 2 is the compact catalog of techniques. Chapters 3–16 provide deeper case-study reasoning; chapter 16 closes with common mistakes. Chapters 18–20 ground the design workflow and reliable data-pipeline notes. See [COVERAGE.md](COVERAGE.md) for the topic-level map.

## Authoritative modern references

These sources support modern implementation notes. They supplement rather than redefine the Kimball material.

### dbt

- [How dbt Labs structures marts](https://docs.getdbt.com/best-practices/how-we-structure/4-marts)
- [Semantic models](https://docs.getdbt.com/docs/build/semantic-models)
- [dbt Semantic Layer](https://docs.getdbt.com/docs/use-dbt-semantic-layer/dbt-sl)
- [Snapshots for Type 2 history](https://docs.getdbt.com/docs/build/snapshots)

### Databricks

- [Medallion architecture](https://docs.databricks.com/aws/en/lakehouse/medallion)
- [Dimensional modeling best practices](https://docs.databricks.com/aws/en/ldp/best-practices/dimensional-modeling)
- [Lakehouse data-warehousing concepts](https://docs.databricks.com/aws/en/sql/get-started/data-warehousing-concepts)

### Microsoft Fabric and Power BI

- [Dimensional modeling in Microsoft Fabric Warehouse](https://learn.microsoft.com/en-us/fabric/data-warehouse/dimensional-modeling-overview)
- [Power BI star-schema guidance](https://learn.microsoft.com/en-us/power-bi/guidance/star-schema)
- [Fact-table guidance](https://learn.microsoft.com/en-us/fabric/data-warehouse/dimensional-modeling-fact-tables)

### SAP

- [SAP Datasphere semantic modeling](https://help.sap.com/docs/SAP_DATASPHERE/c8a54ee704e94e15926551293243fd1d/5c1e3d4a49554fcd8fcf199d664d1109.html)
- [SAP Datasphere fact entities](https://help.sap.com/docs/SAP_DATASPHERE/c8a54ee704e94e15926551293243fd1d/30089bd2aa754ab996a62cf5842ae60a.html)
- [SAP Datasphere dimension entities](https://help.sap.com/docs/SAP_DATASPHERE/c8a54ee704e94e15926551293243fd1d/5aae0e95361a4a4c964e69c52eada87d.html)
- [SAP HANA calculation views with star joins](https://help.sap.com/docs/hana-cloud-database/sap-hana-cloud-sap-hana-database-modeling-guide-for-sap-web-ide-full-stack/create-calculation-views-with-star-joins)

### Data Vault

- [Data Vault 2.0: an introduction](https://datavaultalliance.com/engineering/data-vault-2-0-an-introduction/)
- [Serving data from a Data Vault](https://datavaultalliance.com/engineering/getting-data-out-of-a-data-vault-is-a-lot-easier-than-some-people-think/)
- [AWS: Raw Vault, Business Vault, and information marts](https://aws.amazon.com/blogs/big-data/design-and-build-a-data-vault-model-in-amazon-redshift-from-a-transactional-database/)

## Interpretation guardrails

- **Medallion is a layering and data-quality progression.** Dimensional modeling is a business-facing analytical modeling method. A Gold layer often contains dimensional models, but the terms are not synonyms.
- **A dbt mart is not automatically a Kimball star.** Wide entity marts and semantic-graph models can preserve grain and governed meaning without the same physical shape.
- **SAP Datasphere does not require every dimensional concept to be a physical star.** Facts, dimensions, associations, hierarchies, and measures may be expressed in logical and semantic models.
- **Data Vault and Kimball commonly serve different layers.** Raw and Business Vault models preserve auditable integration and change; dimensional information marts make that history usable for analysis.
- **Modern names are not retroactively attributed to Kimball.** Bronze/Silver/Gold, Data Vault, dbt, lakehouse, HANA Cloud, Datasphere, bitemporal terminology, event time, and idempotency are explicitly labeled as later connections.
