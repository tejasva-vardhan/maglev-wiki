# agencies-with-coverage

## Goal in Context

A client app needs to know which transit operators are available in this server's data bundle and where each one operates, so it can set an appropriate initial map viewport and offer agency-filtering controls.

## Scope

The `GET /api/where/agencies-with-coverage.json` endpoint.

## Level

User-goal.

## Primary Actor

Rider.

## Stakeholders and Interests

**Rider** (via a client app) — wants the list of agencies and their geographic coverage so the app can centre the map on a relevant area and enumerate available operators.

## Preconditions

The server has loaded a GTFS data bundle. If the bundle is still being prepared at request time the server blocks internally until it is ready before processing the query.

## Minimal Guarantees

The server always returns a well-formed JSON envelope with a `version`, `code`, `text`, `currentTime`, and `data` field. The `data.list` array is always present (it may be empty if the bundle contains no agencies).

## Success Guarantees

The rider's client receives the complete list of agencies in the bundle, each with its coverage bounding-box centre and dimensions, and (unless suppressed) the full agency records in the references block.

## Trigger

A `GET` request to `/api/where/agencies-with-coverage.json`.

## Main Success Scenario

1. The rider's client sends `GET /api/where/agencies-with-coverage.json?key=<key>`.
2. The server computes (or retrieves from its in-process cache) the geographic bounding box of each agency's service network. The bounding box for each agency is the smallest rectangle that encloses every stop that appears in the stop-time records of any trip belonging to any route operated by that agency. The computation is performed once on first call and cached for the lifetime of the server process. ([`AgencyServiceImpl.java` lines 110–134](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/AgencyServiceImpl.java#L110-L134))
3. For each agency, the server derives the bounding-box centre as the arithmetic mean of the minimum and maximum latitude and the arithmetic mean of the minimum and maximum longitude, and derives the spans as `maxLat − minLat` and `maxLon − minLon`. ([`TransitDataServiceTemplateImpl.java` lines 191–196](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/federated/TransitDataServiceTemplateImpl.java#L191-L196))
4. The server assembles one list entry per agency containing its ID, centre coordinates, and bounding-box dimensions.
5. The server adds a full agency record for each listed agency to the references block. ([`BeanFactoryV2.java` line 1483](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1483))
6. The server returns HTTP 200. The response body contains the list in `data.list`, the limit-exceeded flag (always `false`) in `data.limitExceeded`, and agency records in `data.references.agencies`.

## Extensions

**1a. References suppressed** (`includeReferences=false`): Step 5 is skipped. `data.references.agencies` is an empty array in the response.

**1b. Unknown version requested** (`version` set to an integer other than 2): The server returns HTTP 500 with `code` 500 and `text` "unknown version: *n*".

## Suspected Defects

### Defects that affect the use case

**`maxCount` parameter accepted but never applied.**
`AgenciesWithCoverageAction` declares a `maxCount` field and a public setter, but at [line 61](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/AgenciesWithCoverageAction.java#L61) the response factory is created without passing the accumulated limit value. As a result, the list is never truncated regardless of the `maxCount` value supplied by the caller, and `data.limitExceeded` is always `false`.

### Implementation defects only

**Non-deterministic list ordering.**
The coverage map is built as a `HashMap` keyed by agency ID at [`AgencyServiceImpl.java` line 113](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/AgencyServiceImpl.java#L113), so the iteration order — and therefore the order of entries in `data.list` — is not guaranteed. A clean reimplementation may produce a different (but equally valid) ordering without this being a behavioural change.

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
  "required": []
}
```

**`key`** — API access key. Required in practice; must be supplied on every request.

**`version`** — API version selector. Must be `2` or omitted. Any other integer returns HTTP 500.

**`includeReferences`** — When `true` (the default), full agency records are included in `data.references.agencies`. When `false`, `data.references.agencies` is an empty array.

**`maxCount`** — Accepted by the server but has no effect (see Suspected Defects). Intended to cap the number of agencies returned; currently ignored.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "version":     { "type": "integer" },
    "code":        { "type": "integer" },
    "text":        { "type": "string" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "data":        { "type": "object" }
  }
}
```

**`version`** — API version in use (always `2`).

**`code`** — HTTP-like status code. `200` on success.

**`text`** — Human-readable status string. `"OK"` on success.

**`currentTime`** — Server time when the response was generated, in Unix milliseconds.

**`data`** — Container for the list payload and references; see below.

---

### data

```json
{
  "type": "object",
  "properties": {
    "limitExceeded": { "type": "boolean" },
    "list":          { "type": "array" },
    "references":    { "type": "object" }
  }
}
```

**`data.limitExceeded`** — Always `false` (see Suspected Defects).

**`data.list`** — Array of agency-with-coverage entries; see `data.list[]` below.

**`data.references`** — Related entity records; see `data.references` below.

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

**`data.list[].agencyId`** — The plain agency identifier string (e.g. `"1"`, `"40"`). Not in the combined `agencyId_entityId` form used for routes and stops, because the agency is itself the top-level entity.

**`data.list[].lat`** — Latitude of the centre of the agency's coverage bounding box, in decimal degrees. Computed as `(minLat + maxLat) / 2` across all stops in the agency's stop-time records.

**`data.list[].lon`** — Longitude of the centre of the coverage bounding box, in decimal degrees. Computed as `(minLon + maxLon) / 2`.

**`data.list[].latSpan`** — Height of the coverage bounding box in decimal degrees of latitude (`maxLat − minLat`). Zero if the agency has no stop-time records.

**`data.list[].lonSpan`** — Width of the coverage bounding box in decimal degrees of longitude (`maxLon − minLon`). Zero if the agency has no stop-time records.

---

### data.references

```json
{
  "type": "object",
  "properties": {
    "agencies":   { "type": "array", "items": { "type": "object" } },
    "routes":     { "type": "array" },
    "stops":      { "type": "array" },
    "trips":      { "type": "array" },
    "stopTimes":  { "type": "array" },
    "situations": { "type": "array" }
  }
}
```

**`data.references.agencies`** — Full agency records for every entry in `data.list`, when `includeReferences=true`. Empty array when `includeReferences=false`. Each element has the shape described in `data.references.agencies[]` below.

**`data.references.routes`**, **`data.references.stops`**, **`data.references.trips`**, **`data.references.stopTimes`**, **`data.references.situations`** — Always empty arrays for this endpoint.

---

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

**`data.references.agencies[].id`** — Agency identifier. Matches the `agencyId` in the corresponding `data.list[]` entry.

**`data.references.agencies[].name`** — Display name of the agency (e.g. `"Metro Transit"`).

**`data.references.agencies[].url`** — URL of the agency's public website.

**`data.references.agencies[].timezone`** — IANA timezone name for the agency (e.g. `"America/Los_Angeles"`).

**`data.references.agencies[].lang`** — Language code for the agency's primary language (e.g. `"en"`). May be an empty string if not provided in the GTFS feed.

**`data.references.agencies[].phone`** — Public contact phone number. May be an empty string.

**`data.references.agencies[].disclaimer`** — Legal disclaimer text the agency requests be shown to users. May be an empty string.

**`data.references.agencies[].email`** — Public contact email address. May be an empty string.

**`data.references.agencies[].fareUrl`** — URL for fare information. May be an empty string.

**`data.references.agencies[].privateService`** — `true` if the agency provides private service not available to the general public; `false` otherwise.
