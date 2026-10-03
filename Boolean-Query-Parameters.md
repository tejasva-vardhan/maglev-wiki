# Boolean Query Parameters

Shared query-parameter binding rules for boolean flags. Endpoint pages define the default for each flag; a default applies only when the parameter is omitted.

## Legacy Java behavior

The API actions bind supplied flags through primitive `boolean` setters. For example, `includeReferences` starts as `true` in [`ApiActionSupport.java#L51`](https://github.com/OneBusAway/onebusaway-application-modules/blob/fed9494e4b3ff345efabcd17b9b21fbf2a9dab7e/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ApiActionSupport.java#L51) and is assigned by [`setIncludeReferences` (#L85–L86)](https://github.com/OneBusAway/onebusaway-application-modules/blob/fed9494e4b3ff345efabcd17b9b21fbf2a9dab7e/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ApiActionSupport.java#L85-L86). `includePolylines` likewise starts as `true` and has a primitive setter in [`StopsForRouteAction.java#L59–L75`](https://github.com/OneBusAway/onebusaway-application-modules/blob/fed9494e4b3ff345efabcd17b9b21fbf2a9dab7e/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForRouteAction.java#L59-L75).

The reference checkout uses Struts 2.5.33. Its [`XWorkBasicConverter.doConvertToBoolean`](https://github.com/apache/struts/blob/STRUTS_2_5_33/core/src/main/java/com/opensymphony/xwork2/conversion/impl/XWorkBasicConverter.java#L159-L165) calls `Boolean.valueOf` for a string: case-insensitive `true` means true, and every other string means false, including the empty string. Omission leaves the action's initialized default unchanged; a supplied empty value overrides it with false.

| Request form | Legacy value |
|---|---|
| Flag omitted | Endpoint default |
| `?flag=true` (any letter case) | true |
| `?flag=false` (any letter case) | false |
| `?flag=` or bare `?flag` | false |
| `?flag=abc`, `?flag=1`, or `?flag=yes` | false, without a boolean field error |

These statements are based on source inspection; no new live Java parity test was run for this documentation update.

## Implementation Decisions

**Maglev rejects malformed non-empty boolean values instead of silently coercing them to false.**

Maglev's shared `utils.ParseBoolParam` accepts case-insensitive `true` and `false`, and reports the field error `must be a boolean value (true/false)` for other non-empty strings. In particular, Go's additional spellings `1`, `0`, `t`, and `f` are not accepted. Handlers return HTTP 400 when this validation is reached; endpoint-specific lookup/validation ordering still applies. See [[trip-details]] and [[trip-for-vehicle]] for their existing decisions about that ordering.

This preserves Maglev's strict-input policy: a typo should be visible to the caller rather than silently disabling an option. If reconsidered, Java's fallback is to read every non-`true` supplied string as false. The broader validation work is tracked in [#1499](https://github.com/OneBusAway/maglev/issues/1499).

**PR #1511 restores Java-compatible empty-versus-omitted semantics.**

[PR #1511](https://github.com/OneBusAway/maglev/pull/1511), reviewed at [`279fe8ef`](https://github.com/OneBusAway/maglev/commit/279fe8ef1354c3571721e7b9cd8c47be0e2eb29f), changes the shared parser so `?flag=` and bare `?flag` return false, even when the endpoint default is true. Omitted flags keep their endpoint defaults. This is parity restoration, not a new deviation. Before this PR, the shared parser used the fallback for empty values; some trip handlers had local workarounds to make empty flags false already.

The rule applies to flags routed through `ParseBoolParam` / `ShouldIncludeReferences`: `includeReferences` on reference-enabled handlers; `includePolylines` on [[stops-for-route]]; and `includeTrip`, `includeSchedule`, and `includeStatus` on [[trip-details]], [[trip-for-vehicle]], [[trips-for-route]], and [[trips-for-location]]. The false-default `userOnVehicle` flag on report-problem-with-trip is unchanged by the empty-value fix. This does not add flag support to endpoints that do not parse those flags.

For example, `stops-for-route?includePolylines=` returns empty route-level and direction-group polyline arrays; `includeReferences=` suppresses reference contents. Response shapes and omitted-parameter defaults are unchanged.

### Client impact

Normal typed JS SDK and Android boolean calls serialize explicit true/false; inspected iOS URL builders omit these flags. No current UI regression was found. The JS SDK's generic serializer emits `null` as an empty value, so untyped/manual null or empty-string calls now disable an option instead of requesting its default. Wayfinder's trip-details and trips-for-route proxies replace empty flag strings with `true`, so those proxies do not expose the new direct-API semantics. Wayfinder was inspected on its local `develop` branch without automatically updating it.
