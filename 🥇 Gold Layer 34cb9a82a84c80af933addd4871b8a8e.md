# 🥈 Silver Layer

**Purpose**

*The Silver Layer normalizes raw Bronze payloads into structured, queryable tables. This layer turns nested Toast API data into clean order, check, selection, payment, discount, tax, and service charge tables that can support reconciliation and reporting.*

**Silver Layer Status Table**

| Area | Status | Notes | Next Step |
| --- | --- | --- | --- |
| Orders | Complete | One row per Toast order. | Use in reconciliation mart. |
| Checks | Complete | One row per check/tab. | Validate against Toast sales logic. |
| Selections | Complete | One row per item selection. | Validate item sales totals. |
| Selection Modifiers | Complete | Modifiers separated from parent selections. | Use for menu mix detail. |
| Payments | Complete | Payments modeled separately from orders. | Validate tender totals. |
| Discounts | Complete | Check-level and selection-level discounts built. | Validate comp/discount totals. |
| Service Charges | Complete | Check-level service charges built. | Separate service charges from tips. |
| Taxes | Complete | Selection-level tax applications built. | Validate tax reporting. |
| Core Dimensions | Complete | Restaurant, menu, config dimensions built. | Use in Gold layer. |
| PeopleVine Silver | Not Started | Member/account tables not yet normalized. | Build after PeopleVine source list is confirmed. |

**Primary Silver Tables**

| Table Name | Grain | Purpose |
| --- | --- | --- |
| stg_toast_order | One row per order | Order-level reporting and timestamps |
| stg_toast_check | One row per check | Check/tab-level sales reporting |
| stg_toast_selection | One row per item selection | Item-level sales and menu mix |
| stg_toast_selection_modifier | One row per modifier | Modifier-level detail |
| stg_toast_payment | One row per payment | Tender/payment reporting |
| stg_toast_payment_refund_void | One row per refund/void event | Refund and void tracking |
| stg_toast_check_service_charge | One row per check service charge | Service charge reporting |
| stg_toast_check_discount | One row per check discount | Check-level discount tracking |
| stg_toast_selection_discount | One row per selection discount | Item-level discount tracking |
| stg_toast_selection_tax | One row per selection tax | Tax reporting |

**Silver Dimension Tables**

| Table Name | Purpose |
| --- | --- |
| stg_dim_restaurant | Restaurant dimension |
| stg_dim_revenue_center | Revenue center dimension |
| stg_dim_sales_category | Sales category dimension |
| stg_dim_dining_option | Dining option dimension |
| stg_dim_discount | Discount dimension |
| stg_dim_service_charge | Service charge dimension |
| stg_dim_tax_rate | Tax rate dimension |
| stg_dim_alt_payment_type | Alternate payment type dimension |
| stg_dim_menu_group | Menu group dimension |
| stg_dim_menu_item | Menu item dimension |
| stg_dim_modifier_group | Modifier group dimension |
| stg_dim_modifier_option | Modifier option dimension |

**Silver Standards**

| Standard | Requirement |
| --- | --- |
| Normalized Grain | Each table has a clearly defined grain. |
| Merge Keys | Each table uses stable Toast GUID-based merge keys. |
| Audit Columns | Tables include ingestion/update metadata where possible. |
| Reconciliation Ready | Sales, payments, discounts, taxes, and service charges are separated. |
| Reporting Ready | Tables should support Gold marts without re-parsing raw JSON. |

**Immediate Silver Next Steps**

| Priority | Task |
| --- | --- |
| High | Use Silver tables to build first reconciliation mart. |
| High | Validate sales-side totals against Toast reports. |
| High | Validate payment-side totals against Toast reports. |
| Medium | Build PeopleVine Silver tables after source list is confirmed. |
| Medium | Add unresolved member attribution queue. |