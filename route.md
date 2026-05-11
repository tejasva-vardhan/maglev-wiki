# `GET /api/where/route/{id}`

## Goal in Context

A rider's client retrieves the descriptive record for a single transit route by its combined agency-and-route ID, so it can display the route's name, type, colour branding, and link to the agency's published timetable.

## Scope

OneBusAway REST API.

## Level

User goal.

## Primary Actor

Rider.

## Stakeholders and Interests

- **Rider** — wants accurate display information for a known route: short name, long name, vehicle/mode type, colour scheme, optional description, and a URL to the agency's published timetable.

## Preconditions

- The server has loaded a valid GTFS feed containing route records.
- The caller supplies a valid API key.
- The route ID is in the combined `{agencyId}_{entityId}` form (e.g., `1_102718`).

## Minimal Guarantees

- The response is well-formed JSON with `code`, `text`, `version`, and `currentTime` fields whenever the request is processed normally. (See Suspected Defects for the malformed-ID edge case, where this guarantee does not hold.)
- On failure the `data` field is absent from the response body.

## Success Guarantees

- `data.entry` contains the complete route record sourced from the GTFS feed.
- `data.references.agencies` contains the full record for the route's owning agency, unless references are suppressed.

## Trigger

The client sends `GET /api/where/route/{id}.json?key={key}`.

## Main Success Scenario

1. The client sends `GET /api/where/route/{id}.json?key={key}`, where `{id}` is the route's combined `{agencyId}_{entityId}` identifier.
2. The system validates the API key ([`ApiKeyInterceptor.isAllowed()` lines 76–84](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ApiKeyInterceptor.java#L76-L84)).
3. The system parses the combined ID, splitting on the first underscore, and looks up the route record by the resulting agency ID and entity ID ([`RouteAction.show()` line 59](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/RouteAction.java#L59)).
4. The system returns HTTP 200. `data.entry` contains the route record. `data.references.agencies` contains the owning agency (see Response Structure).

## Extensions

- **2a. API key is absent** — The system returns HTTP 401. `code` is `401`, `text` is `"permission denied"`.
- **2b. API key has exceeded its rate limit** — The system returns HTTP 429. `code` is `429`, `text` is `"rate limit exceeded"`.
- **3a. Route ID lacks an underscore** — See Suspected Defects; the Java implementation returns HTTP 200 with a JSON body of `null` rather than an error. The Go implementation should return HTTP 400.
- **3b. Route ID contains an underscore but no matching route exists** — ([`RouteAction.show()` lines 61–62](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/RouteAction.java#L61-L62)) The system returns HTTP 404. `code` is `404`, `text` is `"resource not found"`. The `data` field is absent.

## Suspected Defects

### Defects that affect the use case

- **`RouteAction`** — [`line 59`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/RouteAction.java#L59): `show()` calls `_service.getRouteForId(_id)` without catching the `IllegalStateException` that [`AgencyAndIdLibrary.convertFromString` throws at line 48](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-util/src/main/java/org/onebusaway/util/AgencyAndIdLibrary.java#L48) when the ID contains no underscore. Due to how the Struts2 interceptor stack processes the uncaught exception, the server returns HTTP 200 with a JSON body of `null` rather than any recognisable error envelope. The correct behaviour — catching the exception and returning a structured error — is demonstrated by `RouteDetailsAction` ([`lines 86–89`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/RouteDetailsAction.java#L86-L89)), which catches `IllegalStateException` explicitly and calls `setExceptionResponse()`. The Go implementation should return HTTP 400 (invalid argument) when the supplied ID contains no underscore.

### Implementation defects only

- **`RouteV2Bean`** — [`lines 62–66`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/RouteV2Bean.java#L62-L66): The public method `getNullSafeShortName()` — which returns `shortName` when non-null, otherwise returns the route `id` — is automatically serialised to JSON as `nullSafeShortName` because the Java JSON serialiser enumerates all public getter methods on the bean. This field was not deliberately added to the API contract; it was intended as an internal display helper. When `shortName` is non-null (as it is in all observed KCM data), the field is always identical to `shortName`. The Go implementation should decide whether to emit `nullSafeShortName` for backwards compatibility.

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Path parameter — the route's combined agencyId_entityId identifier"
    },
    "key": {
      "type": "string"
    },
    "version": {
      "type": "integer",
      "default": 2
    },
    "includeReferences": {
      "type": "boolean",
      "default": true
    }
  }
}
```

**`id`** — The route's combined identifier in `{agencyId}_{entityId}` form (e.g., `1_102718`). The agency portion is everything before the first underscore; the entity portion is everything after. Entity IDs may themselves contain underscores (e.g., `KCM_40_100479` → agency `KCM`, entity `40_100479`).

**`key`** — API authentication key. Required on every request; omitting it returns HTTP 401.

**`version`** — Selects the response format version. Defaults to 2. Only version 2 is in scope for the Go rewrite.

**`includeReferences`** — When `true` (the default), the owning agency record is included in `data.references.agencies`. When `false`, `data.references.agencies` is an empty array and no agency data is returned ([`BeanFactoryV2` line 1624](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1624)).

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "code":        { "type": "integer" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "text":        { "type": "string" },
    "version":     { "type": "integer" },
    "data":        { "type": "object" }
  }
}
```

**`code`** — HTTP-equivalent status code mirrored in the body: `200` on success, `404` when the route is not found.

**`currentTime`** — The server's current time at the moment the response was generated, in Unix milliseconds.

**`text`** — Human-readable status string: `"OK"` on success, `"resource not found"` on 404.

**`version`** — Response format version. Always `2` when the default is in effect.

**`data`** — Present only on a 200 response; absent on 404 and other errors.

### data

```json
{
  "type": "object",
  "properties": {
    "entry":      { "type": "object" },
    "references": { "type": "object" }
  }
}
```

**`data.entry`** — The route record (see below).

**`data.references`** — Contains the owning agency in `agencies` when `includeReferences` is `true`. All other arrays (`routes`, `stops`, `trips`, `situations`, `stopTimes`) are always empty for this endpoint.

### data.entry

```json
{
  "type": "object",
  "required": ["id", "type", "agencyId"],
  "properties": {
    "id":               { "type": "string" },
    "shortName":        { "type": "string" },
    "longName":         { "type": "string" },
    "description":      { "type": "string" },
    "type":             { "type": "integer" },
    "url":              { "type": "string" },
    "color":            { "type": "string" },
    "textColor":        { "type": "string" },
    "agencyId":         { "type": "string" },
    "nullSafeShortName": { "type": "string" }
  }
}
```

**`data.entry.id`** — The route's combined `{agencyId}_{entityId}` identifier (e.g., `"1_102718"`).

**`data.entry.shortName`** — The route's short public identifier, typically the number or code shown on vehicles and signs (e.g., `"44"` or `"South Lake Union Streetcar"`). Optional per GTFS; may be an empty string or absent when only `longName` is provided.

**`data.entry.longName`** — The route's full descriptive name (e.g., `"Ballard - Montlake"`). Optional per GTFS; may be an empty string or absent when only `shortName` is provided. At least one of `shortName` or `longName` is present in valid GTFS data, but care is needed when constructing a display name because agencies vary in which fields they populate.

**`data.entry.description`** — A human-readable description of the route, often summarising the trip pattern (e.g., `"Auburn Station - White River Junction"`). Optional; may be an empty string.

**`data.entry.type`** — The GTFS route type as an integer. Common values: `0` = tram/light rail, `1` = subway/metro, `2` = rail, `3` = bus, `4` = ferry, `11` = trolleybus, `12` = monorail.

**`data.entry.url`** — URL to the agency's published timetable or information page for this route. Optional; may be an empty string.

**`data.entry.color`** — The route's background colour as a six-digit uppercase hex string without a leading `#` (e.g., `"FDB71A"`). Optional; may be an empty string if not specified in the GTFS feed.

**`data.entry.textColor`** — The route's foreground text colour as a six-digit uppercase hex string without a leading `#`, intended for legible text rendered on top of `color`. Optional; may be an empty string if not specified in the GTFS feed.

**`data.entry.agencyId`** — The plain agency identifier for the route's owning agency (e.g., `"1"`). This is the raw `agency_id` from the GTFS feed and is **not** in the combined `{agencyId}_{entityId}` form ([`BeanFactoryV2` line 435](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L435)).

**`data.entry.nullSafeShortName`** — A derived field: equal to `shortName` when that field is non-null, otherwise equal to `id`. Emitted automatically by the Java serialiser from a public getter method on the route bean. See the Suspected Defects section. When `shortName` is non-null this field duplicates `shortName` exactly.

### data.references.agencies[]

```json
{
  "type": "object",
  "properties": {
    "id":             { "type": "string" },
    "name":           { "type": "string" },
    "url":            { "type": "string" },
    "timezone":       { "type": "string" },
    "lang":           { "type": "string" },
    "phone":          { "type": "string" },
    "disclaimer":     { "type": "string" },
    "email":          { "type": "string" },
    "fareUrl":        { "type": "string" },
    "privateService": { "type": "boolean" }
  }
}
```

Contains one entry: the owning agency for the requested route. Present when `includeReferences` is `true`; empty array when `false`. The fields are identical to those returned by `GET /api/where/agency/{id}` — see that endpoint's specification for field descriptions.

**`data.references.agencies[].id`** — Plain agency identifier (e.g., `"1"`). Not in combined form.
