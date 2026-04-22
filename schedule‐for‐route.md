## Goal in Context

A rider using a transit app wants to see the full timetable for a route on a specific day — every trip, in both directions, with all scheduled stop times — so they can plan travel or understand the route's service span.

## Scope

OBA REST API — `GET /api/where/schedule-for-route/{id}.json`

## Level

User goal

## Primary Actor

Rider

## Stakeholders and Interests

- **Rider** — wants a complete, correct picture of scheduled service for the route on the requested date, organised by direction of travel.

## Preconditions

- The server holds a valid transit data bundle for the queried region.
- The caller supplies a valid `{agencyId}_{routeId}` identifier.

## Minimal Guarantees

- The response is always valid JSON.
- A malformed `date` value (neither a `yyyy-MM-dd` string nor a numeric timestamp) returns HTTP 400 with a field-level validation error and no schedule data.
- A missing `id` path segment returns HTTP 404.

## Success Guarantees

- The response contains one grouping per direction of travel, each holding the canonical stop sequence for that direction, the sorted list of trip IDs operating on the queried date, and a per-trip list of scheduled stop times.
- The references block contains full detail records for every agency, route, trip, stop, and service alert referenced in the entry.

## Trigger

A client sends `GET /api/where/schedule-for-route/{id}.json` with an optional `date` parameter.

## Main Success Scenario

1. The client submits a request with a route `id` and an optional `date`.
2. The server resolves the `date` to a service date. If `date` is omitted, the current time is used. If a `yyyy-MM-dd` string is supplied, it is interpreted as midnight in the server's local timezone. A numeric value is treated as a Unix milliseconds timestamp. The timestamp is floored to the nearest 15-minute boundary to permit response caching.
3. The server locates all block trips belonging to the route's trip patterns and checks each trip's calendar against the resolved service date.
4. For each trip whose calendar marks the service date as active, the server records the trip under its direction ID and collects its stop sequence.
5. The server merges all stop sequences for each direction into a single canonical stop list using a topological sort of the stop graph. The sort gives a best-effort linear order; it works correctly for simple bidirectional routes and may produce an incomplete or unexpected order for loop routes or routes with complex branching.
6. For each direction grouping the server assembles:
   - `directionId` — the GTFS direction string for this group.
   - `tripHeadsigns` — the deduplicated set of trip headsign strings across all trips in the direction.
   - `stopIds` — the canonical stop sequence from step 5.
   - `tripIds` — all trip IDs operating in this direction on the service date, sorted lexicographically as `{agencyId}_{tripId}` strings.
   - `tripsWithStopTimes` — one entry per trip in the same sorted order, each containing the trip ID and all scheduled stop times for that trip sorted by arrival time.
7. The server collects the GTFS service IDs of all active trips and any service alerts affecting the route or its stops.
8. The response is returned with HTTP 200 and JSON code 200.

## Extensions

**2a. `date` format is invalid (neither `yyyy-MM-dd` nor a number):**
The type-conversion layer rejects the value before the action runs. The server returns HTTP 400 with a body of `{"fieldErrors":{"date":["…"]}}`. No schedule data is included.

**3a. Route ID does not exist in the data bundle:**
The server returns HTTP 404 and JSON code 404 with text `"resource not found"` and no `data` field.

**4a. Route exists, but the service date falls after all known calendar dates for every trip on the route:**
No trip's calendar has any date on or after the queried service date. The server returns HTTP 200 and JSON code 510 with text `"ServiceDateOutOfRange"` and no `data` field.

**4b. Route exists, the service date is within the calendar's horizon, but no trips run on that specific date:**
The server returns HTTP 200 and JSON code 510 with text `"NoServiceThatDay"`. The `data` field is present and contains an entry with the `routeId` and `scheduleDate`, empty `serviceIds` and `stopTripGroupings` arrays, and references populated with the agency and route records only.

---

## Suspected Defects

### Defects that affect the use case

**Both 510 error responses return HTTP 200.**
`ScheduleForRouteAction.setNoServiceResponse` and `setNoServiceThatDayResponse` (lines 104–114 of `ScheduleForRouteAction.java`) both call `new DefaultHttpHeaders()` without `.withStatus(code)`. The resource-not-found path correctly calls `.withStatus(404)`, so this is inconsistent. Go implementations should decide whether to return HTTP 200 (matching the Java behaviour) or the semantically correct HTTP 510.

**`serviceId` is always empty in all stop-time objects.**
`BeanFactoryV2.getStopTime` (line 761 of `BeanFactoryV2.java`) populates `arrivalTime`, `departureTime`, `arrivalEnabled`, `departureEnabled`, `stopHeadsign`, `tripId`, and `stopId` from the internal bean, but never calls `bean.setServiceId(…)`. The `serviceId` field defined on `ScheduleStopTimeInstanceV2Bean` is therefore always serialised as an empty string. The service ID is available in the intermediate bean (it is set by `RouteScheduleBeanServiceImpl.addStopTimeReference`), so omitting it from the response appears to be an oversight.

**`includeReferences=false` has no effect.**
`BeanFactoryV2.getRouteSchedule` (lines 949–1008 of `BeanFactoryV2.java`) calls `_references.setAgencies(…)`, `_references.setRoutes(…)`, etc., unconditionally. The `_includeReferences` flag (controlled by the `includeReferences` query parameter) is only consulted by the `shouldAddReferenceWithId` helper, which is not called by this code path. The references block is therefore always present regardless of the parameter value.

### Implementation defects only

**`stopsInDefaultOrder` list in `collapse()` is populated but never read.**
`RouteScheduleBeanServiceImpl.collapse` (line 412 of `RouteScheduleBeanServiceImpl.java`) maintains a `List<StopEntry> stopsInDefaultOrder` and calls `stopsInDefaultOrder.add(stop)` on each edge, but the list is not used anywhere in the method. The return value comes from `graph.getTopologicalSort(c)`. This is dead code.

**Arrival-time comparator violates the total-order contract.**
The `Comparator<StopTimeInstanceBean>` defined in `BeanFactoryV2.getRouteSchedule` (lines 961–969 of `BeanFactoryV2.java`) returns `1` instead of `0` when two stop times belong to the same trip and have equal arrival times. A non-zero return for equal elements can cause `Collections.sort` to produce an unstable or non-deterministic ordering for such stop times. Equal arrival times within the same trip would be unusual in practice but are not impossible.

**Binary search returns −1 for a single-element stop-times list.**
`BeanFactoryV2.getIndexStopTimesByTrip` (line 1044 of `BeanFactoryV2.java`) returns `−1` when `max == min == 0` (a list of exactly one element). `getIndexOfFirstStopTimeMatchForTrip` treats a return value of `−1` as a valid index to decrement, and `getStopTimesForTrip` then accesses `sortedStoptimesList.get(−1)`, which throws `IndexOutOfBoundsException`. A direction grouping whose single trip has exactly one stop would trigger this path.

---

## Open Questions

None. All behaviour described above was confirmed by static analysis of the full call chain and live server responses.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["id"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Path parameter. The route identifier in {agencyId}_{routeId} format."
    },
    "date": {
      "type": "string",
      "description": "The service date to query. Accepts yyyy-MM-dd or a Unix milliseconds integer. Defaults to the current time."
    },
    "key": {
      "type": "string",
      "description": "API key."
    },
    "version": {
      "type": "integer",
      "description": "Response envelope version. Defaults to 2."
    },
    "includeReferences": {
      "type": "boolean",
      "description": "Intended to suppress the references block, but has no effect on this endpoint (see Suspected Defects). Defaults to true."
    }
  }
}
```

**`id`** — Route identifier in `{agencyId}_{routeId}` form. Required. A string without an underscore separator causes an unhandled server error returning HTTP 200 with body `null`.

**`date`** — The service date for which the schedule is requested. A `yyyy-MM-dd` string is interpreted as midnight in the server's configured local timezone; a numeric value is treated as a Unix milliseconds timestamp. The resolved timestamp is floored to the nearest 15-minute boundary before use. When omitted, the current server time is used.

**`key`** — API authentication key. Required in production deployments.

**`version`** — Selects the response envelope version. This endpoint defaults to version 2 and only version 2 is supported.

**`includeReferences`** — Boolean flag that is documented to suppress the references block. Due to a defect, the references block is always present and this parameter has no observable effect.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "code": { "type": "integer", "description": "200 on success; 400 invalid argument; 404 not found; 510 out of service range or no service that day." },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "version": { "type": "integer" },
    "text": { "type": "string" },
    "data": { "type": "object" }
  }
}
```

**`code`** — HTTP-mirrored status code embedded in the JSON body. 200 for success, 400 for a bad `date` parameter, 404 if the route does not exist, 510 if the date is out of service range or has no service. Note: the HTTP status for 510 responses is 200 (see Suspected Defects).

**`currentTime`** — Server wall-clock time at the moment of the response, in Unix milliseconds.

**`version`** — Response envelope version; always 2 for this endpoint.

**`text`** — Human-readable status: `"OK"`, `"resource not found"`, `"ServiceDateOutOfRange"`, or `"NoServiceThatDay"`.

**`data`** — Absent when `code` is 404 or when `text` is `"ServiceDateOutOfRange"`. Present in all other cases.

---

### data.entry

```json
{
  "type": "object",
  "properties": {
    "routeId": { "type": "string" },
    "scheduleDate": { "type": "integer", "description": "Unix ms" },
    "serviceIds": {
      "type": "array",
      "items": { "type": "string" }
    },
    "stopTripGroupings": {
      "type": "array",
      "items": { "$ref": "#/definitions/StopTripGrouping" }
    }
  }
}
```

**`data.entry.routeId`** — The queried route's identifier in `{agencyId}_{routeId}` form.

**`data.entry.scheduleDate`** — Midnight of the resolved service date in the server's local timezone, expressed as Unix milliseconds.

**`data.entry.serviceIds`** — The GTFS service IDs whose calendars include the queried service date and have active trips on this route. The order is non-deterministic (derived from a hash set). Empty when `text` is `"NoServiceThatDay"`.

**`data.entry.stopTripGroupings`** — One element per direction of travel operating on the queried date. Empty when `text` is `"NoServiceThatDay"`. The ordering of groupings is non-deterministic.

---

### data.entry.stopTripGroupings[]

```json
{
  "type": "object",
  "properties": {
    "directionId": { "type": "string" },
    "tripHeadsigns": {
      "type": "array",
      "items": { "type": "string" }
    },
    "stopIds": {
      "type": "array",
      "items": { "type": "string" }
    },
    "tripIds": {
      "type": "array",
      "items": { "type": "string" }
    },
    "tripsWithStopTimes": {
      "type": "array",
      "items": { "$ref": "#/definitions/TripWithStopTimes" }
    }
  }
}
```

**`data.entry.stopTripGroupings[].directionId`** — The GTFS direction string for this group, typically `"0"` or `"1"`.

**`data.entry.stopTripGroupings[].tripHeadsigns`** — The deduplicated set of headsign strings across all trips in this direction. Serialised as an array; element order is non-deterministic.

**`data.entry.stopTripGroupings[].stopIds`** — The canonical stop sequence for this direction, computed as a topological sort of the directed stop-adjacency graph. Provides a best-effort linear ordering; may be incomplete for loop routes. Each element is a `{agencyId}_{stopId}` string.

**`data.entry.stopTripGroupings[].tripIds`** — All trip IDs operating in this direction on the service date, sorted lexicographically as `{agencyId}_{tripId}` strings.

**`data.entry.stopTripGroupings[].tripsWithStopTimes`** — One element per trip, in the same order as `tripIds`.

---

### data.entry.stopTripGroupings[].tripsWithStopTimes[]

```json
{
  "type": "object",
  "properties": {
    "tripId": { "type": "string" },
    "stopTimes": {
      "type": "array",
      "items": { "$ref": "#/definitions/StopTime" }
    }
  }
}
```

**`data.entry.stopTripGroupings[].tripsWithStopTimes[].tripId`** — The trip identifier in `{agencyId}_{tripId}` form.

**`data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes`** — The scheduled stop times for this trip, sorted by arrival time. See the stop time schema below.

---

### data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes[] (and data.references.stopTimes[])

Both the per-trip stop times in `tripsWithStopTimes` and the flat list in `data.references.stopTimes` share the same schema. The references list contains all stop times across all directions in unspecified order; the entry list organises the same records per trip.

```json
{
  "type": "object",
  "properties": {
    "tripId": { "type": "string" },
    "stopId": { "type": "string" },
    "arrivalTime": { "type": "integer" },
    "arrivalEnabled": { "type": "boolean" },
    "departureTime": { "type": "integer" },
    "departureEnabled": { "type": "boolean" },
    "serviceId": { "type": "string" },
    "stopHeadsign": { "type": "string" }
  }
}
```

**`tripId`** — Trip identifier in `{agencyId}_{tripId}` form.

**`stopId`** — Stop identifier in `{agencyId}_{stopId}` form.

**`arrivalTime`** — Scheduled arrival time in seconds elapsed since midnight of the service date. May exceed 86 400 for trips running past midnight. A value of `0` means the stop time is at exactly midnight; `arrivalEnabled` will be `false` in that case (see Suspected Defects).

**`arrivalEnabled`** — `true` if the arrival time is greater than 0. `false` indicates that the stop does not have a published arrival time in the GTFS data, or that the time is exactly midnight (a defect: 0 is used as a sentinel value, conflating "midnight" with "no time").

**`departureTime`** — Scheduled departure time in seconds elapsed since midnight of the service date.

**`departureEnabled`** — Follows the same logic as `arrivalEnabled`.

**`serviceId`** — Always empty; not populated. The service ID is available from `data.entry.serviceIds` (see Suspected Defects).

**`stopHeadsign`** — Always empty; not populated by the data pipeline.

---

### data.references

```json
{
  "type": "object",
  "properties": {
    "agencies": { "type": "array" },
    "routes": { "type": "array" },
    "trips": { "type": "array" },
    "stops": { "type": "array" },
    "stopTimes": { "type": "array" },
    "situations": { "type": "array" }
  }
}
```

**`data.references.agencies[]`** — One entry per agency referenced by the route. Fields: `id`, `name`, `lang`, `email`, `phone`, `disclaimer`, `timezone`, `url`, `fareUrl`, `privateService`.

**`data.references.routes[]`** — One entry per route referenced. Fields: `id`, `agencyId`, `shortName`, `longName`, `description`, `type`, `url`, `color`, `textColor`, and `nullSafeShortName`.

**`data.references.trips[]`** — One entry per trip active on the queried date. Fields: `id`, `routeId`, `serviceId`, `blockId`, `shapeId`, `directionId`, `tripHeadsign`, `routeShortName`, `tripShortName`, `timeZone`, `peakOffpeak`.

**`data.references.stops[]`** — One entry per stop visited by any active trip. Fields: `id`, `code`, `name`, `lat`, `lon`, `direction`, `locationType`, `parent`, `routeIds`, `staticRouteIds`.

**`data.references.stopTimes[]`** — Flat list of all scheduled stop times across all directions, in unspecified order. Same schema as `tripsWithStopTimes[].stopTimes[]`.

**`data.references.situations[]`** — Service alerts affecting the route or any of its stops on the queried date. Empty if no alerts are active.
