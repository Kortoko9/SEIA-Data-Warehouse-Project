# stg_toast_selection_discount

Artifact Type: Power Query M
Build Method: Query
Dependencies: brz_toast_orders_bulk; brz_toast_config_discounts
Grain: One row per selection-level discount application
Last Updated: April 27, 2026
Layer: Silver
Notes: Uses config fallback for discount type/value.
Primary / Merge Key: restaurant_guid + selection_guid + discount_guid + ordinal
Purpose: Normalize selection-level applied discounts.
Source Objects: brz_toast_orders_bulk; brz_toast_config_discounts
Status: Documented
Target Object: stg_toast_selection_discount

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
    BronzeDiscounts = Navigation_2{[Id = "brz_toast_config_discounts", ItemKind = "Table"]}?[Data]?,

    AsText = (x as any) as nullable text =>
        try if x = null then null else Text.From(x) otherwise null,

    AsNumber = (x as any) as nullable number =>
        try if x = null then null else Number.From(x) otherwise null,

    FieldOrNull = (r as any, field as text) as any =>
        try if r is record then Record.Field(r, field) else null otherwise null,

    FirstNonNull = (values as list) as any =>
        let
            cleaned = List.RemoveNulls(values)
        in
            if List.Count(cleaned) = 0 then null else cleaned{0},

    GuidFromRecord = (r as any) as nullable text =>
        if r is record then AsText(FieldOrNull(r, "guid")) else null,

    GetListField = (r as any, fieldNames as list) as list =>
        let
            candidates = List.Transform(fieldNames, each FieldOrNull(r, _)),
            listsOnly = List.Select(candidates, each _ is list)
        in
            if List.Count(listsOnly) = 0 then {} else listsOnly{0},

    ToBusinessDate = (x as any) as nullable date =>
        let
            s = AsText(x),
            padded = if s = null then null else Text.PadStart(s, 8, "0"),
            yyyy = if padded = null then null else Text.Start(padded, 4),
            mm = if padded = null then null else Text.Range(padded, 4, 2),
            dd = if padded = null then null else Text.Range(padded, 6, 2)
        in
            try if padded = null then null else #date(Number.FromText(yyyy), Number.FromText(mm), Number.FromText(dd)) otherwise null,

    DiscountTableFromList = (d as list) as table =>
        let
            positions = List.Positions(d),
            paired = List.Zip({positions, d}),
            tbl =
                if List.Count(paired) = 0 then
                    #table(type table [ordinal = Int64.Type, _discount = any], {})
                else
                    Table.FromRows(paired, type table [ordinal = Int64.Type, _discount = any])
        in
            tbl,

    WithDiscountConfigRecord =
        Table.AddColumn(
            BronzeDiscounts,
            "_discount_cfg",
            each try Json.Document(Text.ToBinary([payload_json], TextEncoding.Utf8)) otherwise null,
            type any
        ),

    KeepValidDiscountConfig =
        Table.SelectRows(
            WithDiscountConfigRecord,
            each [_discount_cfg] <> null and Value.Is([_discount_cfg], type record)
        ),

    DiscountConfigProjected =
        Table.FromRecords(
            List.Transform(
                Table.ToRecords(KeepValidDiscountConfig),
                each [
                    discount_guid = AsText(FieldOrNull([_discount_cfg], "guid")),
                    cfg_discount_type = FirstNonNull({
                        AsText(FieldOrNull([_discount_cfg], "type")),
                        AsText(FieldOrNull([_discount_cfg], "discountType"))
                    }),
                    cfg_discount_value = FirstNonNull({
                        AsNumber(FieldOrNull([_discount_cfg], "percentage")),
                        AsNumber(FieldOrNull([_discount_cfg], "percent")),
                        AsNumber(FieldOrNull([_discount_cfg], "amount")),
                        AsNumber(FieldOrNull([_discount_cfg], "value"))
                    })
                ]
            )
        ),

    DiscountConfigDistinct =
        Table.Distinct(DiscountConfigProjected, {"discount_guid"}),

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
            each GetListField([_order], {"checks"}),
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

    AddSelections =
        Table.AddColumn(
            KeepChecksOnly,
            "_selections",
            each GetListField([_check], {"selections"}),
            type list
        ),

    ExpandSelections =
        Table.ExpandListColumn(AddSelections, "_selections"),

    KeepSelectionRecords =
        Table.SelectRows(
            ExpandSelections,
            each [_selections] <> null and Value.Is([_selections], type record)
        ),

    RenameSelectionRecord =
        Table.RenameColumns(KeepSelectionRecords, {{"_selections", "_selection"}}),

    AddSelectionGuid =
        Table.AddColumn(
            RenameSelectionRecord,
            "selection_guid",
            each AsText(FieldOrNull([_selection], "guid")),
            type text
        ),

    KeepSelectionsOnly =
        Table.SelectRows(AddSelectionGuid, each [selection_guid] <> null),

    AddDiscountList =
        Table.AddColumn(
            KeepSelectionsOnly,
            "_discount_list",
            each GetListField([_selection], {"appliedDiscounts", "discounts"}),
            type list
        ),

    AddDiscountTable =
        Table.AddColumn(
            AddDiscountList,
            "_discount_table",
            each DiscountTableFromList([_discount_list]),
            type table [ordinal = Int64.Type, _discount = any]
        ),

    ExpandDiscountTable =
        Table.ExpandTableColumn(
            AddDiscountTable,
            "_discount_table",
            {"ordinal", "_discount"},
            {"ordinal", "_discount"}
        ),

    KeepDiscountRecords =
        Table.SelectRows(
            ExpandDiscountTable,
            each [_discount] <> null and Value.Is([_discount], type record)
        ),

    AddDiscountGuid =
        Table.AddColumn(
            KeepDiscountRecords,
            "discount_guid",
            each
                FirstNonNull({
                    GuidFromRecord(FieldOrNull([_discount], "discount")),
                    GuidFromRecord(FieldOrNull([_discount], "discountType")),
                    GuidFromRecord([_discount]),
                    AsText(FieldOrNull([_discount], "guid"))
                }),
            type text
        ),

    AddDiscountAmount =
        Table.AddColumn(
            AddDiscountGuid,
            "discount_amount",
            each
                FirstNonNull({
                    AsNumber(FieldOrNull([_discount], "discountAmount")),
                    AsNumber(FieldOrNull([_discount], "amount")),
                    AsNumber(FieldOrNull([_discount], "appliedAmount")),
                    AsNumber(FieldOrNull([_discount], "value"))
                }),
            type number
        ),

    AddRawDiscountType =
        Table.AddColumn(
            AddDiscountAmount,
            "_raw_discount_type",
            each
                FirstNonNull({
                    AsText(FieldOrNull([_discount], "type")),
                    AsText(FieldOrNull([_discount], "discountType"))
                }),
            type text
        ),

    AddRawDiscountValue =
        Table.AddColumn(
            AddRawDiscountType,
            "_raw_discount_value",
            each
                FirstNonNull({
                    AsNumber(FieldOrNull([_discount], "percentage")),
                    AsNumber(FieldOrNull([_discount], "percent")),
                    AsNumber(FieldOrNull([_discount], "amount")),
                    AsNumber(FieldOrNull([_discount], "value"))
                }),
            type number
        ),

    JoinDiscountConfig =
        Table.NestedJoin(
            AddRawDiscountValue,
            {"discount_guid"},
            DiscountConfigDistinct,
            {"discount_guid"},
            "_discount_dim",
            JoinKind.LeftOuter
        ),

    ExpandDiscountConfig =
        Table.ExpandTableColumn(
            JoinDiscountConfig,
            "_discount_dim",
            {"cfg_discount_type", "cfg_discount_value"},
            {"cfg_discount_type", "cfg_discount_value"}
        ),

    AddFinalDiscountType =
        Table.AddColumn(
            ExpandDiscountConfig,
            "discount_type",
            each FirstNonNull({[_raw_discount_type], [cfg_discount_type]}),
            type text
        ),

    AddFinalDiscountValue =
        Table.AddColumn(
            AddFinalDiscountType,
            "discount_value",
            each FirstNonNull({[_raw_discount_value], [cfg_discount_value]}),
            type number
        ),

    Final =
        Table.SelectColumns(
            AddFinalDiscountValue,
            {
                "restaurant_guid",
                "selection_guid",
                "check_guid",
                "order_guid",
                "discount_guid",
                "ordinal",
                "business_date",
                "discount_amount",
                "discount_type",
                "discount_value"
            }
        )
in
    Final
```

## Validation Notes

- Row count checked: ☐
- Key fields populated: ☐
- Nulls / duplicates checked: ☐
- Reconciliation impact understood: ☐