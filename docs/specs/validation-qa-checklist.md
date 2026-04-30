# Validation / QA Checklist

## Status

Active QA Framework / Expands During Implementation

## Purpose

Defines the validation checks required before SEIA Toast / Fabric data can be considered reliable for reporting.

This page answers:

- Did the data load?
- Did the expected tables build?
- Are keys populated?
- Are duplicates controlled?
- Do totals reconcile to Toast?
- Are exceptions documented?

## Validation Philosophy

A table is not considered complete just because it exists.

A table is only considered complete when:

1. It builds successfully.
2. Its grain is confirmed.
3. Its keys are populated.
4. Row counts are reviewed.
5. Duplicates are checked.
6. Relevant totals reconcile to source reports.
7. Issues are logged.

## Layer 1 — Source / Bronze Validation

| Check | Purpose | Complete? |
| --- | --- | --- |
| Toast authentication succeeds | Confirms API access | ☐ |
| Required endpoint returns data | Confirms source availability | ☐ |
| Bronze table exists | Confirms landing table was created | ☐ |
| payload_json populated | Confirms raw payload was captured | ☐ |
| payload_hash populated | Supports dedupe/change detection | ☐ |
| ingested_at_utc populated | Supports auditability | ☐ |
| source_endpoint populated | Confirms lineage | ☐ |
| restaurant_guid populated | Supports multi-location reporting | ☐ |
| extract window populated where applicable | Supports incremental audit | ☐ |

## Bronze Tables to Validate

| Table | Expected Status |
| --- | --- |
| brz_toast_orders_bulk | Built |
| brz_toast_payments | Built |
| brz_toast_restaurants | Built |
| brz_toast_menus_metadata | Built |
| brz_toast_menus | Built |
| brz_toast_config_revenue_centers | Built |
| brz_toast_config_sales_categories | Built |
| brz_toast_config_dining_options | Built |
| brz_toast_config_discounts | Built |
| brz_toast_config_service_charges | Built |
| brz_toast_config_tax_rates | Built |
| brz_toast_config_alt_payment_types | Built |
| brz_toast_config_tip_withholding | Built |

## Layer 2 — Silver Validation

| Check | Purpose | Complete? |
| --- | --- | --- |
| Silver table exists | Confirms transformation ran | ☐ |
| Grain matches documented design | Prevents duplicate/misleading rows | ☐ |
| Primary/natural key populated | Ensures joinability | ☐ |
| Duplicate keys checked | Prevents double counting | ☐ |
| Required money fields populated | Supports finance reporting | ☐ |
| Required date fields populated | Supports daily reporting | ☐ |
| Rowcount audit recorded | Confirms QA lineage | ☐ |
| Data dictionary updated | Keeps documentation synced | ☐ |
| Query / Notebook Inventory updated | Keeps implementation documented | ☐ |

## Silver Tables to Validate

| Table | Grain | Primary / Natural Key |
| --- | --- | --- |
| stg_toast_order | One row per order | restaurant_guid + order_guid |
| stg_toast_check | One row per check | restaurant_guid + check_guid |
| stg_toast_selection | One row per selection | restaurant_guid + selection_guid |
| stg_toast_selection_modifier | One row per modifier line | restaurant_guid + parent_selection_guid + modifier_position |
| stg_toast_payment | One row per payment | restaurant_guid + payment_guid |
| stg_toast_check_service_charge | One row per check service charge | restaurant_guid + check_guid + service_charge_guid + ordinal |
| stg_toast_check_discount | One row per check discount | restaurant_guid + check_guid + discount_guid + ordinal |
| stg_toast_selection_discount | One row per selection discount | restaurant_guid + selection_guid + discount_guid + ordinal |
| stg_toast_selection_tax | One row per selection tax | restaurant_guid + selection_guid + tax_guid + ordinal |

## Layer 3 — Dimension Validation

| Check | Purpose | Complete? |
| --- | --- | --- |
| Dimension table exists | Confirms dimension build ran | ☐ |
| Dimension key populated | Enables joins | ☐ |
| Row hash populated | Enables change tracking | ☐ |
| Rowcount audit recorded | Confirms QA trail | ☐ |
| Hash-change audit recorded | Confirms change lineage | ☐ |
| Bridge tables populated where expected | Enables menu/modifier analysis | ☐ |

## Dimension / Reference Tables to Validate

| Table | Expected Grain |
| --- | --- |
| stg_dim_restaurant | One row per restaurant |
| stg_dim_revenue_center | One row per revenue center |
| stg_dim_sales_category | One row per sales category |
| stg_dim_dining_option | One row per dining option |
| stg_dim_discount | One row per discount |
| stg_dim_service_charge | One row per service charge |
| stg_dim_tax_rate | One row per tax rate |
| stg_dim_alt_payment_type | One row per alternate payment type |
| stg_dim_menu_group | One row per menu group |
| stg_dim_menu_item | One row per menu item |
| stg_dim_modifier_group | One row per modifier group |
| stg_dim_modifier_option | One row per modifier option |
| stg_bridge_menu_item_modifier_group | One row per menu item / modifier group relationship |
| stg_bridge_modifier_group_option | One row per modifier group / modifier option relationship |

## Layer 4 — Gold / Mart Validation

| Check | Purpose | Complete? |
| --- | --- | --- |
| Gold mart exists | Confirms reporting table built | ☐ |
| Mart grain confirmed | Prevents duplicate daily reporting | ☐ |
| Required source tables checked before build | Prevents silent failure | ☐ |
| Required columns checked before build | Prevents schema drift | ☐ |
| Merge/upsert completed | Confirms idempotent refresh | ☐ |
| Rowcount audit recorded | Confirms mart QA trail | ☐ |
| Pipeline run logged | Confirms execution trace | ☐ |
| Variance logic reviewed | Confirms finance logic | ☐ |

## Current Gold Mart to Validate

| Table | Grain | Primary Key |
| --- | --- | --- |
| mart_reconciliation_daily | One row per restaurant per business date | restaurant_guid + business_date |

## Reconciliation Validation

| Metric | Source / Formula | Check Against |
| --- | --- | --- |
| check_count | COUNT DISTINCT check_guid | Toast report check count |
| order_count_from_checks | COUNT DISTINCT order_guid | Toast orders/checks report |
| check_subtotal_amount | SUM amount_subtotal | Toast sales report |
| check_tax_amount | SUM amount_tax | Toast tax report |
| check_total_amount | SUM amount_total | Toast check total |
| payment_count | COUNT DISTINCT payment_guid | Toast payments report |
| payment_amount | SUM payment amount | Toast payments report |
| payment_tip_amount | SUM tip amount | Toast tips/payments report |
| service_charge_amount | SUM service charge amount | Toast service charge report |
| total_discount_amount | check discounts + selection discounts | Toast discount/comp report |
| selection_tax_bridge_amount | SUM selection tax bridge | Toast tax detail report |
| sales_vs_tender_variance | check_total_amount - payment_amount | Internal variance review |

## Toast Report Comparison Checklist

For each validation period:

| Check | Complete? |
| --- | --- |
| Confirm Toast report date range matches Fabric business_date logic | ☐ |
| Confirm Toast report export time / timezone | ☐ |
| Compare sales totals | ☐ |
| Compare payment totals | ☐ |
| Compare tips | ☐ |
| Compare discounts/comps | ☐ |
| Compare service charges | ☐ |
| Compare taxes | ☐ |
| Explain all variances | ☐ |
| Log unresolved items | ☐ |

## Control Table QA

| Control Table | Validation Requirement |
| --- | --- |
| ctl_pipeline_run | Every notebook/pipeline run should log success/failure |
| ctl_rowcount_audit | Every important build should log row counts |
| ctl_hash_change_audit | Dimension merge changes should be captured |
| ctl_recon_result | Future recon results should be stored here |
| ctl_data_quality_issue | Significant issues should be logged here |

## Issue Severity Guide

| Severity | Meaning | Example |
| --- | --- | --- |
| High | Blocks trusted reporting | Missing required table, major variance |
| Medium | Needs review before publishing | Small unexplained variance, partial source mismatch |
| Low | Documentation or minor QA issue | Missing note, field naming ambiguity |

## Required Before Dashboard Use

Before a table or metric is used in Power BI:

| Requirement | Complete? |
| --- | --- |
| Metric definition exists in KPI Catalog | ☐ |
| Source table exists | ☐ |
| Grain is documented | ☐ |
| Validation query/check completed | ☐ |
| Reconciliation reviewed if financial metric | ☐ |
| Known limitations documented | ☐ |

## Known Current Gaps

| Gap | Impact |
| --- | --- |
| Toast report validation not fully completed | Cannot fully lock KPI definitions |
| Tolerance thresholds not finalized | Variance status cannot be automated yet |
| PeopleVine bridge not implemented | Member spend cannot be trusted yet |
| ctl_recon_result not yet auto-populated | Reconciliation results are not fully reportable yet |

## Development Rule

No table should be marked production-ready unless:

1. It exists.
2. Its grain is documented.
3. Its key fields are populated.
4. Row counts are checked.
5. Duplicates/null keys are reviewed.
6. Financial metrics are reconciled against source reports.
7. Any unresolved issue is logged.

## Next Improvement

Add standard validation SQL/notebook snippets for:

- row counts
- duplicate keys
- null keys
- daily sales comparisons
- payment comparisons
- variance thresholds