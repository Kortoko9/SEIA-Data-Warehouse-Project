# stg_toast_selection_modifier

Artifact Type: Power Query M
Build Method: Query
Dependencies: brz_toast_orders_bulk
Grain: One row per modifier line
Last Updated: April 27, 2026
Layer: Silver
Notes: Modifier rows separated from parent selections.
Primary / Merge Key: restaurant_guid + parent_selection_guid + modifier_position
Purpose: Normalize selection modifiers into one row per modifier line.
Source Objects: brz_toast_orders_bulk
Status: Documented
Target Object: stg_toast_selection_modifier

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

    ModifierTableFromList = (mods as list) as table =>
        let
            positions = List.Positions(mods),
            paired = List.Zip({positions, mods}),
            tbl =
                if List.Count(paired) = 0 then
                    #table(type table [modifier_position = Int64.Type, _modifier = any], {})
                else
                    Table.FromRows(paired, type table [modifier_position = Int64.Type, _modifier = any])
        in
            tbl,

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

    AddChecks =
        Table.AddColumn(
            KeepValidOrders,
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

    AddSelections =
        Table.AddColumn(
            RenameCheckRecord,
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

    AddParentSelectionGuid =
        Table.AddColumn(
            RenameSelectionRecord,
            "parent_selection_guid",
            each AsText(FieldOrNull([_selection], "guid")),
            type text
        ),

    KeepSelectionsOnly =
        Table.SelectRows(AddParentSelectionGuid, each [parent_selection_guid] <> null),

    AddModifierList =
        Table.AddColumn(
            KeepSelectionsOnly,
            "_modifier_list",
            each GetListField([_selection], {"modifiers", "modifierOptions", "options"}),
            type list
        ),

    AddModifierTable =
        Table.AddColumn(
            AddModifierList,
            "_modifier_table",
            each ModifierTableFromList([_modifier_list]),
            type table [modifier_position = Int64.Type, _modifier = any]
        ),

    ExpandModifierTable =
        Table.ExpandTableColumn(
            AddModifierTable,
            "_modifier_table",
            {"modifier_position", "_modifier"},
            {"modifier_position", "_modifier"}
        ),

    KeepModifierRecords =
        Table.SelectRows(
            ExpandModifierTable,
            each [_modifier] <> null and Value.Is([_modifier], type record)
        ),

    AddModifierOptionGuid =
        Table.AddColumn(
            KeepModifierRecords,
            "modifier_option_guid",
            each
                FirstNonNull({
                    GuidFromRecord(FieldOrNull([_modifier], "item")),
                    GuidFromRecord(FieldOrNull([_modifier], "modifierOption")),
                    GuidFromRecord(FieldOrNull([_modifier], "option")),
                    GuidFromRecord([_modifier])
                }),
            type text
        ),

    AddModifierGroupGuid =
        Table.AddColumn(
            AddModifierOptionGuid,
            "modifier_group_guid",
            each
                FirstNonNull({
                    GuidFromRecord(FieldOrNull([_modifier], "modifierGroup")),
                    GuidFromRecord(FieldOrNull([_modifier], "group")),
                    GuidFromRecord(FieldOrNull([_modifier], "optionGroup"))
                }),
            type text
        ),

    AddPrice =
        Table.AddColumn(
            AddModifierGroupGuid,
            "price",
            each
                FirstNonNull({
                    AsNumber(FieldOrNull([_modifier], "price")),
                    AsNumber(FieldOrNull([_modifier], "displayPrice")),
                    AsNumber(FieldOrNull([_modifier], "receiptLinePrice")),
                    AsNumber(FieldOrNull([_modifier], "amount"))
                }),
            type number
        ),

    AddTaxAmount =
        Table.AddColumn(
            AddPrice,
            "tax_amount",
            each
                FirstNonNull({
                    AsNumber(FieldOrNull([_modifier], "tax")),
                    AsNumber(FieldOrNull([_modifier], "taxAmount")),
                    AsNumber(FieldOrNull([_modifier], "displayTax"))
                }),
            type number
        ),

    Final =
        Table.SelectColumns(
            AddTaxAmount,
            {
                "restaurant_guid",
                "parent_selection_guid",
                "modifier_position",
                "modifier_option_guid",
                "modifier_group_guid",
                "price",
                "tax_amount"
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