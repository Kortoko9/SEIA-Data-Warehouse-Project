# stg_toast_selection

Artifact Type: Power Query M
Build Method: Query
Dependencies: brz_toast_orders_bulk
Grain: One row per item selection
Last Updated: April 27, 2026
Layer: Silver
Notes: Includes item/menu/sales category names and counts for modifiers/discounts/taxes.
Primary / Merge Key: restaurant_guid + selection_guid
Purpose: Normalize item selections into one row per selection.
Source Objects: brz_toast_orders_bulk
Status: Documented
Target Object: stg_toast_selection

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

    AsLogical = (x as any) as nullable logical =>
        try
            if x = null then null
            else if x is logical then x
            else if Text.Lower(Text.From(x)) = "true" then true
            else if Text.Lower(Text.From(x)) = "false" then false
            else null
        otherwise null,

    FieldOrNull = (r as any, field as text) as any =>
        try if r is record then Record.Field(r, field) else null otherwise null,

    PathOrNull = (r as any, path as list) as any =>
        try
            List.Accumulate(
                path,
                r,
                (state, current) =>
                    if state is record then Record.Field(state, current) else null
            )
        otherwise null,

    FirstNonNull = (values as list) as any =>
        let
            cleaned = List.RemoveNulls(values)
        in
            if List.Count(cleaned) = 0 then null else cleaned{0},

    GuidFromRecord = (r as any) as nullable text =>
        if r is record then AsText(FieldOrNull(r, "guid")) else null,

    NameFromRecord = (r as any) as nullable text =>
        if r is record then AsText(FieldOrNull(r, "name")) else null,

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

    ToDateTimeZone = (x as any) as nullable datetimezone =>
        try if x = null then null else DateTimeZone.From(x) otherwise null,

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

    AddOrderModifiedDate =
        Table.AddColumn(
            AddBusinessDate,
            "_order_modified_date_utc",
            each ToDateTimeZone(
                FirstNonNull({
                    FieldOrNull([_order], "modifiedDate"),
                    FieldOrNull([_order], "modifiedDateUtc")
                })
            ),
            type datetimezone
        ),

    AddChecks =
        Table.AddColumn(
            AddOrderModifiedDate,
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

    AddMenuItemGuid =
        Table.AddColumn(
            KeepSelectionsOnly,
            "menu_item_guid",
            each
                FirstNonNull({
                    GuidFromRecord(FieldOrNull([_selection], "item")),
                    GuidFromRecord(FieldOrNull([_selection], "menuItem")),
                    AsText(FieldOrNull([_selection], "menuItemGuid")),
                    AsText(FieldOrNull([_selection], "itemGuid"))
                }),
            type text
        ),

    AddMenuItemName =
        Table.AddColumn(
            AddMenuItemGuid,
            "menu_item_name",
            each
                FirstNonNull({
                    NameFromRecord(FieldOrNull([_selection], "item")),
                    NameFromRecord(FieldOrNull([_selection], "menuItem")),
                    AsText(FieldOrNull([_selection], "itemName")),
                    AsText(FieldOrNull([_selection], "name")),
                    AsText(FieldOrNull([_selection], "displayName"))
                }),
            type text
        ),

    AddMenuGroupGuid =
        Table.AddColumn(
            AddMenuItemName,
            "menu_group_guid",
            each
                FirstNonNull({
                    GuidFromRecord(FieldOrNull([_selection], "itemGroup")),
                    GuidFromRecord(FieldOrNull([_selection], "menuGroup")),
                    GuidFromRecord(FieldOrNull([_selection], "group")),
                    AsText(FieldOrNull([_selection], "menuGroupGuid")),
                    AsText(FieldOrNull([_selection], "itemGroupGuid"))
                }),
            type text
        ),

    AddMenuGroupName =
        Table.AddColumn(
            AddMenuGroupGuid,
            "menu_group_name",
            each
                FirstNonNull({
                    NameFromRecord(FieldOrNull([_selection], "itemGroup")),
                    NameFromRecord(FieldOrNull([_selection], "menuGroup")),
                    NameFromRecord(FieldOrNull([_selection], "group")),
                    AsText(FieldOrNull([_selection], "menuGroupName")),
                    AsText(FieldOrNull([_selection], "itemGroupName"))
                }),
            type text
        ),

    AddSalesCategoryGuid =
        Table.AddColumn(
            AddMenuGroupName,
            "sales_category_guid",
            each
                FirstNonNull({
                    GuidFromRecord(FieldOrNull([_selection], "salesCategory")),
                    AsText(FieldOrNull([_selection], "salesCategoryGuid"))
                }),
            type text
        ),

    AddSalesCategoryName =
        Table.AddColumn(
            AddSalesCategoryGuid,
            "sales_category_name",
            each
                FirstNonNull({
                    NameFromRecord(FieldOrNull([_selection], "salesCategory")),
                    AsText(FieldOrNull([_selection], "salesCategoryName"))
                }),
            type text
        ),

    AddQuantity =
        Table.AddColumn(
            AddSalesCategoryName,
            "quantity",
            each AsNumber(FieldOrNull([_selection], "quantity")),
            type number
        ),

    AddPreDiscountPrice =
        Table.AddColumn(
            AddQuantity,
            "pre_discount_price",
            each
                FirstNonNull({
                    AsNumber(FieldOrNull([_selection], "preDiscountPrice")),
                    AsNumber(FieldOrNull([_selection], "pre_discount_price"))
                }),
            type number
        ),

    AddPrice =
        Table.AddColumn(
            AddPreDiscountPrice,
            "price",
            each
                FirstNonNull({
                    AsNumber(FieldOrNull([_selection], "price")),
                    AsNumber(FieldOrNull([_selection], "displayPrice")),
                    AsNumber(FieldOrNull([_selection], "receiptLinePrice"))
                }),
            type number
        ),

    AddTaxAmount =
        Table.AddColumn(
            AddPrice,
            "tax_amount",
            each
                FirstNonNull({
                    AsNumber(FieldOrNull([_selection], "taxAmount")),
                    AsNumber(FieldOrNull([_selection], "tax")),
                    AsNumber(FieldOrNull([_selection], "taxInclusionAmount"))
                }),
            type number
        ),

    AddVoidedFlag =
        Table.AddColumn(
            AddTaxAmount,
            "voided_flag",
            each
                FirstNonNull({
                    AsLogical(FieldOrNull([_selection], "voided")),
                    AsLogical(FieldOrNull([_selection], "voidedFlag"))
                }),
            type logical
        ),

    AddRefundFlag =
        Table.AddColumn(
            AddVoidedFlag,
            "refund_flag",
            each
                let
                    explicitRefund =
                        FirstNonNull({
                            AsLogical(FieldOrNull([_selection], "refunded")),
                            AsLogical(FieldOrNull([_selection], "isRefund"))
                        }),
                    refundDetails =
                        FirstNonNull({
                            FieldOrNull([_selection], "refundDetails"),
                            FieldOrNull([_selection], "refundInfo")
                        })
                in
                    if explicitRefund <> null then explicitRefund else refundDetails <> null,
            type logical
        ),

    AddFulfillmentStatus =
        Table.AddColumn(
            AddRefundFlag,
            "fulfillment_status",
            each AsText(FieldOrNull([_selection], "fulfillmentStatus")),
            type text
        ),

    AddModifiedDate =
        Table.AddColumn(
            AddFulfillmentStatus,
            "modified_date_utc",
            each
                FirstNonNull({
                    ToDateTimeZone(FieldOrNull([_selection], "modifiedDate")),
                    ToDateTimeZone(FieldOrNull([_selection], "modifiedDateUtc")),
                    [_order_modified_date_utc]
                }),
            type datetimezone
        ),

    AddModifierCount =
        Table.AddColumn(
            AddModifiedDate,
            "modifier_count",
            each List.Count(GetListField([_selection], {"modifiers", "modifierSelections", "appliedModifiers"})),
            Int64.Type
        ),

    AddSelectionDiscountCount =
        Table.AddColumn(
            AddModifierCount,
            "selection_discount_count",
            each List.Count(GetListField([_selection], {"appliedDiscounts", "discounts"})),
            Int64.Type
        ),

    AddSelectionTaxCount =
        Table.AddColumn(
            AddSelectionDiscountCount,
            "selection_tax_count",
            each List.Count(GetListField([_selection], {"appliedTaxes", "taxes"})),
            Int64.Type
        ),

    Final =
        Table.SelectColumns(
            AddSelectionTaxCount,
            {
                "restaurant_guid",
                "selection_guid",
                "check_guid",
                "order_guid",
                "business_date",
                "menu_item_guid",
                "menu_item_name",
                "menu_group_guid",
                "menu_group_name",
                "sales_category_guid",
                "sales_category_name",
                "quantity",
                "pre_discount_price",
                "price",
                "tax_amount",
                "voided_flag",
                "refund_flag",
                "fulfillment_status",
                "modified_date_utc",
                "modifier_count",
                "selection_discount_count",
                "selection_tax_count",
                "ingested_at_utc",
                "record_source"
            }
        ),

    SortedRows =
        Table.Sort(
            Final,
            {
                {"ingested_at_utc", Order.Descending},
                {"selection_guid", Order.Ascending}
            }
        ),

    DedupedRows =
        Table.Distinct(
            SortedRows,
            {"restaurant_guid", "selection_guid"}
        )
in
    DedupedRows
```

## Validation Notes

- Row count checked: ☐
- Key fields populated: ☐
- Nulls / duplicates checked: ☐
- Reconciliation impact understood: ☐