# Development Standards / Code Architecture

## **Purpose**

*This page defines the standard development approach for the SEIA Toast / Fabric data platform so all future build work follows the same architecture, naming conventions, object patterns, and implementation flow.*

*The goal is to keep development consistent across Bronze, Silver, Gold, QA, and Power BI work.*

**Development Rule #1 — Stay Consistent With Existing Pattern**

| Area | Standard |
| --- | --- |
| Bronze ingestion | Use existing Dataflow / query-based pattern unless a notebook is clearly required |
| Silver staging tables | Use query-based transformations where prior stg_* tables were built with queries |
| Gold marts | Use the same transformation style chosen for the first reconciliation mart |
| Control / audit tables | Use the existing ctl_* table structure |
| Notebook usage | Only use notebooks when needed for complex Python/Spark logic, API orchestration, or Delta merge patterns |
| Query usage | Prefer queries for repeatable SQL-style transformations and consistency with current stg_* build flow |

**Standard Build Flow**

| Step | Action |
| --- | --- |
| 1 | Confirm source table exists in Bronze |
| 2 | Confirm target table grain |
| 3 | Define primary/natural key |
| 4 | Define required columns |
| 5 | Build transformation using the established pattern |
| 6 | Validate row counts |
| 7 | Validate totals against source logic |
| 8 | Log completion in tracker |
| 9 | Update data dictionary |
| 10 | Update blockers/risks if issue found |

**Query vs Notebook Decision Rule**

| Use Query When | Use Notebook When |
| --- | --- |
| Building standard stg_* or mart_* tables | Calling APIs directly |
| Transforming existing Bronze/Silver tables | Handling complex nested JSON parsing |
| Logic can be expressed clearly in SQL/M/Power Query | Performing Delta merge/upsert logic |
| Matching existing stg table build pattern | Writing audit/hash/control logic |
| Building repeatable reporting transformations | Orchestrating multiple endpoint calls |

**Current Standard**

| Layer | Preferred Build Method | Notes |
| --- | --- | --- |
| Bronze | Existing ingestion/query pattern | Continue same pattern unless endpoint requires notebook/API orchestration |
| Silver | Query-based transformations | Keep stg_* tables consistent |
| Gold | TBD after first reconciliation mart | First mart should define the future standard |
| Control Tables | Existing ctl_* pattern | Do not redesign unless necessary |
| QA / Reconciliation | Query-first | Use notebooks only for more advanced validation |

**AI Development Instruction**
Paste this into Claude/ChatGPT before asking it to build anything:

> *Follow the existing SEIA Toast / Fabric development standards.*
> 

> *Do not switch between notebooks, queries, pipelines, or other implementation methods unless there is a clear technical reason.*
> 

> *For Silver stg_* tables, use the same query-based pattern already used for the prior stg_toast_* tables.*
> 

> *Before writing code, identify:*
> 
1. *Target layer*
2. *Target table*
3. *Source table(s)*
4. *Grain*
5. *Primary/natural key*
6. *Whether the existing implementation pattern should be query-based or notebook-based*

> *If a different implementation method is recommended, explain why before providing code.*
> 

**Naming Standards**

| Object Type | Naming Pattern | Example |
| --- | --- | --- |
| Bronze table | brz_source_entity | brz_toast_payments |
| Silver staging table | stg_source_entity | stg_toast_payment |
| Dimension table | stg_dim_entity | stg_dim_revenue_center |
| Bridge table | bridge_entity_to_entity | bridge_toast_payment_to_member |
| Fact table | fact_business_process | fact_payments |
| Mart table | mart_subject_grain | mart_reconciliation_daily |
| Control table | ctl_function | ctl_pipeline_run |

**Required Before Marking Any Build Complete**

| Requirement | Complete? |
| --- | --- |
| Table created successfully | ☐ |
| Grain confirmed | ☐ |
| Key fields populated | ☐ |
| Row count checked | ☐ |
| Nulls / duplicates checked | ☐ |
| Reconciliation impact understood | ☐ |
| Data dictionary updated | ☐ |
| Tracker updated | ☐ |

[Technical Build Inventory](Development%20Standards%20Code%20Architecture/Technical%20Build%20Inventory%2034cb9a82a84c807489bdd25e73cad924.md)