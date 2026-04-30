# stg_toast_check_service_charge

Artifact Type: Power Query M
Build Method: Query
Dependencies: brz_toast_orders_bulk; brz_toast_config_service_charges
Grain: One row per applied check service charge
Last Updated: April 27, 2026
Layer: Silver
Notes: Uses config fallback for gratuity/taxable/destination.
Primary / Merge Key: restaurant_guid + check_guid + service_charge_guid + ordinal
Purpose: Normalize applied check service charges.
Source Objects: brz_toast_orders_bulk; brz_toast_config_service_charges
Status: Documented
Target Object: stg_toast_check_service_charge

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
    DimServiceCharge = Navigation_2{[Id = "brz_toast_config_service_charges", ItemKind = "Table"]}?[Data]?,

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

    ServiceChargeTableFromList = (svc as list) as table =>
        let
            positions = List.Positions(svc),
            paired = List.Zip({positions, svc}),
            tbl =
                if List.Count(paired) = 0 then
                    #table(type table [ordinal = Int64.Type, _service_charge = any], {})
                else
                    Table.FromRows(paired, type table [ordinal = Int64.Type, _service_charge = any])
        in
            tbl,

    // ---------- Service charge config dimension ----------
    WithSvcConfigRecord =
        Table.AddColumn(
            DimServiceCharge,
            "_svc_cfg",
            each try Json.Document(Text.ToBinary([payload_json], TextEncoding.Utf8)) otherwise null,
            type any
        ),

    KeepValidSvcConfig =
        Table.SelectRows(
            WithSvcConfigRecord,
            each [_svc_cfg] <> null and Value.Is([_svc_cfg], type record)
        ),

    SvcConfigProjected =
        Table.FromRecords(
            List.Transform(
                Table.ToRecords(KeepValidSvcConfig),
                each [
                    service_charge_guid = AsText(FieldOrNull([_svc_cfg], "guid")),
                    cfg_gratuity_flag = ToLogical(FieldOrNull([_svc_cfg], "gratuity")),
                    cfg_taxable_flag = ToLogical(FieldOrNull([_svc_cfg], "taxable")),
                    cfg_destination = FirstNonNull({
                        AsText(FieldOrNull([_svc_cfg], "destination")),
                        AsText(FieldOrNull(FieldOrNull([_svc_cfg], "destination"), "type")),
                        AsText(FieldOrNull(FieldOrNull([_svc_cfg], "destination"), "name"))
                    })
                ]
            )
        ),

    SvcConfigDistinct =
        Table.Distinct(SvcConfigProjected, {"service_charge_guid"}),

    // ---------- Orders payload ----------
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

    AddServiceChargeList =
        Table.AddColumn(
            KeepChecksOnly,
            "_service_charge_list",
            each GetListField([_check], {"appliedServiceCharges", "serviceCharges"}),
            type list
        ),

    AddServiceChargeTable =
        Table.AddColumn(
            AddServiceChargeList,
            "_service_charge_table",
            each ServiceChargeTableFromList([_service_charge_list]),
            type table [ordinal = Int64.Type, _service_charge = any]
        ),

    ExpandServiceChargeTable =
        Table.ExpandTableColumn(
            AddServiceChargeTable,
            "_service_charge_table",
            {"ordinal", "_service_charge"},
            {"ordinal", "_service_charge"}
        ),

    KeepServiceChargeRecords =
        Table.SelectRows(
            ExpandServiceChargeTable,
            each [_service_charge] <> null and Value.Is([_service_charge], type record)
        ),

    AddServiceChargeGuid =
        Table.AddColumn(
            KeepServiceChargeRecords,
            "service_charge_guid",
            each
                FirstNonNull({
                    GuidFromRecord(FieldOrNull([_service_charge], "serviceCharge")),
                    GuidFromRecord(FieldOrNull([_service_charge], "serviceChargeType")),
                    GuidFromRecord([_service_charge]),
                    AsText(FieldOrNull([_service_charge], "guid"))
                }),
            type text
        ),

    AddAmount =
        Table.AddColumn(
            AddServiceChargeGuid,
            "amount",
            each
                FirstNonNull({
                    AsNumber(FieldOrNull([_service_charge], "chargeAmount")),
                    AsNumber(FieldOrNull([_service_charge], "amount")),
                    AsNumber(FieldOrNull([_service_charge], "price"))
                }),
            type number
        ),

    AddRawGratuityFlag =
        Table.AddColumn(
            AddAmount,
            "_raw_gratuity_flag",
            each
                FirstNonNull({
                    ToLogical(FieldOrNull([_service_charge], "gratuity")),
                    ToLogical(FieldOrNull([_service_charge], "isGratuity"))
                }),
            type logical
        ),

    AddRawTaxableFlag =
        Table.AddColumn(
            AddRawGratuityFlag,
            "_raw_taxable_flag",
            each
                FirstNonNull({
                    ToLogical(FieldOrNull([_service_charge], "taxable")),
                    ToLogical(FieldOrNull([_service_charge], "isTaxable"))
                }),
            type logical
        ),

    AddRawDestination =
        Table.AddColumn(
            AddRawTaxableFlag,
            "_raw_destination",
            each
                FirstNonNull({
                    AsText(FieldOrNull([_service_charge], "destination")),
                    AsText(FieldOrNull(FieldOrNull([_service_charge], "destination"), "type")),
                    AsText(FieldOrNull(FieldOrNull([_service_charge], "destination"), "name"))
                }),
            type text
        ),

    JoinSvcConfig =
        Table.NestedJoin(
            AddRawDestination,
            {"service_charge_guid"},
            SvcConfigDistinct,
            {"service_charge_guid"},
            "_svc_dim",
            JoinKind.LeftOuter
        ),

    ExpandSvcConfig =
        Table.ExpandTableColumn(
            JoinSvcConfig,
            "_svc_dim",
            {"cfg_gratuity_flag", "cfg_taxable_flag", "cfg_destination"},
            {"cfg_gratuity_flag", "cfg_taxable_flag", "cfg_destination"}
        ),

    AddFinalGratuityFlag =
        Table.AddColumn(
            ExpandSvcConfig,
            "gratuity_flag",
            each FirstNonNull({[_raw_gratuity_flag], [cfg_gratuity_flag]}),
            type logical
        ),

    AddFinalTaxableFlag =
        Table.AddColumn(
            AddFinalGratuityFlag,
            "taxable_flag",
            each FirstNonNull({[_raw_taxable_flag], [cfg_taxable_flag]}),
            type logical
        ),

    AddFinalDestination =
        Table.AddColumn(
            AddFinalTaxableFlag,
            "destination",
            each FirstNonNull({[_raw_destination], [cfg_destination]}),
            type text
        ),

    Final =
        Table.SelectColumns(
            AddFinalDestination,
            {
                "restaurant_guid",
                "check_guid",
                "order_guid",
                "service_charge_guid",
                "amount",
                "gratuity_flag",
                "taxable_flag",
                "destination",
                "business_date",
                "ordinal"
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