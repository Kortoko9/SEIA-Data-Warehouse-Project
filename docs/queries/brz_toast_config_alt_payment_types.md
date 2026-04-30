# brz_toast_config_alt_payment_types

Artifact Type: Power Query M
Build Method: Query
Dependencies: Toast auth; restaurant GUID; config:read scope
Grain: One row per raw config payload/object
Last Updated: April 27, 2026
Layer: Bronze
Notes: Used for tender/payment classification.
Primary / Merge Key: payload_hash
Purpose: Pull alternate payment type config into Bronze.
Source Objects: Toast API /config/v2/alternatePaymentTypes
Status: Documented
Target Object: brz_toast_config_alt_payment_types

## Implementation Notes

**Code source:** uploaded query export from `brzqueries.md.txt`

**Credential handling:** do not paste live client secrets into this page. If adding code, replace any `clientSecret = "..."` value with `clientSecret = "<SECURE_STORE>"`.

## Paste Query Here

```
let
    clientId = "MPM8Uoze9yLWAF5PG2HDWcrhwXTuZ7uK",
    clientSecret = "rrhKfZIdCbmaxRO7RfAc1xHHeZbNVCZIfmPxsrmI6z2V-g9wL8MTfyd2XzOtwR3K",
    restaurantGuid = "4d5e20cd-c9f9-4280-a722-4bee7853e1b5",
    apiHost = "https://ws-api.toasttab.com",
    sourceSystem = "toast",
    sourceEndpoint = "/config/v2/alternatePaymentTypes",
    recordSource = "config_alt_payment_types_api",
    ingestedAtUtc = DateTimeZone.ToText(DateTimeZone.UtcNow(), "yyyy-MM-ddTHH:mm:ss.fffZ"),

    authBody = Text.ToBinary(
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

    response =
        Json.Document(
            Web.Contents(
                apiHost,
                [
                    RelativePath = "config/v2/alternatePaymentTypes",
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
            {response},

    rawTable = Table.FromList(rows, Splitter.SplitByNothing(), {"payload"}),

    withPayloadJson =
        Table.AddColumn(
            rawTable,
            "payload_json",
            each Text.FromBinary(Json.FromValue([payload]), TextEncoding.Utf8),
            type text
        ),

    HashText = (input as text) as text =>
        let
            chars = List.Transform(Text.ToList(input), each Character.ToNumber(_)),
            state =
                List.Accumulate(
                    chars,
                    [h1 = 7, h2 = 11],
                    (s, c) => [
                        h1 = Number.Mod((s[h1] * 131) + c, 2147483647),
                        h2 = Number.Mod((s[h2] * 137) + c, 2147483629)
                    ]
                ),
            hashText = Text.From(state[h1]) & "-" & Text.From(state[h2])
        in
            hashText,

    withPayloadHash =
        Table.AddColumn(
            withPayloadJson,
            "payload_hash",
            each HashText([payload_json]),
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
            each null,
            type nullable text
        ),

    withWindowEnd =
        Table.AddColumn(
            withWindowStart,
            "extract_window_end_utc",
            each null,
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