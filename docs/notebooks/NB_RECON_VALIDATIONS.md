# NB_RECON_VALIDATIONS

Artifact Type: Notebook
Build Method: Notebook
Dependencies: brz_toast_payments; stg_toast_payment; stg_toast_check; stg_dim_alt_payment_type; mart_reconciliation_daily
Grain: Diagnostic — no output table
Last Updated: May 14, 2026
Layer: QA / Validation
Notes: Run after NB_BUILD_RECON_MART. Never add validation cells to NB_BUILD_RECON_MART — keep build and validation notebooks separate.
Primary / Merge Key: N/A
Purpose: Validate reconciliation mart row counts, date coverage, variance spot checks, and Deposit Redeem payment diagnostics.
Source Objects: brz_toast_payments; stg_toast_payment; stg_toast_check; stg_dim_alt_payment_type; mart_reconciliation_daily
Status: Documented
Target Object: None — read-only diagnostics

## Run Order

Run cells in this order. Diagnostics 1–4 are sequential (each depends on a DataFrame defined in the previous cell).

1. Bronze coverage check
2. Silver coverage check
3. Mart spot check — April 22
4. Mart all-dates overview
5. Deposit Redeem Diagnostic 1 — full detail
6. Deposit Redeem Diagnostic 2 — duplicate fingerprint (depends on Diagnostic 1)
7. Deposit Redeem Diagnostic 3 — check payment composition (depends on Diagnostic 1)
8. Deposit Redeem Diagnostic 4 — check total vs all payments (depends on Diagnostic 1)

---

## Section 1 — Bronze Coverage

### Cell 1 — brz_toast_payments row count by extraction window

```
%%sql
SELECT extract_window_start_utc, COUNT(*) AS row_count
FROM brz_toast_payments
GROUP BY extract_window_start_utc
ORDER BY extract_window_start_utc
```

Expected: rows spanning from 20260313 to today. Only two extract_window_start_utc values are normal — the backfill batch (20260313) and any subsequent rolling-window runs.

---

## Section 2 — Silver Coverage

### Cell 2 — stg_toast_payment row count by paid_business_date

```
%%sql
SELECT paid_business_date, COUNT(*) AS payment_count
FROM stg_toast_payment
GROUP BY paid_business_date
ORDER BY paid_business_date
```

Expected: rows from 2026-03-13 to today with no gaps on trading days.

---

## Section 3 — Mart Spot Checks

### Cell 3 — April 22 reconciliation row

```
%%sql
SELECT
    business_date,
    check_total_amount,
    payment_amount,
    deposit_redeem_amount,
    unclassified_other_payment_amount,
    payment_tip_amount,
    sales_vs_tender_variance
FROM mart_reconciliation_daily
WHERE business_date = '2026-04-22'
```

### Cell 4 — All dates overview

```
%%sql
SELECT
    business_date,
    check_total_amount,
    payment_amount,
    sales_vs_tender_variance
FROM mart_reconciliation_daily
ORDER BY business_date
```

Watch for: large unexplained variances, missing dates, April 23 anomaly (~$284K check_total with only 22 payments — known data issue, do not treat as valid).

---

## Section 4 — Deposit Redeem Diagnostics

Run cells 5–8 in sequence. Cells 6, 7, and 8 reference DataFrames created in Cell 5.

### Cell 5 — Diagnostic 1: Full Deposit Redeem payment records

Purpose: Inspect all Deposit Redeem payment rows for April 22 to identify duplicates, split tenders, void/refund artifacts, or separate events.

```python
from pyspark.sql import functions as F

VALIDATION_BUSINESS_DATE = F.to_date(F.lit("2026-04-22"))
DEPOSIT_REDEEM_GUID = "3de10cc8-96d7-4eb6-8ac1-d411736d54ed"

deposit_redeem_full_detail = (
    spark.table("stg_toast_payment").alias("p")
    .join(
        spark.table("stg_dim_alt_payment_type").alias("apt"),
        F.col("p.other_payment_reference") == F.col("apt.alt_payment_type_guid"),
        "left"
    )
    .where(
        (F.col("p.paid_business_date") == VALIDATION_BUSINESS_DATE) &
        (F.col("p.payment_type") == "OTHER") &
        (F.col("p.other_payment_reference") == DEPOSIT_REDEEM_GUID)
    )
    .select(
        "p.restaurant_guid",
        "p.payment_guid",
        "p.order_guid",
        "p.check_guid",
        "p.paid_business_date",
        "p.void_business_date",
        "p.refund_business_date",
        "p.paid_date_utc",
        F.round(F.col("p.amount"), 2).alias("amount"),
        F.round(F.coalesce(F.col("p.tip_amount"), F.lit(0)), 2).alias("tip_amount"),
        "p.payment_type",
        "p.card_payment_id",
        "p.tender_transaction_guid",
        "p.house_account_reference",
        "p.other_payment_reference",
        "apt.alt_payment_type_name",
        "p.server_guid",
        "p.cash_drawer_guid",
        "p.modified_date_utc",
        "p.ingested_at_utc"
    )
    .orderBy("p.check_guid", F.col("amount").desc(), "p.payment_guid")
)

display(deposit_redeem_full_detail)
```

### Cell 6 — Diagnostic 2: Deposit Redeem duplicate fingerprint test

Purpose: Identify whether rows are duplicated by non-GUID business fields. Depends on `deposit_redeem_full_detail` from Cell 5.

```python
from pyspark.sql import functions as F

deposit_redeem_duplicate_fingerprint = (
    deposit_redeem_full_detail
    .groupBy(
        "restaurant_guid",
        "order_guid",
        "check_guid",
        "paid_business_date",
        "paid_date_utc",
        "amount",
        "tip_amount",
        "payment_type",
        "other_payment_reference",
        "alt_payment_type_name",
        "tender_transaction_guid",
        "card_payment_id",
        "house_account_reference",
        "void_business_date",
        "refund_business_date"
    )
    .agg(
        F.countDistinct("payment_guid").alias("payment_guid_count"),
        F.collect_set("payment_guid").alias("payment_guids"),
        F.min("ingested_at_utc").alias("first_ingested_at_utc"),
        F.max("ingested_at_utc").alias("last_ingested_at_utc")
    )
    .where(F.col("payment_guid_count") > 1)
    .orderBy(F.col("payment_guid_count").desc(), F.col("amount").desc())
)

display(deposit_redeem_duplicate_fingerprint)
```

Expected: empty result if no duplicates. Any rows here indicate the same business event was ingested multiple times with different payment_guids.

### Cell 7 — Diagnostic 3: Full payment composition for checks with Deposit Redeem

Purpose: See all tenders on the checks where Deposit Redeem appears. Depends on `deposit_redeem_full_detail` from Cell 5.

```python
from pyspark.sql import functions as F

deposit_redeem_check_keys = (
    deposit_redeem_full_detail
    .select("restaurant_guid", "check_guid")
    .distinct()
)

check_payment_composition = (
    spark.table("stg_toast_payment").alias("p")
    .join(
        deposit_redeem_check_keys.alias("k"),
        (F.col("p.restaurant_guid") == F.col("k.restaurant_guid")) &
        (F.col("p.check_guid") == F.col("k.check_guid")),
        "inner"
    )
    .join(
        spark.table("stg_dim_alt_payment_type").alias("apt"),
        F.col("p.other_payment_reference") == F.col("apt.alt_payment_type_guid"),
        "left"
    )
    .select(
        F.col("p.restaurant_guid"),
        F.col("p.order_guid"),
        F.col("p.check_guid"),
        F.col("p.payment_guid"),
        F.col("p.paid_business_date"),
        F.col("p.void_business_date"),
        F.col("p.refund_business_date"),
        F.col("p.paid_date_utc"),
        F.col("p.payment_type"),
        F.col("apt.alt_payment_type_name"),
        F.round(F.col("p.amount"), 2).alias("amount"),
        F.round(F.coalesce(F.col("p.tip_amount"), F.lit(0)), 2).alias("tip_amount"),
        F.col("p.tender_transaction_guid"),
        F.col("p.card_payment_id"),
        F.col("p.other_payment_reference"),
        F.col("p.house_account_reference")
    )
    .orderBy("check_guid", "payment_type", F.col("amount").desc(), "payment_guid")
)

display(check_payment_composition)

check_payment_composition_summary = (
    check_payment_composition
    .groupBy(
        "check_guid",
        "payment_type",
        "alt_payment_type_name"
    )
    .agg(
        F.countDistinct("payment_guid").alias("payment_count"),
        F.round(F.sum("amount"), 2).alias("payment_amount"),
        F.round(F.sum("tip_amount"), 2).alias("tip_amount")
    )
    .orderBy("check_guid", "payment_type", "alt_payment_type_name")
)

display(check_payment_composition_summary)
```

### Cell 8 — Diagnostic 4: Linked check total vs all payments on Deposit Redeem checks

Purpose: Compare check totals to all payment rows tied to those checks. Depends on `deposit_redeem_check_keys` from Cell 7.

```python
from pyspark.sql import functions as F

deposit_redeem_check_payment_vs_sales = (
    spark.table("stg_toast_check").alias("c")
    .join(
        deposit_redeem_check_keys.alias("k"),
        (F.col("c.restaurant_guid") == F.col("k.restaurant_guid")) &
        (F.col("c.check_guid") == F.col("k.check_guid")),
        "inner"
    )
    .join(
        spark.table("stg_toast_payment").alias("p"),
        (F.col("c.restaurant_guid") == F.col("p.restaurant_guid")) &
        (F.col("c.check_guid") == F.col("p.check_guid")),
        "left"
    )
    .groupBy(
        F.col("c.restaurant_guid"),
        F.col("c.order_guid"),
        F.col("c.check_guid"),
        F.col("c.business_date").alias("check_business_date"),
        F.col("c.payment_status"),
        F.col("c.tab_name")
    )
    .agg(
        F.round(F.max(F.col("c.amount_total")), 2).alias("check_total_amount"),
        F.countDistinct("p.payment_guid").alias("all_payment_count"),
        F.round(F.sum(F.coalesce(F.col("p.amount"), F.lit(0))), 2).alias("all_payment_amount"),
        F.round(F.sum(F.coalesce(F.col("p.tip_amount"), F.lit(0))), 2).alias("all_tip_amount")
    )
    .withColumn(
        "all_payment_less_check_total",
        F.round(F.col("all_payment_amount") - F.col("check_total_amount"), 2)
    )
    .orderBy(F.col("all_payment_less_check_total").desc())
)

display(deposit_redeem_check_payment_vs_sales)
```

## Validation Notes

- Bronze coverage confirmed: ☐
- Silver date coverage confirmed: ☐
- April 22 variance investigated: ☐
- Deposit Redeem duplicates checked: ☐
- April 23 anomaly investigated in Toast: ☐
