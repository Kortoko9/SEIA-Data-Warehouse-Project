# Known Issues / Lessons Learned

## Status

Active Log / Continuously Updated

## Purpose

Captures known issues, bugs, edge cases, and lessons learned during development of the SEIA Toast / Fabric data platform.

Prevents repeated mistakes and accelerates future development.

---

## Critical Issues

### Missing Silver Table During Build

Issue:

`stg_toast_selection` was missing during downstream builds.

Impact:

- Broke reconciliation mart
- Invalidated assumption that Silver layer was complete

Lesson:

Never assume completion — validate against defined table list.

Prevention:

- Always reconcile expected vs actual tables
- Use require_tables() checks in notebooks

---

### Discount Field Misinterpretation

Issue:

Confusion between:

- `discount_amount`
- `discount_value`

Impact:

- Incorrect discount totals
- Misleading financial reporting

Lesson:

Always use:

`discount_amount`

Reason:

`discount_value` represents configured rule (e.g., %), not actual dollars.

---

### API Credential Exposure Risk

Issue:

Client secret stored directly in query code.

Impact:

- Security vulnerability
- Risk of credential leak

Lesson:

Never store credentials in code or Notion.

Prevention:

- Use secure storage (Fabric / Key Vault)
- Replace with `<SECURE_STORE>` in documentation

---

### Fabric Notebook / Session Instability

Issue:

Notebooks sometimes stuck on:

"Starting session..."

Impact:

- Delays development
- Interrupts builds

Lesson:

Fabric environment is not always stable.

Workaround:

- Restart session
- Switch capacity mode

---

## Data Modeling Lessons

### Payments Must Be Separate from Orders

Lesson:

Do not combine payments with orders.

Reason:

- Different timing (business date vs paid date)
- Refunds and voids occur later

---

### Business Date vs Calendar Date

Lesson:

Always use:

`business_date`

Reason:

Toast reporting is based on business day, not calendar day.

---

### Replay Logic Is Required

Lesson:

Data is not static.

Must handle:

- refunds
- voids
- modified orders

Impact:

Reconciliation will fail without replay logic.

---

## Development Process Lessons

### Documentation Must Lead Implementation

Lesson:

Do not rely on memory or assumptions.

Always:

- update Data Dictionary
- update Query Inventory
- update Architecture pages

---

### AI Needs Strict Context

Lesson:

Without strict documentation, AI will:

- assume missing tables exist
- guess column names
- change patterns

Solution:

Use Notion as source of truth.

---

### Consistency > Speed

Lesson:

Changing between notebooks, queries, and patterns creates confusion.

Rule:

Follow established pattern unless clearly justified.

---

## System Design Lessons

### Control Tables Are Not Optional

Lesson:

A working pipeline is not a production pipeline.

Must include:

- logging
- auditing
- reconciliation tracking

---

### Build for Debugging, Not Just Output

Lesson:

You must be able to explain every number.

Solution:

- audit tables
- rowcount tracking
- hash tracking

---

## Known Gaps (Current)

| Gap | Impact |
| --- | --- |
| PeopleVine not integrated | No member attribution |
| Toast validation not complete | KPIs not fully locked |
| Tolerance thresholds not defined | Variance status unclear |
| ctl_recon_result not populated | No automated recon tracking |

---

## Recent Issues Added

### ordersBulk Pagination / Partial Window Issue

Issue:

`brz_toast_orders_bulk` originally pulled only a rolling 24-hour UTC window and used `pageSize = 100` without pagination.

Impact:

- Bronze landed exactly 100 order rows.
- Silver showed only 95 report-style non-deleted orders for 2026-04-22.
- Toast Orders report showed 149 orders for the same business date.

Root Cause:

The query was not paginating beyond the first 100 orders and the rolling UTC window did not intentionally target the validation business date.

Fix:

- Updated `brz_toast_orders_bulk` for validation to pull business date `20260422`.
- Added pagination logic.
- Bronze orders increased to 159 raw rows.
- After excluding deleted orders, report-style order count matched Toast at 149.

Prevention:

Production orders ingestion must support pagination and controlled incremental/replay windows. Do not validate against Toast from a partial rolling UTC pull.

### Deleted Orders Included in Raw API Payload

Issue:

Toast API returned 159 raw orders for 2026-04-22, while Toast report showed 149 orders.

Finding:

The difference was explained by 10 deleted orders.

Validated Result:

`159 raw orders - 10 deleted orders = 149 report-style orders`

Prevention:

Bronze should retain all raw orders, but report-style Silver/Gold logic must clearly document whether deleted and voided records are included or excluded.

### Silver Dataflow Stale After Bronze Refresh

Issue:

Bronze orders and payments were refreshed correctly, but Silver still showed old counts until the Silver dataflow was refreshed.

Impact:

- Bronze payments showed 164 rows.
- `stg_toast_payment` initially showed 0 payments for 2026-04-22.
- After refreshing Silver, `stg_toast_payment` showed 164 payments, payment amount $79,371.71, and tips $1,513.78.

Lesson:

Refreshing Bronze does not automatically update Silver unless the Silver dataflow/notebook is also refreshed.

Prevention:

Validation sequence must be: Bronze refresh → Silver refresh → Gold mart rebuild → validation checks.

### Toast Last Check Number Is Not Check Count

Issue:

Toast web report showed last check/order number 179, creating confusion because report-style order count was 149.

Lesson:

Last check number is not a count of valid checks/orders. Toast can skip numbers or include numbers tied to deleted, voided, split, or non-final checks.

Prevention:

For validation, compare actual report count and financial totals, not the last check number.

### Hardcoded businessDate in Bronze Transactional Queries

Issue:

Both `brz_toast_payments` and `brz_toast_orders_bulk` had a single hardcoded business date (`"20260422"`) used during initial validation. Neither query fetched more than one day of data.

Impact:

- Bronze tables contained only one date of data.
- All downstream Silver tables and the reconciliation mart reflected only that one date.
- Historical data from March 13 onward was missing entirely.

Fix:

- Replaced hardcoded date with a dynamic rolling 3-day window (`today`, `today-1`, `today-2`) for ongoing scheduled runs.
- For the one-time backfill, replaced with a full date range from `#date(2026, 3, 13)` to today using `List.Numbers`.
- Both Bronze destinations set to **Append** mode before backfill run.

Prevention:

Bronze transactional queries must always use a dynamic date window. Never hardcode a business date in a scheduled query. Validate that `extract_window_start_utc` spans multiple dates after any Bronze refresh.

---

### Fabric Dataflow Replace Mode Wiped Bronze Data

Issue:

`brz_toast_payments` destination was set to Replace (the Fabric default). After switching from a hardcoded date to the rolling window, a dataflow run replaced the entire Bronze table with only 3 days of data, deleting all previously loaded rows.

Impact:

- `payment_amount` in the reconciliation mart dropped to `0.0` for April 22 after the run.
- All historical Bronze payment rows were permanently deleted.

Fix:

- Disabled "Use automatic settings" in the Fabric dataflow destination panel.
- Switched update method to **Append** for all Bronze transactional tables.

Prevention:

All Bronze transactional tables (`brz_toast_payments`, `brz_toast_orders_bulk`) must use **Append** update method. Silver tables (which deduplicate on each run) may use Replace. Never run a Bronze dataflow without confirming the destination mode first.

---

### Voided Payments Inflating Reconciliation Tender Total

Issue:

`NB_BUILD_RECON_MART` was including voided payments in the `payment_amount` and `deposit_redeem_amount` totals. Voided payments have a non-null `void_business_date` in `stg_toast_payment`.

Impact:

- Tender-side total was overstated.
- `sales_vs_tender_variance` was incorrect.
- For April 22: two voided Deposit Redeem payments worth ~$18,404.90 were being included.

Fix:

Added `.where(F.col("p.void_business_date").isNull())` to the `payment_enriched` DataFrame after the payment bucket classification, before the daily aggregation.

Prevention:

Any payment-side aggregation must filter `void_business_date IS NULL`. Voided payments should be retained in Bronze and Silver for auditability but excluded from all financial totals.

---

### Referenced Non-Existent Table stg_toast_payment_refund_void

Issue:

During reconciliation debugging, a query was written referencing `stg_toast_payment_refund_void`. This table does not exist and was never built.

Impact:

Query failed with table-not-found error.

Fix:

Used `void_business_date IS NULL` filter directly on `stg_toast_payment` instead.

Prevention:

Only reference table names confirmed in `query-notebook-inventory.csv`. Do not assume a table exists because it was mentioned in a spec or conversation.

---

## Future Additions

Add new issues here when encountered:

- Data mismatches
- API inconsistencies
- Performance issues
- Fabric limitations
- Power BI issues

---

## Development Rule

Every major issue must be logged here with:

1. What happened
2. Why it happened
3. Impact
4. Fix
5. Prevention

---

## Next Step

Continue updating this page during Gold build, reconciliation validation, and PeopleVine integration.