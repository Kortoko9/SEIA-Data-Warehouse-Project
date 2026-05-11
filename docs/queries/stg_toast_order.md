# stg_toast_order

Artifact Type: Power Query M
Build Method: Query
Dependencies: brz_toast_orders_bulk
Grain: One row per order
Last Updated: April 27, 2026
Layer: Silver
Notes: Dedupes by restaurant_guid/order_guid using modified and ingested timestamps.
Primary / Merge Key: restaurant_guid + order_guid
Purpose: Normalize orders into one row per Toast order.
Source Objects: brz_toast_orders_bulk
Status: Documented
Target Object: stg_toast_order

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

    FieldOrNull = (r as any, field as text) as any =>
        try if r is record then Record.Field(r, field) else null otherwise null,

    GuidFromRecord = (r as any) as nullable text =>
        if r is record then AsText(FieldOrNull(r, "guid")) else null,

    ToLogical = (x as any) as nullable logical =>
        try if x = null then null else Logical.From(x) otherwise null,

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

    KeepValid =
        Table.SelectRows(
            WithOrderRecord,
            each [_order] <> null and Value.Is([_order], type record)
        ),

    AddOrderGuid =
        Table.AddColumn(
            KeepValid,
            "order_guid",
            each AsText(FieldOrNull([_order], "guid")),
            type text
        ),

    KeepOrdersOnly =
        Table.SelectRows(AddOrderGuid, each [order_guid] <> null),

    AddBusinessDate =
        Table.AddColumn(
            KeepOrdersOnly,
            "business_date",
            each ToBusinessDate(FieldOrNull([_order], "businessDate")),
            type date
        ),

    AddOpenedDate =
        Table.AddColumn(
            AddBusinessDate,
            "opened_date_utc",
            each AsText(FieldOrNull([_order], "openedDate")),
            type text
        ),

    AddClosedDate =
        Table.AddColumn(
            AddOpenedDate,
            "closed_date_utc",
            each AsText(FieldOrNull([_order], "closedDate")),
            type text
        ),

    AddModifiedDate =
        Table.AddColumn(
            AddClosedDate,
            "modified_date_utc",
            each AsText(FieldOrNull([_order], "modifiedDate")),
            type text
        ),

    AddPromisedDate =
        Table.AddColumn(
            AddModifiedDate,
            "promised_date_utc",
            each AsText(FieldOrNull([_order], "promisedDate")),
            type text
        ),

    AddCreatedDate =
        Table.AddColumn(
            AddPromisedDate,
            "created_date_utc",
            each AsText(FieldOrNull([_order], "createdDate")),
            type text
        ),

    AddServerGuid =
        Table.AddColumn(
            AddCreatedDate,
            "server_guid",
            each GuidFromRecord(FieldOrNull([_order], "server")),
            type text
        ),

    AddRevenueCenterGuid =
        Table.AddColumn(
            AddServerGuid,
            "revenue_center_guid",
            each GuidFromRecord(FieldOrNull([_order], "revenueCenter")),
            type text
        ),

    AddDiningOptionGuid =
        Table.AddColumn(
            AddRevenueCenterGuid,
            "dining_option_guid",
            each GuidFromRecord(FieldOrNull([_order], "diningOption")),
            type text
        ),

    AddSourceField =
        Table.AddColumn(
            AddDiningOptionGuid,
            "source",
            each AsText(FieldOrNull([_order], "source")),
            type text
        ),

    AddVoidedFlag =
        Table.AddColumn(
            AddSourceField,
            "voided_flag",
            each ToLogical(FieldOrNull([_order], "voided")),
            type logical
        ),

    AddDeletedFlag =
        Table.AddColumn(
            AddVoidedFlag,
            "deleted_flag",
            each ToLogical(FieldOrNull([_order], "deleted")),
            type logical
        ),

    SelectSilverColumns =
        Table.SelectColumns(
            AddDeletedFlag,
            {
                "restaurant_guid",
                "order_guid",
                "business_date",
                "opened_date_utc",
                "closed_date_utc",
                "modified_date_utc",
                "promised_date_utc",
                "created_date_utc",
                "server_guid",
                "revenue_center_guid",
                "dining_option_guid",
                "source",
                "voided_flag",
                "deleted_flag",
                "ingested_at_utc",
                "record_source"
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
            {"restaurant_guid", "order_guid"}
        )
in
    Deduped
```

## Validation Notes

- Row count checked: ☐
- Key fields populated: ☐
- Nulls / duplicates checked: ☐
- Reconciliation impact understood: ☐