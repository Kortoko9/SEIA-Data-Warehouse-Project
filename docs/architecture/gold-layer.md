# 🥇 Gold Layer

**Purpose**

*The Gold Layer is the curated reporting layer that turns normalized Silver tables into finance-ready facts, marts, and Power BI-ready datasets.*

**Gold Layer Status Table**

| Area | Status | Notes | Next Step |
| --- | --- | --- | --- |
| Reconciliation Mart | Not Started | First major Gold build item. | Build using Silver sales and payment tables. |
| Sales Facts | Not Started | Needed for daily sales and check-level reporting. | Build after reconciliation logic is defined. |
| Payments Facts | Not Started | Needed for tender/payment reporting. | Build with stg_toast_payment. |
| Member Spend Mart | Not Started | Depends on PeopleVine bridge. | Build after member attribution prototype. |
| Power BI Semantic Model | Not Started | Final model layer for dashboards. | Build after first Gold marts exist. |

**Target Gold Tables**

| Table Name | Purpose |
| --- | --- |
| fact_sales_check | Check-level sales reporting |
| fact_sales_selection | Item/selection-level sales reporting |
| fact_payments | Payment and tender reporting |
| fact_discounts | Discount and comp reporting |
| fact_service_charges | Service charge reporting |
| fact_refunds_voids | Refund and void reporting |
| fact_member_spend | Member-attributed spend reporting |
| mart_daily_sales | Daily sales summary |
| mart_daily_tenders | Daily tender/payment summary |
| mart_member_spend_daily | Daily member spend reporting |
| mart_discounts_comps_daily | Daily discount and comp reporting |
| mart_service_charges_daily | Daily service charge reporting |
| mart_reconciliation_daily | Daily reconciliation summary |

**Immediate Gold Next Steps**

| Priority | Task |
| --- | --- |
| High | Build first reconciliation mart. |
| High | Define sales-side reconciliation logic. |
| High | Define tender-side reconciliation logic. |
| High | Compare Gold outputs against Toast reports. |
| Medium | Build first Power BI-ready semantic model. |