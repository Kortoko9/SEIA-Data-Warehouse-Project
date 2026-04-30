# Data Dictionary

**Purpose**

*This data dictionary documents the key tables, columns, definitions, source systems, and reporting use cases for the SEIA Toast / Fabric data platform.*

**Table Inventory**

| Layer | Table Name | Source | Grain | Purpose |
| --- | --- | --- | --- | --- |
| Bronze | brz_toast_orders_bulk | Toast API | Raw order payload | Raw Toast orders landing table |
| Bronze | brz_toast_payments | Toast API | Raw payment payload | Raw Toast payment-detail landing table |
| Silver | stg_toast_order | Toast | One row per order | Order-level reporting and timestamps |
| Silver | stg_toast_check | Toast | One row per check/tab | Check-level sales reporting |
| Silver | stg_toast_selection | Toast | One row per item selection | Item/menu sales detail |
| Silver | stg_toast_selection_modifier | Toast | One row per modifier | Modifier-level sales detail |
| Silver | stg_toast_payment | Toast | One row per payment | Tender/payment reporting |
| Silver | stg_toast_check_service_charge | Toast | One row per check service charge | Service charge reporting |
| Silver | stg_toast_check_discount | Toast | One row per check discount | Check-level discount reporting |
| Silver | stg_toast_selection_discount | Toast | One row per selection discount | Item-level discount reporting |
| Silver | stg_toast_selection_tax | Toast | One row per selection tax | Tax reporting |
| Gold | mart_reconciliation_daily | Fabric / Derived | One row per business date | Daily sales vs payment reconciliation |
| Gold | mart_daily_sales | Fabric / Derived | One row per business date / grouping | Daily sales reporting |
| Gold | mart_daily_tenders | Fabric / Derived | One row per business date / tender | Daily tender reporting |
| Gold | mart_member_spend_daily | Fabric / Derived | One row per member per day | Member spend reporting |

**Core Column Dictionary**

| Table | Column | Definition | Source System | Notes |
| --- | --- | --- | --- | --- |
| stg_toast_order | order_guid | Unique Toast order identifier | Toast | Primary order key |
| stg_toast_order | restaurant_guid | Unique Toast restaurant/location identifier | Toast | Required for multi-location scalability |
| stg_toast_order | business_date | Toast business date for the order | Toast | Not always the same as calendar/payment date |
| stg_toast_order | opened_date_utc | Timestamp when order was opened | Toast | UTC timestamp |
| stg_toast_order | closed_date_utc | Timestamp when order was closed | Toast | Used for sales timing |
| stg_toast_order | modified_date_utc | Last modified timestamp | Toast | Important for incremental replay |
| stg_toast_order | server_guid | Toast employee/server identifier | Toast | Joins to employee dimension later |
| stg_toast_order | revenue_center_guid | Revenue center tied to order | Toast | Joins to revenue center dimension |
| stg_toast_order | dining_option_guid | Dining option tied to order | Toast | Dine-in, takeout, etc. |
| stg_toast_order | voided_flag | Indicates whether order was voided | Toast | Used for QA/reconciliation |
| stg_toast_order | deleted_flag | Indicates deleted record status | Toast | Used for exception handling |
| stg_toast_order | ingested_at_utc | Timestamp when record landed in Fabric | Fabric | Audit field |

**Check Table Columns**

| Table | Column | Definition | Source System | Notes |
| --- | --- | --- | --- | --- |
| stg_toast_check | check_guid | Unique Toast check/tab identifier | Toast | Primary check key |
| stg_toast_check | order_guid | Parent order identifier | Toast | Links check to order |
| stg_toast_check | business_date | Toast business date | Toast | Used for daily sales |
| stg_toast_check | tab_name | Check/tab name | Toast | Useful for member/name matching |
| stg_toast_check | payment_status | Payment state of the check | Toast | Helps identify unpaid/closed checks |
| stg_toast_check | amount_subtotal | Check subtotal before tax/fees | Toast | Sales-side metric |
| stg_toast_check | amount_tax | Tax amount on check | Toast | Tax reporting |
| stg_toast_check | amount_total | Total check amount | Toast | Key reconciliation field |
| stg_toast_check | guest_reference | Guest/customer reference if available | Toast | Potential matching field |

**Selection Table Columns**

| Table | Column | Definition | Source System | Notes |
| --- | --- | --- | --- | --- |
| stg_toast_selection | selection_guid | Unique item/selection identifier | Toast | Primary selection key |
| stg_toast_selection | check_guid | Parent check identifier | Toast | Links item to check |
| stg_toast_selection | order_guid | Parent order identifier | Toast | Links item to order |
| stg_toast_selection | menu_item_guid | Menu item identifier | Toast | Joins to menu item dimension |
| stg_toast_selection | menu_group_guid | Menu group identifier | Toast | Joins to menu group dimension |
| stg_toast_selection | sales_category_guid | Sales category identifier | Toast | Food, beverage, etc. |
| stg_toast_selection | quantity | Quantity sold | Toast | Menu mix reporting |
| stg_toast_selection | pre_discount_price | Price before discount | Toast | Gross sales logic |
| stg_toast_selection | price | Final selection price | Toast | Net item value depending logic |
| stg_toast_selection | tax_amount | Tax applied to selection | Toast | Tax reporting |
| stg_toast_selection | voided_flag | Indicates voided item | Toast | Exclude/include based on report logic |
| stg_toast_selection | refund_flag | Indicates refunded item | Toast | Refund analysis |

**Payment Table Columns**

| Table | Column | Definition | Source System | Notes |
| --- | --- | --- | --- | --- |
| stg_toast_payment | payment_guid | Unique Toast payment identifier | Toast | Primary payment key |
| stg_toast_payment | order_guid | Parent order identifier | Toast | Links payment to order |
| stg_toast_payment | check_guid | Parent check identifier | Toast | Links payment to check |
| stg_toast_payment | paid_business_date | Business date payment was paid | Toast | Payment-side reporting date |
| stg_toast_payment | void_business_date | Business date payment was voided | Toast | Used for void replay |
| stg_toast_payment | refund_business_date | Business date payment was refunded | Toast | Used for refund replay |
| stg_toast_payment | paid_date_utc | Actual paid timestamp | Toast | UTC timestamp |
| stg_toast_payment | amount | Payment amount | Toast | Tender-side metric |
| stg_toast_payment | tip_amount | Tip amount | Toast | Must be separated from service charges |
| stg_toast_payment | payment_type | Payment method/type | Toast | Card, cash, house account, other |
| stg_toast_payment | card_payment_id | Card payment reference | Toast | Used for card reconciliation |
| stg_toast_payment | tender_transaction_guid | Tender transaction identifier | Toast | Useful for matching |
| stg_toast_payment | house_account_reference | House account reference | Toast / PV bridge | Key for member attribution |
| stg_toast_payment | other_payment_reference | Other tender reference | Toast | Useful for exception handling |

**Reporting Definitions**

| Metric | Definition | Primary Tables | Notes |
| --- | --- | --- | --- |
| Check Total | Total amount of a Toast check | stg_toast_check | Sales-side total |
| Payment Total | Total amount paid through tenders | stg_toast_payment | Payment-side total |
| Tips | Tip amounts tied to payments | stg_toast_payment | Separate from service charges |
| Service Charges | Toast service charges on checks | stg_toast_check_service_charge | Important for gratuity reporting |
| Discounts / Comps | Discounts applied to checks or selections | stg_toast_check_discount, stg_toast_selection_discount | Separate check vs item discounts |
| Taxes | Taxes applied to selections | stg_toast_selection_tax | Used for tax reporting |
| Member Spend | Spend attributed to member/account | stg_toast_payment + PeopleVine bridge | Requires PeopleVine enrichment |

**Join Map**

| From Table | Join Column | To Table | Join Column | Purpose |
| --- | --- | --- | --- | --- |
| stg_toast_check | order_guid | stg_toast_order | order_guid | Connect checks to orders |
| stg_toast_selection | check_guid | stg_toast_check | check_guid | Connect items to checks |
| stg_toast_payment | check_guid | stg_toast_check | check_guid | Connect payments to checks |
| stg_toast_selection | menu_item_guid | stg_dim_menu_item | menu_item_guid | Add menu item names |
| stg_toast_check | revenue_center_guid | stg_dim_revenue_center | revenue_center_guid | Add revenue center |
| stg_toast_payment | house_account_reference | bridge_toast_payment_to_member | house_account_reference | Attribute spend to member |

**Date Logic**

| Date Field | Meaning | Use For |
| --- | --- | --- |
| business_date | Toast business date | Sales reporting |
| paid_business_date | Business date payment was paid | Tender/payment reporting |
| opened_date_utc | Actual order open timestamp | Operational timing |
| closed_date_utc | Actual order close timestamp | Sales close timing |
| modified_date_utc | Last update timestamp | Incremental refresh/replay |
| ingested_at_utc | Fabric load timestamp | Audit / troubleshooting |

**Known Reporting Rules**

| Rule | Why It Matters |
| --- | --- |
| Payments must be modeled separately from orders | Sales date and payment date can differ. |
| Tips are not the same as service charges | Needed for accurate gratuity reporting. |
| Business date is not always the same as calendar date | Toast reporting may use business-day logic. |
| Member spend requires PeopleVine enrichment | Toast alone may not identify the true member/account. |
| Refunds/voids may affect later periods | Replay windows are needed for accurate reporting. |