# stg_toast_selection_tax

Artifact Type: Power Query M
Build Method: Query
Dependencies: brz_toast_orders_bulk; brz_toast_config_tax_rates
Grain: One row per selection tax application
Last Updated: April 27, 2026
Layer: Silver
Notes: Uses config fallback for tax name/rate/type.
Primary / Merge Key: restaurant_guid + selection_guid + tax_guid + ordinal
Purpose: Normalize selection tax applications.
Source Objects: brz_toast_orders_bulk; brz_toast_config_tax_rates
Status: Documented
Target Object: stg_toast_selection_tax

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
    BronzeTaxRates = Navigation_2{[Id = "brz_toast_config_tax_rates", ItemKind = "Table"]}?[Data]?,

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

    TaxTableFromList = (d as list) as table =>
        let
            positions = List.Positions(d),
            paired = List.Zip({positions, d}),
            tbl =
                if List.Count(paired) = 0 then
                    #table(type table [ordinal = Int64.Type, _tax = any], {})
                else
                    Table.FromRows(paired, type table [ordinal = Int64.Type, _tax = any])
        in
            tbl,

    WithTaxConfigRecord =
        Table.AddColumn(
            BronzeTaxRates,
            "_tax_cfg",
            each try Json.Document(Text.ToBinary([payload_json], TextEncoding.Utf8)) otherwise null,
            type any
        ),

    KeepValidTaxConfig =
        Table.SelectRows(
            WithTaxConfigRecord,
            each [_tax_cfg] <> null and Value.Is([_tax_cfg], type record)
        ),

    TaxConfigProjected =
        Table.FromRecords(
            List.Transform(
                Table.ToRecords(KeepValidTaxConfig),
                each [
                    tax_guid = AsText(FieldOrNull([_tax_cfg], "guid")),
                    cfg_tax_name = AsText(FieldOrNull([_tax_cfg], "name")),
                    cfg_tax_rate = FirstNonNull({
                        AsNumber(FieldOrNull([_tax_cfg], "rate")),
                        AsNumber(FieldOrNull([_tax_cfg], "percentage")),
                        AsNumber(FieldOrNull([_tax_cfg], "percent"))
                    }),
                    cfg_tax_type = AsText(FieldOrNull([_tax_cfg], "type"))
                ]
            )
        ),

    TaxConfigDistinct =
        Table.Distinct(TaxConfigProjected, {"tax_guid"}),

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

    AddTaxList =
        Table.AddColumn(
            KeepSelectionsOnly,
            "_tax_list",
            each GetListField([_selection], {"appliedTaxes", "taxes"}),
            type list
        ),

    AddTaxTable =
        Table.AddColumn(
            AddTaxList,
            "_tax_table",
            each TaxTableFromList([_tax_list]),
            type table [ordinal = Int64.Type, _tax = any]
        ),

    ExpandTaxTable =
        Table.ExpandTableColumn(
            AddTaxTable,
            "_tax_table",
            {"ordinal", "_tax"},
            {"ordinal", "_tax"}
        ),

    KeepTaxRecords =
        Table.SelectRows(
            ExpandTaxTable,
            each [_tax] <> null and Value.Is([_tax], type record)
        ),

    AddTaxGuid =
        Table.AddColumn(
            KeepTaxRecords,
            "tax_guid",
            each
                FirstNonNull({
                    GuidFromRecord(FieldOrNull([_tax], "taxRate")),
                    GuidFromRecord(FieldOrNull([_tax], "tax")),
                    GuidFromRecord([_tax]),
                    AsText(FieldOrNull([_tax], "guid"))
                }),
            type text
        ),

    AddTaxAmount =
        Table.AddColumn(
            AddTaxGuid,
            "tax_amount",
            each
                FirstNonNull({
                    AsNumber(FieldOrNull([_tax], "taxAmount")),
                    AsNumber(FieldOrNull([_tax], "amount")),
                    AsNumber(FieldOrNull([_tax], "value"))
                }),
            type number
        ),

    AddRawTaxRate =
        Table.AddColumn(
            AddTaxAmount,
            "_raw_tax_rate",
            each
                FirstNonNull({
                    AsNumber(FieldOrNull([_tax], "rate")),
                    AsNumber(FieldOrNull([_tax], "percentage")),
                    AsNumber(FieldOrNull([_tax], "percent"))
                }),
            type number
        ),

    AddRawTaxType =
        Table.AddColumn(
            AddRawTaxRate,
            "_raw_tax_type",
            each AsText(FieldOrNull([_tax], "type")),
            type text
        ),

    JoinTaxConfig =
        Table.NestedJoin(
            AddRawTaxType,
            {"tax_guid"},
            TaxConfigDistinct,
            {"tax_guid"},
            "_tax_dim",
            JoinKind.LeftOuter
        ),

    ExpandTaxConfig =
        Table.ExpandTableColumn(
            JoinTaxConfig,
            "_tax_dim",
            {"cfg_tax_name", "cfg_tax_rate", "cfg_tax_type"},
            {"cfg_tax_name", "cfg_tax_rate", "cfg_tax_type"}
        ),

    AddFinalTaxRate =
        Table.AddColumn(
            ExpandTaxConfig,
            "tax_rate",
            each FirstNonNull({[_raw_tax_rate], [cfg_tax_rate]}),
            type number
        ),

    AddFinalTaxType =
        Table.AddColumn(
            AddFinalTaxRate,
            "tax_type",
            each FirstNonNull({[_raw_tax_type], [cfg_tax_type]}),
            type text
        ),

    Final =
        Table.SelectColumns(
            AddFinalTaxType,
            {
                "restaurant_guid",
                "selection_guid",
                "check_guid",
                "order_guid",
                "tax_guid",
                "ordinal",
                "business_date",
                "tax_amount",
                "tax_rate",
                "tax_type",
                "cfg_tax_name"
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