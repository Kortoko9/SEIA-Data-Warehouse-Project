# stg_toast_check

Artifact Type: Power Query M
Build Method: Query
Dependencies: brz_toast_orders_bulk
Grain: One row per check
Last Updated: April 27, 2026
Layer: Silver
Notes: Nested checks expanded from orders payload.
Primary / Merge Key: restaurant_guid + check_guid
Purpose: Normalize checks/tabs into one row per check.
Source Objects: brz_toast_orders_bulk
Status: Documented
Target Object: stg_toast_check

## Implementation Notes

**Code source:** uploaded query export from `stgqueries.md.txt`

**Credential handling:** do not paste live client secrets into this page. If adding code, replace any `clientSecret = "..."` value with `clientSecret = "<SECURE_STORE>"`.

## Paste Query Here

```
let
    Source = Lakehouse.Contents([CreateNavigationProperties = false, EnableFolding = false, HierarchicalNavigation = null, PreserveOrder = null]),
    Navigation_1 = Source{[workspaceId = "9a10c019-6b70-4cb0-8fa7-50938b6991ce"]}[Data],
    Navigation_2 = Navigation_1{[lakehouseId = "61fc61d6-f409-4c3d-b460-3162197a75ad"]}[Data],
    BronzeOrders = Navigation_2{[Id = "brz_toast_orders_bulk", ItemKind = "Table"]}?[Data]?,

    AsText = (x as any) as nullable text =>
        try if x = null then null else Text.From(x) otherwise null,

    AsNumber = (x as any) as nullable number =>
        try if x = null then null else Number.From(x) otherwise null,

    ToLogical = (x as any) as nullable logical =>
        try if x = null then null else Logical.From(x) otherwise null,

    FieldOrNull = (r as any, field as text) as any =>
        try if r is record then Record.Field(r, field) else null otherwise null,

    FirstNonNull = (values as list) as any =>
        let
            cleaned = List.RemoveNulls(values)
        in
            if List.Count(cleaned) = 0 then null else cleaned{0},

    GuidFromRecord = (r as any) as nullable text =>
        if r is record then AsText(FieldOrNull(r, "guid")) else null,

    ToBusinessDate = (x as any) as nullable date =>
        let
            s = AsText(x),
            padded = if s = null then null else Text.PadStart(s, 8, "0"),
            yyyy = if padded = null then null else Text.Start(padded, 4),
            mm = if padded = null then null else Text.Range(padded, 4, 2),
            dd = if padded = null then null else Text.Range(padded, 6, 2)
        in
            try if padded = null then null else #date(Number.FromText(yyyy), Number.FromText(mm), Number.FromText(dd)) otherwise null,

    WithOrderRecord =
        Table.AddColumn(
            BronzeOrders,
            "_order",
            each try Json.Document(Text.ToBinary([payload_json], TextEncoding.Utf8)) otherwise null,
            type any
        ),

    KeepValidOrders =
        Table.SelectRows(
            WithOrderRecord,
            each [_order] <> null and Value.Is([_order], type record)
        ),

    AddOrderGuid =
        Table.AddColumn(
            KeepValidOrders,
            "order_guid",
            each AsText(FieldOrNull([_order], "guid")),
            type text
        ),

    AddBusinessDate =
        Table.AddColumn(
            AddOrderGuid,
            "business_date",
            each ToBusinessDate(FieldOrNull([_order], "businessDate")),
            type date
        ),

    AddChecks =
        Table.AddColumn(
            AddBusinessDate,
            "_checks",
            each
                let
                    c = FieldOrNull([_order], "checks")
                in
                    if c is list then c else {},
            type list
        ),

    ExpandChecks =
        Table.ExpandListColumn(AddChecks, "_checks"),

    KeepCheckRecords =
        Table.SelectRows(
            ExpandChecks,
            each [_checks] <> null and Value.Is([_checks], type record)
        ),

    RenameCheckRecord =
        Table.RenameColumns(KeepCheckRecords, {{"_checks", "_check"}}),

    AddCheckGuid =
        Table.AddColumn(
            RenameCheckRecord,
            "check_guid",
            each AsText(FieldOrNull([_check], "guid")),
            type text
        ),

    KeepChecksOnly =
        Table.SelectRows(AddCheckGuid, each [check_guid] <> null),

    AddTabName =
        Table.AddColumn(
            KeepChecksOnly,
            "tab_name",
            each FirstNonNull({
                AsText(FieldOrNull([_check], "tabName")),
                AsText(FieldOrNull([_check], "displayName")),
                AsText(FieldOrNull([_check], "name"))
            }),
            type text
        ),

    AddPaymentStatus =
        Table.AddColumn(
            AddTabName,
            "payment_status",
            each FirstNonNull({
                AsText(FieldOrNull([_check], "paymentStatus")),
                AsText(FieldOrNull([_check], "status"))
            }),
            type text
        ),

    AddTaxExemptFlag =
        Table.AddColumn(
            AddPaymentStatus,
            "tax_exempt_flag",
            each FirstNonNull({
                ToLogical(FieldOrNull([_check], "taxExempt")),
                ToLogical(FieldOrNull([_check], "taxExempted"))
            }),
            type logical
        ),

    AddAmountSubtotal =
        Table.AddColumn(
            AddTaxExemptFlag,
            "amount_subtotal",
            each FirstNonNull({
                AsNumber(FieldOrNull([_check], "subtotalAmount")),
                AsNumber(FieldOrNull([_check], "amount")),
                AsNumber(FieldOrNull([_check], "preDiscountAmount"))
            }),
            type number
        ),

    AddAmountTax =
        Table.AddColumn(
            AddAmountSubtotal,
            "amount_tax",
            each FirstNonNull({
                AsNumber(FieldOrNull([_check], "taxAmount")),
                AsNumber(FieldOrNull([_check], "totalTax"))
            }),
            type number
        ),

    AddAmountTotal =
        Table.AddColumn(
            AddAmountTax,
            "amount_total",
            each FirstNonNull({
                AsNumber(FieldOrNull([_check], "totalAmount")),
                AsNumber(FieldOrNull([_check], "amount"))
            }),
            type number
        ),

    AddOpenedDate =
        Table.AddColumn(
            AddAmountTotal,
            "opened_date_utc",
            each FirstNonNull({
                AsText(FieldOrNull([_check], "openedDate")),
                AsText(FieldOrNull([_order], "openedDate"))
            }),
            type text
        ),

    AddClosedDate =
        Table.AddColumn(
            AddOpenedDate,
            "closed_date_utc",
            each FirstNonNull({
                AsText(FieldOrNull([_check], "closedDate")),
                AsText(FieldOrNull([_order], "closedDate"))
            }),
            type text
        ),

    AddModifiedDate =
        Table.AddColumn(
            AddClosedDate,
            "modified_date_utc",
            each FirstNonNull({
                AsText(FieldOrNull([_check], "modifiedDate")),
                AsText(FieldOrNull([_order], "modifiedDate"))
            }),
            type text
        ),

    AddGuestReference =
        Table.AddColumn(
            AddModifiedDate,
            "guest_reference",
            each
                let
                    checkCustomer = FieldOrNull([_check], "customer"),
                    orderCustomer = FieldOrNull([_order], "customer")
                in
                    FirstNonNull({
                        GuidFromRecord(checkCustomer),
                        AsText(FieldOrNull(checkCustomer, "externalId")),
                        AsText(FieldOrNull(checkCustomer, "displayName")),
                        GuidFromRecord(orderCustomer),
                        AsText(FieldOrNull(orderCustomer, "externalId")),
                        AsText(FieldOrNull(orderCustomer, "displayName"))
                    }),
            type text
        ),

    SelectSilverColumns =
        Table.SelectColumns(
            AddGuestReference,
            {
                "restaurant_guid",
                "check_guid",
                "order_guid",
                "business_date",
                "tab_name",
                "payment_status",
                "tax_exempt_flag",
                "amount_subtotal",
                "amount_tax",
                "amount_total",
                "opened_date_utc",
                "closed_date_utc",
                "modified_date_utc",
                "guest_reference",
                "ingested_at_utc"
            }
        ),

    Sorted =
        Table.Sort(
            SelectSilverColumns,
            {
                {"modified_date_utc", Order.Descending},
                {"ingested_at_utc", Order.Descending}
            }
        ),

    Buffered = Table.Buffer(Sorted),

    Deduped =
        Table.Distinct(
            Buffered,
            {"restaurant_guid", "check_guid"}
        )
in
    Deduped
```

## Validation Notes

- Row count checked: ☐
- Key fields populated: ☐
- Nulls / duplicates checked: ☐
- Reconciliation impact understood: ☐