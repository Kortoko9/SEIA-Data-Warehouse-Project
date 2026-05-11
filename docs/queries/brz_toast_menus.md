# brz_toast_menus

Artifact Type: Power Query M
Build Method: Query
Dependencies: Toast auth; restaurant GUID; menus:read scope
Grain: One row per full menu payload
Last Updated: April 27, 2026
Layer: Bronze
Notes: Full menu JSON source for menu/item/modifier dimensions.
Primary / Merge Key: payload_hash
Purpose: Pull full menu payload into Bronze.
Source Objects: Toast API /menus/v2/menus
Status: Documented
Target Object: brz_toast_menus

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
    sourceEndpoint = "/menus/v2/menus",
    recordSource = "menus_api",
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
                    RelativePath = "menus/v2/menus",
                    Headers = [
                        Authorization = tokenType & " " & accessToken,
                        #"Toast-Restaurant-External-ID" = restaurantGuid
                    ]
                ]
            )
        ),

    payloadJson = Text.FromBinary(Json.FromValue(response), TextEncoding.Utf8),

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

    payloadHash = Fingerprint(payloadJson),

    final =
        #table(
            type table [
                ingested_at_utc = text,
                source_system = text,
                source_endpoint = text,
                restaurant_guid = text,
                record_source = text,
                extract_window_start_utc = nullable text,
                extract_window_end_utc = nullable text,
                payload_hash = text,
                payload_json = text
            ],
            {
                {
                    ingestedAtUtc,
                    sourceSystem,
                    sourceEndpoint,
                    restaurantGuid,
                    recordSource,
                    null,
                    null,
                    payloadHash,
                    payloadJson
                }
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