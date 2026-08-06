# stops-for-route

## Goal in Context

A rider's client application needs to know which stops a route serves and how those stops are grouped by travel direction, so the client can render route maps, populate stop-picker interfaces, and show passengers which direction their bus is heading.

## Scope

OneBusAway REST API — `GET /api/where/stops-for-route/{id}.json`

## Level

User-goal

## Primary Actor

Rider (via a client application)

## Stakeholders and Interests

- **Rider** — wants an accurate, ordered list of stops for a route, grouped by direction, with optional geometric path data for map display.

## Preconditions

- The caller supplies a valid combined route ID (`{agencyId}_{entityId}`) in the URL path.
- The API key is present in the request.

## Minimal Guarantees

- The response always uses HTTP 200 for successful lookups and for well-formed requests against unknown routes (404).
- When the route is found, the response always includes at least `routeId`, `stopIds`, `stopGroupings`, and a single grouping of type `"direction"` (even if it contains no groups).

## Success Guarantees

- `data.entry.stopIds` contains the combined IDs of all stops serving the route, sorted lexicographically.
- `data.entry.stopGroupings[0].stopGroups` contains one group per GTFS direction, each with stops ordered to approximate the physical route sequence.
- Full stop and route objects appear in `data.references`.
- When `includePolylines` is `true` (the default), encoded polyline geometry is present at both the route level and per direction group.

## Trigger

A GET request to `/api/where/stops-for-route/{id}.json`.

## Main Success Scenario

1. The caller issues a GET request with a combined route ID in the URL path (e.g., `/api/where/stops-for-route/1_102718.json`). The `id` path segment is required; omitting it produces a validation error.

2. The server checks the `display.serviceDateFiltering` configuration. When that flag is **disabled** (the default), and no `date` parameter is supplied, the server queries for every stop that appears on any trip for the route across all service dates — weekdays, weekends, holiday schedules, and future service are all included. When the flag is **enabled** and no `date` is supplied, the server substitutes today's date before proceeding as in step 3. ([`StopsForRouteAction.java` lines 88–100](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForRouteAction.java#L88-L100))

3. When a `date` parameter is supplied (either as `yyyy-MM-dd` or as a Unix epoch millisecond timestamp), the server converts it to a service interval whose boundaries are rounded to the nearest 15-minute bin ([`ApiIntervalFactory.java` lines 45–52](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/services/ApiIntervalFactory.java#L45-L52)). Only trips that are active within that interval are considered for both the flat stop list and the direction groups. When no trips are active on the requested date the flat list and all direction groups are empty, but the direction grouping element is still present in the response.

4. The server assembles the flat stop list by collecting the IDs of all stops from the scheduled stop times of qualifying trips, deduplicating them, and sorting the result lexicographically by combined stop ID. ([`RouteServiceImpl.java` lines 67–95](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/RouteServiceImpl.java#L67-L95); [`RouteBeanServiceImpl.java` line 390](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteBeanServiceImpl.java#L390)) This flat list is not ordered by stop position and should not be used to infer route sequence.

5. The server builds direction groups from the route's block trip indices. Qualifying block trips are gathered and their stop sequences are grouped by GTFS `direction_id`. When all sequences carry a direction ID the grouping is exact; when any sequence lacks a direction ID the server falls back to a similarity-based grouping that assigns integer strings ("0", "1", …) as group identifiers. ([`StopSequenceCollectionServiceImpl.java` lines 159–230](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/StopSequenceCollectionServiceImpl.java#L159-L230))

6. Within each direction group, the server infers a canonical stop order by constructing a directed graph of stop adjacencies across all stop sequences in that group, then performing a topological sort. For simple linear routes this approximates the physical sequence from departure to terminus. For loop routes or routes with complex branching patterns the topological sort may not yield a meaningful linear order; for a small set of known problematic routes (configured on the server) the insertion order is used directly instead. ([`RouteBeanServiceImpl.java` lines 344–371](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteBeanServiceImpl.java#L344-L371))

7. Each direction group's human-readable name is the trip headsign that appears most frequently across all trips in that direction. When two or more directions share the same most-common headsign, a direction disambiguator is appended to each name. ([`StopSequenceCollectionServiceImpl.java` lines 260–292](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/StopSequenceCollectionServiceImpl.java#L260-L292))

8. Direction groups are sorted alphabetically by their destination name. The flat stop list is sorted lexicographically by combined stop ID. ([`RouteBeanServiceImpl.java` lines 388–415](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteBeanServiceImpl.java#L388-L415))

9. When `includePolylines` is `true` (the default), the response includes encoded polyline geometry both at the entry level (the merged route path) and within each direction group (per-direction shapes derived from the shape IDs of qualifying trips). When `includePolylines` is `false`, both the entry-level and group-level polylines fields are present in the response but empty. ([`BeanFactoryV2.java` lines 1182–1190](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1182-L1190))

10. Full stop objects (id, name, lat/lon, code, direction, location type, wheelchair boarding, route IDs) and the full route object are placed in `data.references`. The caller can resolve any ID in `stopIds` or in any group's `stopIds` by looking up the matching entry in `data.references.stops`.

11. The server returns HTTP 200 with the assembled response.

## Extensions

**2a. Route ID not found:**
If the combined route ID is syntactically valid but refers to a route that does not exist, the service method returns null. The action returns HTTP 404 with `code: 404` and `text: "resource not found"`. ([`StopsForRouteAction.java` lines 101–102](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForRouteAction.java#L101-L102))

**2b. Required `id` is blank:**
Struts2 validates that `id` is non-empty before the action executes. If it is missing or empty, the action returns HTTP 400 with `code: 400` and `text: "validation error"`.

**2c. Malformed route ID (no underscore separator):**
If the route ID string contains no underscore, the combined-ID parser throws an internal exception. The action does not catch it, and the framework returns HTTP 200 with a JSON body of `null`. See Suspected Defects.

## Suspected Defects

### Defects that affect the use case

**Malformed ID returns HTTP 200 null instead of a client error (`StopsForRouteAction` / `AgencyAndIdLibrary`)**

When a caller supplies a route ID that contains no underscore (e.g., `stops-for-route/invalid.json`), `AgencyAndIdLibrary.convertFromString` throws an `IllegalStateException`. ([`AgencyAndIdLibrary.java` lines 46–48](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-util/src/main/java/org/onebusaway/util/AgencyAndIdLibrary.java#L46-L48)) This exception is unhandled in `StopsForRouteAction.show()`, so the response bean is never set and the framework serialises a `null` model, producing HTTP 200 with a JSON body of `"null"`. The intended behaviour is HTTP 400 (the ID is syntactically invalid) or HTTP 404.

### Implementation defects only

**`BeanFactoryV2.getStopsForRoute` mutates shared `StopGroupBean` objects (`BeanFactoryV2`)**

When `includePolylines` is `false`, the factory sets `group.setPolylines(null)` on the `StopGroupBean` instances it received from the service layer. ([`BeanFactoryV2.java` lines 1183–1186](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1183-L1186)) These objects are shared with the `StopsForRouteBean` that was returned from the cache. In the current deployment the cache serialises values on write and deserialises on read (EhCache with `isValueSerializable = true`), so each call operates on a fresh copy and the mutation is contained. If the cache were replaced with a reference-based store, any request with `includePolylines=false` would permanently null out the cached group polylines for the remainder of the cache entry's lifetime, causing subsequent requests with `includePolylines=true` to silently return empty group polylines.

## Request Parameters

```json
{
  "type": "object",
  "required": ["id"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Combined route ID, encoded in the URL path: /api/where/stops-for-route/{id}.json"
    },
    "includePolylines": {
      "type": "boolean",
      "default": true,
      "description": "Whether to include encoded polyline geometry in the response"
    },
    "date": {
      "type": "string",
      "description": "Service date filter. Accepts yyyy-MM-dd or Unix epoch milliseconds. When omitted, all stops across all service dates are returned (unless server-side date filtering is enabled)."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "When false, the references block is returned empty"
    }
  }
}
```

**`id`** — Combined route ID in the form `{agencyId}_{entityId}` (e.g., `1_102718`). Must contain an underscore. Supplied as the final path segment before the `.json` extension.

**`includePolylines`** — Controls whether encoded polyline geometry is included in the response. When `true` (the default), polylines are present at both the entry level and within each direction group. When `false`, both locations return an empty array.

**`date`** — Restricts the result to stops that are served by trips active on the specified date. Accepts `yyyy-MM-dd` (parsed in the server's local timezone) or a Unix epoch millisecond integer. When omitted and server-side service date filtering is disabled (the default), stops from all service dates are included. When omitted and server-side filtering is enabled, today's date is substituted.

**`includeReferences`** — Standard parameter present on all endpoints. When `false`, the `data.references` block is present but empty.

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
    "data": {
      "type": "object",
      "properties": {
        "entry":      { "type": "object" },
        "references": { "type": "object" }
      }
    }
  }
}
```

**`version`** — Always `2` for this endpoint.

**`code`** — HTTP-style status: `200` on success, `400` for a missing `id`, `404` for an unknown route.

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`data.entry`** — The stops-for-route payload (see below).

**`data.references`** — Shared objects referenced by ID in the entry: `stops`, `routes`, `agencies`, `trips`, `situations`.

---

### data.entry

```json
{
  "type": "object",
  "properties": {
    "routeId":       { "type": "string" },
    "stopIds":       { "type": "array", "items": { "type": "string" } },
    "stopGroupings": { "type": "array", "items": { "type": "object" } },
    "polylines":     { "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.entry.routeId`** — Combined route ID (`{agencyId}_{entityId}`) identifying the queried route.

**`data.entry.stopIds`** — Flat, deduplicated list of every combined stop ID served by the route, sorted lexicographically by the combined ID string. This list is not ordered by position along the route; use `stopGroupings[].stopGroups[].stopIds` for sequenced stop data.

**`data.entry.stopGroupings`** — Always contains exactly one element, whose `type` is `"direction"` and `ordered` is `true`. Each element in `stopGroups` represents one direction of travel.

**`data.entry.polylines`** — Merged encoded polyline geometry covering the entire route. Empty array when `includePolylines=false` or when no qualifying trips have associated shapes.

---

### data.entry.stopGroupings[]

```json
{
  "type": "object",
  "properties": {
    "type":       { "type": "string" },
    "ordered":    { "type": "boolean" },
    "stopGroups": { "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.entry.stopGroupings[].type`** — Always `"direction"`.

**`data.entry.stopGroupings[].ordered`** — Always `true`, indicating that the stop IDs within each group are in route sequence order (best-effort topological sort).

**`data.entry.stopGroupings[].stopGroups`** — One element per GTFS direction. Sorted alphabetically by the direction group's destination name. Empty when no qualifying trips exist for the requested date.

---

### data.entry.stopGroupings[].stopGroups[]

```json
{
  "type": "object",
  "properties": {
    "id":       { "type": "string" },
    "name": {
      "type": "object",
      "properties": {
        "type":  { "type": "string" },
        "names": { "type": "array", "items": { "type": "string" } }
      }
    },
    "stopIds":   { "type": "array", "items": { "type": "string" } },
    "polylines": { "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.entry.stopGroupings[].stopGroups[].id`** — Direction identifier. When the GTFS feed supplies `direction_id` for all trips on the route, this is that GTFS value (typically `"0"` or `"1"`). When the feed lacks `direction_id`, the server assigns sequential integer strings based on stop-sequence similarity grouping.

**`data.entry.stopGroupings[].stopGroups[].name.type`** — Always `"destination"`.

**`data.entry.stopGroupings[].stopGroups[].name.names`** — One or more destination strings. Typically a single entry containing the most common trip headsign for that direction. When two directions share the same most-common headsign, a direction disambiguator is appended to each.

**`data.entry.stopGroupings[].stopGroups[].stopIds`** — Combined stop IDs for this direction, ordered by an inferred canonical sequence. The order is derived from a topological sort of the stop adjacency graph built from all trip patterns in this direction; for simple linear routes it approximates the physical travel order. For routes with loop patterns, parallel branches, or other topological cycles the order may be approximate or arbitrary.

**`data.entry.stopGroupings[].stopGroups[].polylines`** — Encoded polyline shapes for the route path in this direction. Empty array when `includePolylines=false` or when qualifying trips have no associated shapes.

---

### data.entry.polylines[] and data.entry.stopGroupings[].stopGroups[].polylines[]

```json
{
  "type": "object",
  "properties": {
    "points": { "type": "string" },
    "length": { "type": "integer" },
    "levels": { "type": "string" }
  }
}
```

**`points`** — Google Encoded Polyline Algorithm string representing the sequence of lat/lon coordinates.

**`length`** — Number of coordinate points encoded in `points`.

**`levels`** — Zoom-level hints encoded as a string; may be empty or absent when not provided by the data source.

---

### data.references

```json
{
  "type": "object",
  "properties": {
    "stops":     { "type": "array", "items": { "type": "object" } },
    "routes":    { "type": "array", "items": { "type": "object" } },
    "agencies":  { "type": "array", "items": { "type": "object" } },
    "trips":     { "type": "array", "items": { "type": "object" } },
    "situations":{ "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.references.stops`** — Full stop objects for every stop ID that appears in `data.entry.stopIds` or in any `stopGroups[].stopIds`. Each stop object includes: `id` (combined), `lat`, `lon`, `name`, `code`, `direction` (taken from the feed when defined, otherwise inferred from shape geometry — see [[Stop-Direction-Calculation]]), `locationType`, `wheelchairBoarding`, `routeIds`, and `staticRouteIds`.

**`data.references.routes`** — Full route object for the queried route, including `id`, `agencyId`, `shortName`, `longName`, `description`, `type`, `url`, `color`, and `textColor`.

**`data.references.agencies`** — Agency objects for any agency referenced in the route or stops.

**`data.references.trips`** — Populated only if trip references are added elsewhere; typically empty for this endpoint.

**`data.references.situations`** — Service alert objects; typically empty for this endpoint.
