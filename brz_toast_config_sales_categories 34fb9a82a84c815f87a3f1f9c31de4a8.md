# stg_toast_payment

Artifact Type: Power Query M
Build Method: Query
Dependencies: brz_toast_payments
Grain: One row per payment
Last Updated: April 27, 2026
Layer: Silver
Notes: Payments modeled separately from orders.
Primary / Merge Key: restaurant_guid + payment_guid
Purpose: Normalize Toast payments into one row per payment.
Source Objects: brz_toast_payments
Status: Documented
Target Object: stg_toast_payment

## Implementation Notes

**Code source:** uploaded query export from `stgqueries.md.txt`

**Credential handling:** do not paste live client secrets into this page. If adding code, replace any `clientSecret = "..."` value with `clientSecret = "<SECURE_STORE>"`.

## Paste Query Here

```
let
    Source = Lakehouse.Contents([CreateNavigationProperties = false, EnableFolding = false, HierarchicalNavigation = null, PreserveOrder = null]),
    Navigation_1 = Source{[workspaceId = "9a10c019-6b70-4cb0-8fa7-50938b6991ce"]}[Data],
    Navigation_2 = Navigation_1{[lakehouseId = "61fc61d6-f409-4c3d-b460-3162197a75ad"]}[Data],
    BronzePayments = Navigation_2{[Id = "brz_toast_payments", ItemKind = "Table"]}?[Data]?,

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

    ToBusinessDate = (x as any) as nullable date =>
        let
            s = AsText(x),
            padded = if s = null then null else Text.PadStart(s, 8, "0"),
            yyyy = if padded = null then null else Text.Start(padded, 4),
            mm = if padded = null then null else Text.Range(padded, 4, 2),
            dd = if padded = null then null else Text.Range(padded, 6, 2)
        in
            try if padded = null then null else #date(Number.FromText(yyyy), Number.FromText(mm), Number.FromText(dd)) otherwise null,

    WithPaymentRecord =
        Table.AddColumn(
            BronzePayments,
            "_payment",
            each try Json.Document(Text.ToBinary([payload_json], TextEncoding.Utf8)) otherwise null,
            type any
        ),

    KeepValidPayments =
        Table.SelectRows(
            WithPaymentRecord,
            each [_payment] <> null and Value.Is([_payment], type record)
        ),

    AddPaymentGuid =
        Table.AddColumn(
            KeepValidPayments,
            "payment_guid",
            each AsText(FieldOrNull([_payment], "guid")),
            type text
        ),

    KeepPaymentsOnly =
        Table.SelectRows(AddPaymentGuid, each [payment_guid] <> null),

    AddOrderGuid =
        Table.AddColumn(
            KeepPaymentsOnly,
            "order_guid",
            each AsText(FieldOrNull([_payment], "orderGuid")),
            type text
        ),

    AddCheckGuid =
        Table.AddColumn(
            AddOrderGuid,
            "check_guid",
            each AsText(FieldOrNull([_payment], "checkGuid")),
            type text
        ),

    AddPaidBusinessDate =
        Table.AddColumn(
            AddCheckGuid,
            "paid_business_date",
            each ToBusinessDate(FieldOrNull([_payment], "paidBusinessDate")),
            type date
        ),

    AddVoidBusinessDate =
        Table.AddColumn(
            AddPaidBusinessDate,
            "void_business_date",
            each
                ToBusinessDate(
                    FirstNonNull({
                        FieldOrNull([_payment], "voidBusinessDate"),
                        FieldOrNull(FieldOrNull([_payment], "voidInfo"), "businessDate"),
                        FieldOrNull(FieldOrNull([_payment], "voidInfo"), "voidBusinessDate")
                    })
                ),
            type date
        ),

    AddRefundBusinessDate =
        Table.AddColumn(
            AddVoidBusinessDate,
            "refund_business_date",
            each
                ToBusinessDate(
                    FirstNonNull({
                        FieldOrNull([_payment], "refundBusinessDate"),
                        FieldOrNull(FieldOrNull([_payment], "refund"), "businessDate"),
                        FieldOrNull(FieldOrNull([_payment], "refund"), "refundBusinessDate")
                    })
                ),
            type date
        ),

    AddPaidDateUtc =
        Table.AddColumn(
            AddRefundBusinessDate,
            "paid_date_utc",
            each AsText(FieldOrNull([_payment], "paidDate")),
            type text
        ),

    AddAmount =
        Table.AddColumn(
            AddPaidDateUtc,
            "amount",
            each AsNumber(FieldOrNull([_payment], "amount")),
            type number
        ),

    AddTipAmount =
        Table.AddColumn(
            AddAmount,
            "tip_amount",
            each AsNumber(FieldOrNull([_payment], "tipAmount")),
            type number
        ),

    AddPaymentType =
        Table.AddColumn(
            AddTipAmount,
            "payment_type",
            each AsText(FieldOrNull([_payment], "type")),
            type text
        ),

    AddServerGuid =
        Table.AddColumn(
            AddPaymentType,
            "server_guid",
            each GuidFromRecord(FieldOrNull([_payment], "server")),
            type text
        ),

    AddCashDrawerGuid =
        Table.AddColumn(
            AddServerGuid,
            "cash_drawer_guid",
            each GuidFromRecord(FieldOrNull([_payment], "cashDrawer")),
            type text
        ),

    AddCardPaymentId =
        Table.AddColumn(
            AddCashDrawerGuid,
            "card_payment_id",
            each
                FirstNonNull({
                    AsText(FieldOrNull([_payment], "cardPaymentId")),
                    GuidFromRecord(FieldOrNull([_payment], "cardPayment"))
                }),
            type text
        ),

    AddTenderTransactionGuid =
        Table.AddColumn(
            AddCardPaymentId,
            "tender_transaction_guid",
            each AsText(FieldOrNull([_payment], "tenderTransactionGuid")),
            type text
        ),

    AddHouseAccountReference =
        Table.AddColumn(
            AddTenderTransactionGuid,
            "house_account_reference",
            each
                let
                    h = FieldOrNull([_payment], "houseAccount")
                in
                    FirstNonNull({
                        GuidFromRecord(h),
                        AsText(FieldOrNull(h, "externalId")),
                        AsText(FieldOrNull(h, "accountNumber")),
                        AsText(FieldOrNull(h, "name"))
                    }),
            type text
        ),

    AddOtherPaymentReference =
        Table.AddColumn(
            AddHouseAccountReference,
            "other_payment_reference",
            each
                let
                    o = FieldOrNull([_payment], "otherPayment")
                in
                    FirstNonNull({
                        GuidFromRecord(o),
                        AsText(FieldOrNull(o, "externalId")),
                        AsText(FieldOrNull(o, "name"))
                    }),
            type text
        ),

    AddModifiedDateUtc =
        Table.AddColumn(
            AddOtherPaymentReference,
            "modified_date_utc",
            each
                FirstNonNull({
                    AsText(FieldOrNull([_payment], "modifiedDate")),
                    AsText(FieldOrNull(FieldOrNull([_payment], "voidInfo"), "modifiedDate")),
                    AsText(FieldOrNull(FieldOrNull([_payment], "refund"), "modifiedDate")),
                    AsText(FieldOrNull([_payment], "paidDate"))
                }),
            type text
        ),

    FinalColumns =
        Table.SelectColumns(
            AddModifiedDateUtc,
            {
                "restaurant_guid",
                "payment_guid",
                "order_guid",
                "check_guid",
                "paid_business_date",
                "void_business_date",
                "refund_business_date",
                "paid_date_utc",
                "amount",
                "tip_amount",
                "payment_type",
                "server_guid",
                "cash_drawer_guid",
                "card_payment_id",
                "tender_transaction_guid",
                "house_account_reference",
                "other_payment_reference",
                "modified_date_utc",
                "ingested_at_utc"
            }
        ),

    Sorted =
        Table.Sort(
            FinalColumns,
            {
                {"modified_date_utc", Order.Descending},
                {"ingested_at_utc", Order.Descending}
            }
        ),

    Buffered = Table.Buffer(Sorted),

    Deduped =
        Table.Distinct(
            Buffered,
            {"restaurant_guid", "payment_guid"}
        )
in
    Deduped
```

## Validation Notes

- Row count checked: ☐
- Key fields populated: ☐
- Nulls / duplicates checked: ☐
- Reconciliation impact understood: ☐