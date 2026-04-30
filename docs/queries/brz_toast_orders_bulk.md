# brz_toast_orders_bulk

Artifact Type: Power Query M
Build Method: Query
Dependencies: Toast auth; restaurant GUID; orders:read scope
Grain: One row per raw order payload
Last Updated: April 27, 2026
Layer: Bronze
Notes: Uses modified startDate/endDate window.
Primary / Merge Key: payload_hash
Purpose: Pull raw order payloads from Toast API into Bronze.
Source Objects: Toast API /orders/v2/ordersBulk
Status: Documented
Target Object: brz_toast_orders_bulk

## Implementation Notes

**Code source:** uploaded query export from `brzqueries.md.txt`

**Credential handling:** do not paste live client secrets into this page. If adding code, replace any `clientSecret = "..."` value with `clientSecret = "<SECURE_STORE>"`.

## Paste Query Here

```
let
    clientId = "MPM8Uoze9yLWAF5PG2HDWcrhwXTuZ7uK",
    clientSecret = "<SECURE_STORE>",
    restaurantGuid = "4d5e20cd-c9f9-4280-a722-4bee7853e1b5",
    apiHost = "https://ws-api.toasttab.com",

    sourceSystem = "toast",
    sourceEndpoint = "/orders/v2/ordersBulk",
    recordSource = "ordersBulk",

    today = Date.From(DateTime.LocalNow()),
    businessDatesToFetch = List.Transform(
        {0, 1, 2},
        each Date.ToText(Date.AddDays(today, -_), "yyyyMMdd")
    ),

    pageSize = 100,
    maxPages = 50,

    ingestedAtUtc = DateTimeZone.ToText(DateTimeZone.UtcNow(), "yyyy-MM-ddTHH:mm:ss.fffZ"),

    authBody =
        Text.ToBinary(
            "{""clientId"":""" & clientId & """,""clientSecret"":""" & clientSecret & """,""userAccessType"":""TOAST_MACHINE_CLIENT""}"
        ),

    authResponse =
        Json.Document(
            Web.Contents(
                apiHost,
                [
                    RelativePath = "authentication/v1/authentication/login",
                    Headers = [#"Content-Type" = "application/json"],
                    Content = authBody
                ]
            )
        ),

    accessToken = authResponse[token][accessToken],
    tokenType = authResponse[token][tokenType],

    GetPageForDate = (bizDate as text, pageNumber as number) as list =>
        let
            response =
                Json.Document(
                    Web.Contents(
                        apiHost,
                        [
                            RelativePath = "orders/v2/ordersBulk",
                            Query = [
                                businessDate = bizDate,
                                pageSize = Text.From(pageSize),
                                page = Text.From(pageNumber)
                            ],
                            Headers = [
                                Authorization = tokenType & " " & accessToken,
                                #"Toast-Restaurant-External-ID" = restaurantGuid
                            ]
                        ]
                    )
                ),

            rows =
                if Type.Is(Value.Type(response), type list) then
                    response
                else
                    {response}
        in
            rows,

    GetAllPagesForDate = (bizDate as text) as list =>
        List.Combine(
            List.Generate(
                () => [Page = 1, Rows = GetPageForDate(bizDate, 1)],
                each List.Count([Rows]) > 0 and [Page] <= maxPages,
                each [Page = [Page] + 1, Rows = GetPageForDate(bizDate, [Page] + 1)],
                each [Rows]
            )
        ),

    AllRows = List.Combine(
        List.Transform(businessDatesToFetch, each GetAllPagesForDate(_))
    ),

    rawTable =
        if List.Count(AllRows) = 0 then
            #table({"payload"}, {})
        else
            Table.FromList(AllRows, Splitter.SplitByNothing(), {"payload"}),

    withPayloadJson =
        Table.AddColumn(
            rawTable,
            "payload_json",
            each Text.FromBinary(Json.FromValue([payload]), TextEncoding.Utf8),
            type text
        ),

    Fingerprint = (input as text) as text =>
        let
            len = Text.Length(input),
            headLen = if len < 400 then len else 400,
            tailLen = if len < 400 then 0 else 400,
            headText = Text.Start(input, headLen),
            tailText = if tailLen = 0 then "" else Text.End(input, tailLen),
            sampleText = headText & "|" & tailText & "|" & Text.From(len),
            chars = List.Transform(Text.ToList(sampleText), each Character.ToNumber(_)),
            state =
                List.Accumulate(
                    chars,
                    [h1 = 7, h2 = 11],
                    (s, c) => [
                        h1 = Number.Mod((s[h1] * 131) + c, 2147483647),
                        h2 = Number.Mod((s[h2] * 137) + c, 2147483629)
                    ]
                ),
            hashText = Text.From(state[h1]) & "-" & Text.From(state[h2]) & "-" & Text.From(len)
        in
            hashText,

    withPayloadHash =
        Table.AddColumn(
            withPayloadJson,
            "payload_hash",
            each Fingerprint([payload_json]),
            type text
        ),

    withIngestedAt =
        Table.AddColumn(
            withPayloadHash,
            "ingested_at_utc",
            each ingestedAtUtc,
            type text
        ),

    withSourceSystem =
        Table.AddColumn(
            withIngestedAt,
            "source_system",
            each sourceSystem,
            type text
        ),

    withSourceEndpoint =
        Table.AddColumn(
            withSourceSystem,
            "source_endpoint",
            each sourceEndpoint,
            type text
        ),

    withRestaurantGuid =
        Table.AddColumn(
            withSourceEndpoint,
            "restaurant_guid",
            each restaurantGuid,
            type text
        ),

    withRecordSource =
        Table.AddColumn(
            withRestaurantGuid,
            "record_source",
            each recordSource,
            type text
        ),

    withWindowStart =
        Table.AddColumn(
            withRecordSource,
            "extract_window_start_utc",
            each List.Min(businessDatesToFetch),
            type nullable text
        ),

    withWindowEnd =
        Table.AddColumn(
            withWindowStart,
            "extract_window_end_utc",
            each List.Max(businessDatesToFetch),
            type nullable text
        ),

    final =
        Table.SelectColumns(
            withWindowEnd,
            {
                "ingested_at_utc",
                "source_system",
                "source_endpoint",
                "restaurant_guid",
                "record_source",
                "extract_window_start_utc",
                "extract_window_end_utc",
                "payload_hash",
                "payload_json"
            }
        )
in
    final
```

## Validation Notes

- Row count checked: ☐
- Key fields populated: ☐
- Nulls / duplicates checked: ☐
- Reconciliation impact understood: ☐