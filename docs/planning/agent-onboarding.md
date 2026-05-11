# Agent Onboarding — SEIA Data Warehouse Project

This document briefs an AI coding agent that has no prior context on this project. Read it in full before taking any action.

---

## 1. What this project is

SEIA Club is building a Microsoft Fabric data warehouse on top of the Toast POS Standard API. The goal is a reconciled finance/operations mart in Gold, member-attribution bridges joining Toast tenders to PeopleVine accounts, and Power BI reporting on top of the Gold layer.

Architecture follows a Bronze / Silver / Gold medallion pattern.

- **Bronze:** Power Query M dataflows pulling raw JSON payloads from the Toast API into Fabric Lakehouse Delta tables.
- **Silver:** Power Query M dataflows that normalize Bronze payloads into one row per logical entity (order, check, selection, payment, etc.) with dedup.
- **Gold:** PySpark notebooks that build reconciliation and analytics marts from Silver, plus dimensions via `NB_BUILD_CORE_DIMENSIONS`.
- **Control:** `ctl_*` Delta tables track pipeline runs, row counts, hashes, and audit results.

There is **no PySpark in Bronze or Silver** — those layers are all Power Query M dataflows in Fabric. Notebooks only run in Gold and for control table creation.

---

## 2. Critical reading list

Before writing or modifying anything, read these files in this order:

1. `docs/planning/query-notebook-inventory.csv` — the canonical list of every Bronze/Silver/Gold artifact, what it depends on, its grain, target object, and status. **This is the source of truth for what exists.** If a table is not in this CSV, do not assume it exists.
2. `docs/planning/active-task-tracker.md` — open workstream tasks (T-numbers) with status and notes.
3. `docs/planning/completed-implementation-log.md` — completed tasks with success criteria. Never re-do something logged as Complete without explicit user confirmation.
4. `docs/planning/known-issues.md` — every bug, edge case, and lesson learned. Always check this before debugging.
5. `docs/specs/reconciliation-rules.md` — the reconciliation logic and known variance treatment.
6. `docs/specs/development-standards.md` — coding patterns and conventions.
7. `docs/architecture/architecture.md`, `bronze-layer.md`, `silver-layer.md`, `gold-layer.md`, `data-dictionary.md` — layer-by-layer reference.
8. `docs/queries/` — every Bronze and Silver query, one file per table. The code block in each file is the source of truth for what is in Fabric.
9. `docs/notebooks/` — every Gold/control notebook. Same rule.

---

## 3. Where we are right now

| Workstream | Status | Notes |
|---|---|---|
| Security (T01–T03) | Complete except T03 | Permanent Fabric capacity decision still pending with IT. |
| Fabric setup (T04–T05) | Complete | Workspace and control tables live. |
| Bronze (T06–T11) | Complete | All Bronze tables ingest. Both transactional Bronze queries (`brz_toast_payments`, `brz_toast_orders_bulk`) use a rolling 3-day window for scheduled runs. |
| Silver (T12–T17) | Complete | All Silver tables and core dimensions built. |
| Gold reconciliation mart (T18) | **In Progress** | `mart_reconciliation_daily` is built and producing rows. Void filter applied. Full historical backfill from 2026-03-13 completed. Variances not yet closed — see Section 6. |
| Reconciliation validation (T23) | **In Progress** | April 22 sales-side validated within $4 of Toast. Tender-side variances remain unexplained on multiple dates. |
| PeopleVine (T19–T22) | Not Started | Endpoint inventory not yet confirmed. |
| Power BI (T24–T25) | Not Started | No semantic model exists yet. Will be built on top of Gold. |

---

## 4. Architecture cheat sheet

```
Toast API ──> Bronze (PQM dataflows, append)
                │
                v
            Silver (PQM dataflows, replace + dedup)
                │
                v
            Gold (PySpark notebooks, Delta merge)
                │
                v
            Power BI (not yet built)
```

Source-of-truth files for each layer:
- Bronze query code: `docs/queries/brz_toast_*.md`
- Silver query code: `docs/queries/stg_toast_*.md`
- Gold notebook code: `docs/notebooks/NB_BUILD_RECON_MART.md`, `NB_BUILD_CORE_DIMENSIONS.md`
- Control table DDL: `docs/notebooks/NB_CREATE_CONTROL_TABLES.md`

---

## 5. Key tables and grain

### Bronze (raw payloads)
- `brz_toast_orders_bulk` — one row per raw order payload from `/orders/v2/ordersBulk`. Rolling 3-day window with per-date pagination.
- `brz_toast_payments` — one row per payment detail payload from `/orders/v2/payments/{guid}`. Rolling 3-day window.
- `brz_toast_restaurants`, `brz_toast_menus*`, `brz_toast_config_*` — config and snapshot tables with no date param.

### Silver (normalized, one-row-per-entity)
- `stg_toast_order`, `stg_toast_check`, `stg_toast_selection`, `stg_toast_selection_modifier`, `stg_toast_selection_tax`, `stg_toast_selection_discount`, `stg_toast_check_service_charge`, `stg_toast_check_discount` — all derived from `brz_toast_orders_bulk`.
- `stg_toast_payment` — derived from `brz_toast_payments`. Has `void_business_date` and `refund_business_date` columns.
- `stg_dim_*`, `stg_bridge_*` — built by `NB_BUILD_CORE_DIMENSIONS` from Bronze config/menu/restaurant tables.

### Gold
- `mart_reconciliation_daily` — grain: `restaurant_guid + business_date`. First Gold mart.

---

## 6. Active reconciliation issues (READ BEFORE TOUCHING GOLD)

These are the open variance investigations in `mart_reconciliation_daily`:

### 6.1 Deposit Redeem treatment
Toast Deposit Redeem (alt payment type GUID `3de10cc8-96d7-4eb6-8ac1-d411736d54ed`) is an `OTHER` payment type that closes real checks but should not inflate normal payment subtotal. It is bucketed into `deposit_redeem_amount` separately from `payment_amount`.

`sales_vs_tender_variance` currently equals `check_total_amount - payment_amount`. Because `payment_amount` excludes Deposit Redeem, the variance overstates the gap by exactly the Deposit Redeem amount. We are evaluating whether to change the formula to subtract `deposit_redeem_amount` as well.

### 6.2 April 23, 2026 anomaly
`mart_reconciliation_daily` shows ~$284,599 in `check_total_amount` for 2026-04-23 with only 22 payments. This is not a real business day — needs manual investigation in Toast. Do not treat this row as valid.

### 6.3 Date misalignment
Toast records a check on its open `business_date` and payments on `paid_business_date`. Checks opened one day and paid the next will appear on different mart rows. This is a structural cause of variances on most dates and won't fully resolve without tolerance thresholds or date-range matching.

### 6.4 Voided payments
`NB_BUILD_RECON_MART` filters `void_business_date IS NULL` in the `payment_enriched` DataFrame. Any new payment aggregation must do the same. Voided rows are retained in Bronze and Silver for audit.

### 6.5 PV House Account
Alt payment type GUID `d45433c4-bef2-476c-bd30-23f7300616e5` is PV House Account, classified as `PAYMENT_SUBTOTAL` (real tender). Do not move it to a separate bucket.

---

## 7. Rules for working in this repo

### Branching
- Your branch: create a feature branch off `main` (e.g. `coworker/<your-name>-<topic>`). Do not commit to `claude/reorganize-files-structure-PZDfs` — that is Kris's active branch.
- Always pull main first before branching.
- Push your branch only — never push to main.

### Documentation is mandatory, not optional
- Every code change to a Fabric query, notebook, or table must update the corresponding `docs/queries/*.md` or `docs/notebooks/*.md` file in the same commit.
- Any new artifact must be added to `docs/planning/query-notebook-inventory.csv`.
- Task progress must be reflected in `docs/planning/active-task-tracker.md` (move tasks between In Progress / Complete) and `docs/planning/completed-implementation-log.md` (when a task is done).
- New bugs, edge cases, or lessons must be appended to `docs/planning/known-issues.md`.

### Credentials
- Never commit a live `clientSecret`. In all doc query blocks, replace any real secret with `<SECURE_STORE>`. The real value lives only in Fabric.

### Fabric dataflow destination settings
- All Bronze transactional dataflow destinations (`brz_toast_payments`, `brz_toast_orders_bulk`) **must be set to Append**. Replace mode will wipe historical data.
- Silver destinations can use Replace (they rebuild from Bronze with dedup).
- Toggle off "Use automatic settings" in the destination panel to access the update method.

### Date windows
- Never hardcode a single `businessDate` in a Bronze query. Always use a rolling window for scheduled runs.
- For one-time backfills, use a date range with `List.Numbers` and switch back to the rolling window when done.

### Table references
- Only reference table names that appear in `docs/planning/query-notebook-inventory.csv`. Do not assume a table exists because it was mentioned in a spec.

---

## 8. Workstream guidance

### If you are working on Power BI (T24/T25)
- The semantic model and reports do not exist yet. Build them on top of `mart_reconciliation_daily` to start.
- Connect Power BI to the Fabric Lakehouse SQL endpoint.
- Use `restaurant_guid + business_date` as the grain for the first finance view.
- Do not build measures that depend on the resolution of the active reconciliation issues (Section 6) — those numbers are still moving.
- Document the first semantic model in `docs/reporting/` (create the doc if it doesn't exist) and add a row to `docs/planning/query-notebook-inventory.csv` describing it.

### If you are working on PeopleVine (T19–T22)
- T19 (endpoint inventory) is the blocker. Nothing downstream can proceed until the PeopleVine source list is confirmed.
- Read `docs/specs/peoplevine-bridge-spec.md` first.
- Bronze ingestion will follow the same PQM pattern as Toast Bronze: Append destination, rolling window, full-payload-into-JSON, payload_hash fingerprint.

### If you are working on Gold marts
- Read `docs/notebooks/NB_BUILD_RECON_MART.md` end to end. The void filter, payment bucket classification, and merge logic are the patterns to follow.
- New marts go in `docs/notebooks/NB_BUILD_*.md`.
- Always log to `ctl_pipeline_run` and `ctl_rowcount_audit` (helper functions are in the existing notebook).

### If you are working on Bronze or Silver
- Talk to Kris first. These layers are complete and stable. Don't refactor them without explicit need.
- If a new Toast endpoint needs to be added, follow the pattern of `brz_toast_payments.md` (rolling window, payload_hash, append destination).

---

## 9. Hard rules — never do these

1. Never commit a live API key, client secret, or token to the repo.
2. Never push to `main`. Never push to another developer's feature branch.
3. Never set a Bronze transactional dataflow destination to Replace mode.
4. Never hardcode a single business date in a Bronze query.
5. Never reference a table name not in `query-notebook-inventory.csv`.
6. Never skip the doc update — code change and doc update go in the same commit.
7. Never modify `mart_reconciliation_daily` schema without first asking Kris. Power BI may already depend on the column set.
8. Never close a task in `active-task-tracker.md` without moving it to `completed-implementation-log.md` with success criteria.
9. Never run `git push --force` or `git reset --hard` on a shared branch.
10. If you are unsure whether a change is safe, ask before acting.

---

## 10. Quick orientation commands

If you have shell access, run these to confirm your environment before starting:

```bash
git status
git branch --show-current
ls docs/queries/ | wc -l       # should be 21 query docs
ls docs/notebooks/ | wc -l     # should be 3 notebook docs
cat docs/planning/active-task-tracker.md
```

If you do not have shell access, request that Kris paste the contents of `query-notebook-inventory.csv` and `active-task-tracker.md` into your context.

---

**Last updated:** This doc reflects state through the April 22 reconciliation validation session. Re-read `active-task-tracker.md` and `known-issues.md` for anything that has moved since.
