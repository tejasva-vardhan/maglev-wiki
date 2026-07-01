# Use Case: Get Stop

## Goal in Context

A rider's client application retrieves the full static record for a single transit stop — its location, name, and the routes that serve it — so the rider can confirm stop details or build a departure board.

## Scope

OneBusAway REST API, endpoint `GET /api/where/stop/{id}.json`

## Level

User goal

## Primary Actor

Rider (accessing the system via a client application)

## Stakeholders and Interests

- **Rider** — wants accurate stop details (location, name, routes) to plan a journey or check whether a stop is wheelchair-accessible.

## Preconditions

- The server has a loaded transit data bundle.
- The caller knows the combined stop ID in the form `{agencyId}_{entityId}` (e.g. `1_11870`).

## Minimal Guarantees

- If the request cannot be satisfied, the response body is `null` with HTTP status 200. (This is a defect; see Suspected Defects.)

## Success Guarantees

- The response contains the stop's geographic coordinates, human-readable name, stop code, direction, location type, wheelchair boarding annotation, the list of routes currently serving the stop, and — when the stop is part of a station — the combined ID of its parent station.
- All referenced routes and agencies appear in the references block.
- If the stop is a sub-platform of a parent station, the parent station's full record also appears in the references block.

## Trigger

Client sends `GET /api/where/stop/{id}.json?key={key}`.

## Main Success Scenario

1. The client supplies a combined stop ID in the URL path (e.g. `/api/where/stop/1_11870.json`).
2. The system parses the ID by splitting on the first underscore: everything before it is the agency ID; everything after is the stop's entity ID. ([`AgencyAndIdLibrary.convertFromString`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-util/src/main/java/org/onebusaway/util/AgencyAndIdLibrary.java#L40-L53))
3. The system looks up the stop in the transit graph. If the ID matches a consolidated stop alias, the canonical ID is substituted transparently. The extracted agency ID is not validated — the stop is returned regardless of whether the requested agency serves it. ([`StopBeanServiceImpl.getStopForIdForServiceDate`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopBeanServiceImpl.java#L113-L115)) *Maglev deviates from this — see [Implementation Decisions](#implementation-decisions).*
4. The system retrieves the stop's geographic coordinates (latitude and longitude) from the transit graph and its narrative attributes (name, code, direction, location type) from the narrative store. The code falls back to the raw entity portion of the combined ID if the GTFS feed supplies no `stop_code`. ([`fillStopBean`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopBeanServiceImpl.java#L173-L185))
5. The system retrieves every route collection that serves this stop, drawing from the full set of service dates without filtering to any particular date. Routes are sorted by name in natural string order (treating embedded digit runs as numbers). ([`fillRoutesForStopBean` with null service interval](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopBeanServiceImpl.java#L152-L170))
6. The system resolves the static route list for the stop. If the narrative store holds a configured static route list, that list is used; otherwise the static route list is identical to the route list from step 5. ([`fillTransfersForStopBean`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopBeanServiceImpl.java#L136-L149))
7. If the stop has a parent station, the system resolves the parent station's data (coordinates, narrative, routes) and adds the parent's combined ID to the entry's `parent` field. The full parent record is placed in the `stops` array of the references block. ([`BeanFactoryV2.getStop` parent handling](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L488-L501))
8. The system assembles the response: the entry for the stop, and — unless `includeReferences=false` was requested — a references block containing the full records for all routes in `routeIds` and their owning agencies.
9. The system returns HTTP 200 with the assembled JSON response.

## Extensions

**1a. The stop ID does not contain an underscore:**
- Parsing fails with an internal error.
- The exception interceptor attempts to return HTTP 500, but due to a defect (see Suspected Defects) the actual HTTP response is 200 with body `null`.

**2a. No stop exists for the supplied ID (and it is not a consolidated alias):**
- The service throws a not-found exception.
- The exception interceptor attempts to return HTTP 404, but due to a defect (see Suspected Defects) the actual HTTP response is 200 with body `null`.

**3a. `includeReferences=false` is supplied:**
- The references block is present in the response but all arrays within it are empty. Route and agency records are omitted.

## Suspected Defects

### Defects that affect the use case

**1. Error responses return HTTP 200 with a null body.**

`ExceptionInterceptor` ([lines 55–64](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ExceptionInterceptor.java#L55-L64)) catches all unhandled exceptions (including not-found and internal errors) and constructs an error response bean with the appropriate status code (404 or 500). It sets that code on a `DefaultHttpHeaders` instance and passes both to the content-type handler. However, the status code set on the `DefaultHttpHeaders` is not applied to the actual HTTP response by this pathway — the HTTP response is committed as 200, and the serialised body is `null` rather than the expected JSON error structure.

**Observable impact:** Callers cannot detect stop-not-found or malformed-ID conditions by HTTP status code. The body `null` must be used as the error signal.

**Likely intended behaviour:** HTTP 404 with a JSON envelope containing `"code": 404` and a descriptive `text` field for stop-not-found; HTTP 400 or 500 with `"code": 400`/`500` for malformed IDs and unexpected errors. *Maglev intentionally corrects this — see [Implementation Decisions](#implementation-decisions).*

**2. Error response version is hardcoded to 1.**

When an exception is caught, `ExceptionInterceptor.getExceptionAsResponseBean` ([line 72](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ExceptionInterceptor.java#L72)) constructs a `ResponseBean` with the version literal `1`, regardless of the `version` parameter supplied by the caller. The `version` field in error responses is therefore always 1 rather than the requested version.

### Implementation defects only

**3. Dead null-check in `StopAction.show()`.**

`StopAction.show()` ([line 61](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopAction.java#L61-L62)) checks `if (stop == null)` and calls `setResourceNotFoundResponse()`. This branch is unreachable: the service method (`StopBeanServiceImpl.getStopForIdForServiceDate`) either throws `NoSuchStopServiceException` or returns a non-null `StopBean`. The null-return path does not exist.

## Implementation Decisions

**Unknown stop ID returns HTTP 404 and malformed ID returns HTTP 400 (deviates from legacy).**

The legacy implementation returns HTTP 200 with a null body for both stop-not-found and malformed ID conditions (see [Suspected Defects](#suspected-defects)). Maglev intentionally corrects this: an unknown stop ID returns HTTP 404 with a standard error envelope, and a malformed ID (no underscore) returns HTTP 400, consistent with the treatment of other unknown and invalid entity IDs across the API.

**Agency namespace validation returns HTTP 404 when the stop is not served by the requested agency (deviates from legacy).**

The legacy implementation extracts the agency ID from the combined stop ID but uses only the entity ID portion when looking up the stop. As a result, `WrongAgency_StopX` and `RealAgency_StopX` return the same stop record as long as `StopX` exists — the agency prefix is parsed but never validated. Maglev intentionally enforces namespace correctness: after fetching the stop's routes, the handler checks that at least one route belongs to the requested agency. If none do, the endpoint returns HTTP 404. This prevents the agency ID from functioning as a cosmetic namespace alias with no enforcement.

A stop can be served by routes from multiple agencies (interlining). In that case the stop is reachable under any of those agencies' namespaces and Maglev returns 200 for all of them.

**Orphaned stops (no routes) are retrievable under any agency namespace that exists in the database.**

Maglev's `stops` table has no `agency_id` column; agency ownership of a stop is inferred through the route-trip-stop_time join chain. A stop with no routes has no inferable agency. Rather than reject all requests for such stops, Maglev falls back to confirming that the requested agency ID exists in the database. If it does, the stop is returned with an empty `routeIds` list. If the agency does not exist, the endpoint returns HTTP 404. This is a pragmatic concession to the schema limitation; it means orphaned stops are less strictly namespaced than routed stops.

## Request Parameters

```json
{
  "type": "object",
  "required": ["id"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Combined stop ID in the form {agencyId}_{entityId}, encoded in the URL path"
    },
    "key": {
      "type": "string",
      "description": "API key"
    },
    "version": {
      "type": "integer",
      "default": 2,
      "description": "API version"
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "Whether to populate the references block in the response"
    }
  }
}
```

**`id`** — Combined stop ID of the form `{agencyId}_{entityId}` (e.g. `1_11870`). The system splits on the first underscore, so the entity ID portion may itself contain underscores. Supplied as a URL path segment: `/api/where/stop/{id}.json`.

**`key`** — API key for the requesting application. Passed as a query parameter.

**`version`** — Selects the API response format. Only version 2 is implemented in Maglev; version 1 is out of scope.

**`includeReferences`** — When `false`, the references block is present but all arrays are empty. Defaults to `true`.

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "code": { "type": "integer" },
    "text": { "type": "string" },
    "version": { "type": "integer" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "data": { "type": "object" }
  }
}
```

**`code`** — HTTP status mirror: 200 on success.

**`text`** — Human-readable status string (e.g. `"OK"`).

**`version`** — API version of the response (2).

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`data`** — Container for the entry and references.

---

### data.entry

```json
{
  "type": "object",
  "properties": {
    "id":                { "type": "string" },
    "lat":               { "type": "number" },
    "lon":               { "type": "number" },
    "name":              { "type": "string" },
    "code":              { "type": "string" },
    "direction":         { "type": "string" },
    "locationType":      { "type": "integer" },
    "wheelchairBoarding":{ "type": "string" },
    "routeIds":          { "type": "array", "items": { "type": "string" } },
    "staticRouteIds":    { "type": "array", "items": { "type": "string" } },
    "parent":            { "type": "string" }
  }
}
```

**`data.entry.id`** — Combined stop ID (`agencyId_entityId`).

**`data.entry.lat`** — WGS-84 latitude of the stop.

**`data.entry.lon`** — WGS-84 longitude of the stop.

**`data.entry.name`** — Human-readable stop name (e.g. `"S Jackson St & 30th Ave S"`).

**`data.entry.code`** — Short public stop code shown on signage (e.g. `"11870"`). Taken from the GTFS `stop_code` field; falls back to the entity portion of the combined ID when the feed supplies no code.

**`data.entry.direction`** — Compass direction of travel at this stop (e.g. `"N"`, `"S"`, `"NE"`). Optional GTFS extension; absent when the feed does not supply it. Serialised as an empty string `""` when absent.

**`data.entry.locationType`** — GTFS location type integer: `0` = boarding/alighting platform (default), `1` = station, `2` = station entrance or exit, `3` = generic pathway node, `4` = boarding area.

**`data.entry.wheelchairBoarding`** — Wheelchair accessibility annotation drawn from the GTFS feed. One of `"UNKNOWN"`, `"ACCESSIBLE"`, `"NOT_ACCESSIBLE"`, or `"PARTIALLY_ACCESSIBLE"` (experimental). Absent when the feed provides no annotation.

**`data.entry.routeIds`** — Ordered array of combined route IDs (`agencyId_entityId`) for every route that serves this stop, across all service dates. Sorted by route short name in natural string order (numeric digit runs are compared as numbers, so `"14"` < `"101"` < `"B"`). When short name is absent, long name is used for sorting.

**`data.entry.staticRouteIds`** — Ordered array of combined route IDs representing the curated static route list for this stop. Used by some clients for display purposes (e.g. a stop sign showing which routes stop here regardless of real-time service). When no static list is configured for the stop, this field is identical to `routeIds`.

**`data.entry.parent`** — Combined ID of the enclosing station stop, if this stop is a sub-platform of a GTFS station. When no parent exists, this field is serialised as an empty string `""`. The full parent record is provided in `data.references.stops[]`.

---

### data.references

```json
{
  "type": "object",
  "properties": {
    "agencies":   { "type": "array", "items": { "type": "object" } },
    "routes":     { "type": "array", "items": { "type": "object" } },
    "stops":      { "type": "array", "items": { "type": "object" } },
    "trips":      { "type": "array", "items": { "type": "object" } },
    "situations": { "type": "array", "items": { "type": "object" } },
    "stopTimes":  { "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.references.agencies[]`** — Full agency records for each agency that owns a route in `routeIds`. Empty when `includeReferences=false`.

**`data.references.routes[]`** — Full route records for each route in `routeIds`. Empty when `includeReferences=false`.

**`data.references.stops[]`** — Contains the full record of the parent station stop, if `data.entry.parent` is non-empty. Empty when the stop has no parent, or when `includeReferences=false`. The parent record follows the same structure as `data.entry`, with all fields populated from the database. Note: the GTFS spec permits a three-level hierarchy (Station → Platform → Boarding Area), but this spec assumes a maximum of two levels; boarding area stops are out of scope.

**`data.references.trips[]`** — Always empty for this endpoint.

**`data.references.situations[]`** — Always empty for this endpoint.

**`data.references.stopTimes[]`** — Always empty for this endpoint.
