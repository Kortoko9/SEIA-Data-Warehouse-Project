# Reconciliation Rules

## Purpose

Defines the reconciliation logic for the SEIA Toast / Fabric reporting system.

This page explains how sales-side values, tender-side values, discounts, taxes, tips, and service charges should be compared so finance reporting remains consistent and auditable.

## Current Reconciliation Mart

Primary Gold table:

`mart_reconciliation_daily`

Primary build notebook:

`NB_BUILD_RECON_MART`

Current grain:

`restaurant_guid + business_date`

## Source Tables

| Area | Table | Use |
| --- | --- | --- |
| Checks | stg_toast_check | Sales-side check totals |
| Selections | stg_toast_selection | Item-level sales support |
| Payments | stg_toast_payment | Tender/payment totals |
| Service Charges | stg_toast_check_service_charge | Service charge totals |
| Check Discounts | stg_toast_check_discount | Check-level discounts |
| Selection Discounts | stg_toast_selection_discount | Item-level discounts |
| Taxes | stg_toast_selection_tax | Selection-level tax bridge |

## Core Grain Rule

All daily reconciliation outputs must aggregate to:

`restaurant_guid + business_date`

Do not mix this with calendar date unless explicitly documented.

## Sales-Side Rule

Sales-side totals come primarily from `stg_toast_check`.

Core fields:

| Metric | Source Field | Source Table |
| --- | --- | --- |
| Check Count | count distinct check_guid | stg_toast_check |
| Order Count | count distinct order_guid | stg_toast_check |
| Check Subtotal | amount_subtotal | stg_toast_check |
| Check Tax | amount_tax | stg_toast_check |
| Check Total | amount_total | stg_toast_check |

Primary sales-side total:

`check_total_amount = SUM(stg_toast_check.amount_total)`

## Tender-Side Rule

Tender-side totals come from `stg_toast_payment`.

Primary reporting date:

`paid_business_date`

Core fields:

| Metric | Source Field | Source Table |
| --- | --- | --- |
| Payment Count | count distinct payment_guid | stg_toast_payment |
| Payment Amount | amount | stg_toast_payment |
| Tip Amount | tip_amount | stg_toast_payment |

Primary tender-side total:

`payment_amount = SUM(stg_toast_payment.amount)`

## Main Variance Rule

Primary variance:

`check_total_amount - payment_amount`

Stored as:

`sales_vs_tender_variance`

Variance percentage:

`sales_vs_tender_variance / check_total_amount`

Stored as:

`sales_vs_tender_variance_pct`

## Discount Rule

Use actual applied dollars, not configured values.

Correct field:

`discount_amount`

Do not use:

`discount_value`

Reason:

`discount_value` may represent the configured percentage or rule value. It is not the actual applied discount dollars.

Discount total:

`total_discount_amount = check_discount_amount + selection_discount_amount`

## Tax Rule

Taxes are tracked from both check and selection tax structures.

Current mart fields:

| Metric | Meaning |
| --- | --- |
| check_tax_amount | Tax amount at check level |
| selection_tax_bridge_amount | Tax amount from selection tax bridge |
| selection_tax_amount_from_selection | Tax amount directly on selection rows |

Rule:

Do not assume these always match until validated against Toast reports.

## Tips vs Service Charges Rule

Tips and service charges must remain separate.

| Item | Source | Treatment |
| --- | --- | --- |
| Tips | stg_toast_payment.tip_amount | Payment/tender-side gratuity |
| Service Charges | stg_toast_check_service_charge.amount | Check-side service charge |

Do not combine tips and service charges unless a report explicitly requires a combined gratuity view.

## Business Date Rule

Sales side uses:

`business_date`

Tender side uses:

`paid_business_date`, normalized into `business_date` inside the mart.

Reason:

Sales and payments can have different timing. The mart intentionally separates source logic before aggregating by daily business date.

## Null Handling Rule

All numeric reconciliation metrics should coalesce null values to zero before final variance logic.

Reason:

Missing rows from one side of the join should not produce null variance calculations.

## Replay / Late Change Rule

Reconciliation must account for:

- late payments
- refunds
- voids
- modified orders
- changed discounts
- changed service charges

The Gold mart must support re-running and updating prior business dates.

## Current Tolerance Rule

Initial tolerance status:

`Not finalized`

Recommended starting rule:

| Variance Type | Suggested Status |
| --- | --- |
| $0.00 variance | PASS |
| Small rounding variance | REVIEW |
| Meaningful dollar variance | FAIL / Investigate |

Final thresholds should be set after comparing against Toast reports.

## Validation Checklist

Before marking reconciliation complete:

| Check | Complete? |
| --- | --- |
| Required Silver tables exist | ☐ |
| Required columns exist | ☐ |
| Mart builds successfully | ☐ |
| Row count logged to ctl_rowcount_audit | ☐ |
| Pipeline run logged to ctl_pipeline_run | ☐ |
| Check totals compared to Toast report | ☐ |
| Payment totals compared to Toast report | ☐ |
| Discount totals reviewed | ☐ |
| Tax totals reviewed | ☐ |
| Service charge totals reviewed | ☐ |
| Variances explained | ☐ |

## Known Current Status

`NB_BUILD_RECON_MART` exists and is the build notebook for `mart_reconciliation_daily`.

Validation work on 2026-04-22 identified upstream Bronze/Silver refresh issues before the mart should be considered trusted.

Current validated upstream status for 2026-04-22:

| Area | Result |
| --- | --- |
| Bronze raw orders | 159 |
| Deleted orders | 10 |
| Report-style order count | 149 |
| Toast report order count | 149 |
| Silver payment count | 164 |
| Silver payment amount | $79,371.71 |
| Silver tip amount | $1,513.78 |

The next step is to resolve payment classification logic for `OTHER` payment types before finalizing tender-side reconciliation.

## 2026-04-22 Validation Findings

### Confirmed Sales-Side Match

After filtering reconciliation to `payment_status = CLOSED` checks:

| Metric | Fabric | Toast | Status |
| --- | --- | --- | --- |
| --- | ---: | ---: | --- |
| Check Total | $71,279.50 | $71,279.50 | MATCH |
| Service Charge / Gratuity | $10,853.42 | $10,853.42 | MATCH |
| Tips | $1,513.78 | $1,513.78 | MATCH |

Conclusion:

Sales-side logic is now confirmed for closed checks.

### Required Mart Logic Change Applied

Financial reconciliation must use CLOSED checks only.

Affected sections in `NB_BUILD_RECON_MART`:

- `check_daily`
- `service_charge_daily`
- `check_discount_daily`

Implementation pattern:

- Build `closed_check_keys` from `stg_toast_check`
- Filter `stg_toast_check.payment_status = 'CLOSED'`
- Join check-based support tables to `closed_check_keys`

### Remaining Payment-Side Issue

Current Fabric payment amount for 2026-04-22:

| Payment Type | Payment Count | Payment Amount | Tip Amount |
| --- | --- | --- | --- |
| --- | ---: | ---: | ---: |
| CASH | 1 | $99.16 | $0.00 |
| CREDIT | 133 | $33,385.31 | $1,473.78 |
| HOUSE_ACCOUNT | 1 | $4.40 | $0.00 |
| OTHER | 29 | $45,882.84 | $40.00 |

Toast Sales Summary shows:

| Toast Metric | Amount |
| --- | --- |
| --- | ---: |
| Payments Subtotal | $61,814.94 |
| Deposit Sales Collected | $9,464.56 |
| Total Amount | $71,279.50 |

Payment classification issue is isolated to `payment_type = OTHER`.

### OTHER Payment Breakdown

| other_payment_reference | Payment Count | Payment Amount | Tip Amount |
| --- | --- | --- | --- |
| --- | ---: | ---: | ---: |
| 3de10cc8-96d7-4eb6-8ac1-d411736d54ed | 5 | $41,767.80 | $0.00 |
| d45433c4-bef2-476c-bd30-23f7300616e5 | 24 | $4,115.04 | $40.00 |

Next diagnostic query:

```python
from pyspark.sql import functions as F

spark.table("stg_dim_alt_payment_type") \
    .select(
        "alt_payment_type_guid",
        "alt_payment_type_name",
        "payment_type",
        "active_flag"
    ) \
    .where(
        F.col("alt_payment_type_guid").isin(
            "3de10cc8-96d7-4eb6-8ac1-d411736d54ed",
            "d45433c4-bef2-476c-bd30-23f7300616e5"
        )
    ) \
    .show(truncate=False)
```

Decision update as of 2026-04-29:

`OTHER` payment references have been identified from `stg_dim_alt_payment_type`:

- `3de10cc8-96d7-4eb6-8ac1-d411736d54ed` = `Deposit Redeem`
- `d45433c4-bef2-476c-bd30-23f7300616e5` = `PV House Account`

Current mart behavior:

- `PV House Account` is included in `payment_amount` as normal tender.
- `Deposit Redeem` is isolated into `deposit_redeem_amount` and excluded from normal `payment_amount`.
- `unclassified_other_payment_amount` is 0.00 for 2026-04-22.

Latest validation for 2026-04-22:

- `payment_count` = 159
- `payment_amount` = 37,603.91
- `payment_tip_amount` = 1,513.78
- `deposit_redeem_amount` = 41,767.80
- `sales_vs_tender_variance` = 33,675.59

Important finding:

All five `Deposit Redeem` payments are tied to CLOSED checks on 2026-04-22, but four payment rows are tied to the same check. Therefore, `Deposit Redeem` cannot be resolved by simple GUID-level include/exclude logic. The next rule must determine how Toast splits Deposit Redeem between Payments Subtotal, Deposit Sales Collected, and excluded/overapplied deposit activity.

## Development Rule

Do not change reconciliation formulas casually.

If a formula changes, document:

1. Old formula
2. New formula
3. Reason for change
4. Impact on prior reporting
5. Whether historical mart rows need replay

## Next Improvement

Add automated writes into `ctl_recon_result` so each reconciliation metric can be stored as a PASS / REVIEW / FAIL result with variance thresholds.