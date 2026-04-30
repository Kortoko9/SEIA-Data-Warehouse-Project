# brz_toast_restaurants

Artifact Type: Power Query M
Build Method: Query
Dependencies: Toast auth; restaurant GUID; restaurants:read scope
Grain: One row per restaurant metadata payload
Last Updated: April 27, 2026
Layer: Bronze
Notes: Contains location metadata such as closeout/timezone when present.
Primary / Merge Key: payload_hash
Purpose: Pull restaurant metadata into Bronze.
Source Objects: Toast API /restaurants/v1/restaurants/{restaurantGUID}
Status: Documented
Target Object: brz_toast_restaurants

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
    sourceEndpoint = "/restaurants/v1/restaurants/" & restaurantGuid,
    recordSource = "restaurants_api",
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
                    RelativePath = "restaurants/v1/restaurants/" & restaurantGuid,
                    Headers = [
                        Authorization = tokenType & " " & accessToken,
                        #"Toast-Restaurant-External-ID" = restaurantGuid
                    ]
                ]
            )
        ),

    payloadJson = Text.FromBinary(Json.FromValue(response), TextEncoding.Utf8),

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

    payloadHash = HashText(payloadJson),

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