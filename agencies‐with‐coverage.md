** DO NOT IMPLEMENT - DRAFT **

## Goal in Context

A rider's client application needs to discover which transit agencies are present in this server instance and the geographic extent of each agency's service network, so that it can initialise a map viewport, populate an agency picker, and know which agencies to query for real-time data.

## Scope

OBA REST API — `GET /api/where/agencies-with-coverage.json`

## Level

User goal

## Primary Actor

Rider (via client application)

## Stakeholders and Interests

**Rider** — wants a list of agencies the server knows about, each with enough geographic information to position a map view over the service area without having to query individual routes or stops first.

## Preconditions

- The caller holds a valid API key.
- The server has completed loading a transit data bundle (at least one agency is present in the graph).

## Minimal Guarantees

- Any response, including error responses, carries a JSON envelope with `code` and `currentTime`.

## Success Guarantees

- The response lists every agency present in the transit data graph.
- Each entry includes the centre point and bounding-box dimensions of that agency's service area, computed from the coordinates of all stops appearing in scheduled trips for that agency.
- When references are requested, full agency detail objects are included in the references block.

## Trigger

The client sends `GET /api/where/agencies-with-coverage.json?key=<key>`.

## Main Success Scenario

1. The client sends a GET request with a valid API key.
2. The server retrieves all agencies registered in the transit data graph.
3. For each agency the server constructs a geographic bounding box by iterating every stop that appears in a scheduled stop time for any trip belonging to any route of that agency.
4. From the bounding box the server derives a centre point (arithmetic midpoint of the min/max latitude and longitude, **not** the geographic centroid of all stop positions) and the full angular extent of the box.
5. The server assembles one coverage entry per agency — the agency ID plus the derived centre and dimensions — and, when references are enabled, emits full agency detail objects in the references block.
6. The server returns HTTP 200 with the list.

## Extensions

**1a. Missing or invalid API key:**
- The request is rejected before reaching the action; the server returns HTTP 401 with no response body.

**2a. Unknown `version` value:**
- The action returns HTTP 500 with message `"unknown version: <n>"`. The response envelope echoes back the requested version number.

**2b. Transit graph contains an agency ID with no corresponding agency record (data inconsistency):**
- The service layer throws an internal error; the exception interceptor returns HTTP 500. The response envelope's `version` field is hardcoded to `1` regardless of the version the caller specified. (See Suspected Defects.)

## Suspected Defects

### Defects that affect the use case

**`maxCount` parameter is accepted but never applied**
`AgenciesWithCoverageAction.java:62` — The action declares a `setMaxCount` setter, which the framework wires to the `maxCount` query parameter, and stores the value in a `MaxCountSupport` instance. However, when constructing the response the action calls `getBeanFactoryV2()` rather than `getBeanFactoryV2(_maxCount)`, so the count limit is never passed to the response factory. The factory's internal `_maxCount` field remains `null`, causing `filter()` to return the full list unconditionally. `limitExceeded` is therefore always `false` regardless of the value of `maxCount`. The Go implementor must decide whether to honour `maxCount` correctly or drop the parameter.

### Implementation defects only

**Exception responses hard-code version 1**
`ExceptionInterceptor.java:72,82` — When an unhandled exception propagates to the interceptor, the response envelope is constructed with `version = 1` regardless of the `version` parameter the caller passed. For this endpoint this would only matter if the data-inconsistency branch in extension 2b were triggered by a caller using `version=2`.

## Open Questions

_(none)_

---

## Request Parameters

```json
{
  "type": "object",
  "properties": {
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
    },
    "maxCount": {
      "type": "integer"
    }
  },
  "required": ["key"]
}
```

**`key`** — API key. Required on every request; omitting it returns HTTP 401.

**`version`** — Selects the response envelope format. Only version `2` is specified here; version `1` is a legacy format and is outside the scope of this spec. Passing an unrecognised value returns HTTP 500.

**`includeReferences`** — When `true` (default), full agency detail objects are placed in `data.references.agencies`. When `false`, `data.references.agencies` is an empty array; coverage entries in the list still identify agencies by ID.

**`maxCount`** — Accepted but not applied due to a bug; the result set is always the full list of agencies. See Suspected Defects.

---

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

**`code`** — HTTP status code mirrored in the body; `200` on success.

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`text`** — Human-readable status string; `"OK"` on success.

**`version`** — API version that was used to format the response; echoes the `version` request parameter (or `2` if omitted).

**`data`** — The response payload; see below.

---

### data

```json
{
  "type": "object",
  "properties": {
    "limitExceeded": { "type": "boolean" },
    "list":          { "type": "array", "items": { "type": "object" } },
    "references":    { "type": "object" }
  }
}
```

**`data.limitExceeded`** — Always `false` for this endpoint (see Suspected Defects regarding `maxCount`).

**`data.list`** — Array of coverage entries, one per agency. Order is unspecified; callers must not rely on a particular sequence.

**`data.references`** — Standard references block. For this endpoint only `data.references.agencies` will ever be populated; all other reference arrays (`routes`, `stops`, `trips`, `situations`, `stopTimes`) are always empty.

---

### data.list[]

```json
{
  "type": "object",
  "properties": {
    "agencyId": { "type": "string" },
    "lat":      { "type": "number" },
    "lon":      { "type": "number" },
    "latSpan":  { "type": "number" },
    "lonSpan":  { "type": "number" }
  }
}
```

**`data.list[].agencyId`** — The agency's identifier string. Matches the `id` field of the corresponding agency in `data.references.agencies`.

**`data.list[].lat`** — Centre latitude of the agency's service-area bounding box, in decimal degrees. Computed as the arithmetic mean of the northernmost and southernmost stop latitudes across all of the agency's scheduled trips. This is the midpoint of the bounding box, not the geographic centroid of all stop positions.

**`data.list[].lon`** — Centre longitude of the agency's service-area bounding box, in decimal degrees. Computed as the arithmetic mean of the easternmost and westernmost stop longitudes.

**`data.list[].latSpan`** — Full north–south extent of the bounding box in decimal degrees (max latitude minus min latitude).

**`data.list[].lonSpan`** — Full east–west extent of the bounding box in decimal degrees (max longitude minus min longitude).

---

### data.references.agencies[]

Present when `includeReferences=true` (the default). One object per agency, matching the agency IDs in `data.list`.

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
    "email":          { "type": "string" },
    "fareUrl":        { "type": "string" },
    "disclaimer":     { "type": "string" },
    "privateService": { "type": "boolean" }
  }
}
```

**`data.references.agencies[].id`** — Agency identifier; corresponds to `agencyId` in the list entries.

**`data.references.agencies[].name`** — Human-readable agency name (e.g. `"Metro Transit"`).

**`data.references.agencies[].url`** — Agency home page URL.

**`data.references.agencies[].timezone`** — IANA timezone name for the agency's operating area (e.g. `"America/Los_Angeles"`). All times in schedule responses for this agency are anchored to this timezone.

**`data.references.agencies[].lang`** — Two-letter ISO 639-1 language code for the agency's primary language (e.g. `"en"`). May be an empty string if not specified in the feed.

**`data.references.agencies[].phone`** — Public customer-service telephone number. May be an empty string.

**`data.references.agencies[].email`** — Public contact email address. May be an empty string.

**`data.references.agencies[].fareUrl`** — URL of the agency's fares information page. May be an empty string.

**`data.references.agencies[].disclaimer`** — Optional disclaimer text to display alongside data from this agency. May be an empty string.

**`data.references.agencies[].privateService`** — `true` if the agency operates private (non-public) service. `false` for standard public transit operators.
