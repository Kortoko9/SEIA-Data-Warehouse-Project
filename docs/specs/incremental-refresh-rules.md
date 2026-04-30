# Incremental Refresh Rules

## Purpose

Defines how data is incrementally loaded, refreshed, and replayed across Bronze, Silver, and Gold layers.

This ensures:

- complete data capture
- correct handling of late-arriving data
- accurate reconciliation

## Orders (Toast)

- Use modifiedDate window (NOT just businessDate)
- Always include overlap window (30–60 minutes)
- Deduplicate using:
    - restaurant_guid
    - order_guid

Reason:

Orders can be modified after initial creation.

## Payments (Toast)

- Primary reporting date = paid_business_date
- Must replay using:
    - paidBusinessDate
    - refundBusinessDate
    - voidBusinessDate

Replay rule:

- Always pull current day + prior 2 business dates

Reason:

Refunds and voids occur after original payment date.

## Menus

- Use metadata endpoint to detect changes
- Only reload full menu when metadata changes

Reason:

Menu payloads are large and rarely change.

## Config Tables

- Refresh daily
- Full reload is acceptable

Includes:

- revenue centers
- sales categories
- dining options
- discounts
- service charges
- tax rates

## Replay Strategy

- Allow late-arriving data
- Maintain overlap windows
- Always prioritize correctness over speed

## Silver Layer Impact

- Deduplicate using business keys
- Use latest modified_date_utc
- Maintain idempotent transformations

## Gold Layer Impact

- Reconciliation must reflect replayed data
- Historical values may change due to refunds/voids

## Control Table Dependencies

- ctl_watermark → incremental state
- ctl_endpoint_window_log → API tracking
- ctl_rowcount_audit → data validation

## Critical Rule

Incorrect incremental logic is the #1 cause of reconciliation mismatches.

These rules must be followed exactly for all ingestion and transformation logic.