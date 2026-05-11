# schedule-for-route

## Goal in Context

A rider wants to see the full published timetable for a route on a specific day — all trips, every stop, with departure times — so they can plan a journey without relying on real-time data.

## Scope

Transit information system.

## Level

User-goal.

## Primary Actor

Rider (accessing the system via a client application).

## Stakeholders and Interests

- **Rider** — wants the complete scheduled timetable for a route on a given day, organised by direction of travel, so they can read it like a printed route schedule.

## Preconditions

- The server has a loaded transit bundle.
- The caller supplies a valid combined route ID.

## Minimal Guarantees

- Every response includes a numeric status code and a human-readable text field.
- A route ID that does not exist in the data always produces a not-found response.

## Success Guarantees

- The response contains all trips operating on the requested route for the requested service date, grouped by direction of travel.
- Each direction group includes an ordered list of stops (canonical travel order), an ordered list of trip IDs, and per-trip stop-time detail.
- The references block contains full objects for all agencies, routes, stops, and trips referenced by the entry.

## Trigger

A GET request arrives at `/api/where/schedule-for-route/{id}.json` (or `.xml`).

## Main Success Scenario

1. The caller supplies a combined route ID in the URL path and an optional service date.
2. The system resolves the service date: if no date is supplied, it uses the current wall-clock time; otherwise it parses the supplied value. The date is then rounded down to the nearest 15-minute boundary as a cache-optimisation hint before the data layer looks up the corresponding calendar date. ([`ApiIntervalFactory.java#L45-L52`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/services/ApiIntervalFactory.java#L45-L52))
3. The system looks up the route collection for the supplied ID. If no such route exists, it returns a not-found response (see Extension 3a).
4. The system iterates over every block trip belonging to that route collection and checks each trip's calendar against the requested service date. ([`RouteScheduleBeanServiceImpl.java#L160-L210`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteScheduleBeanServiceImpl.java#L160-L210))
5. For each trip whose calendar is active on the service date, the system:
   - Records the trip as running that day.
   - Groups the trip under its direction ID.
   - Collects the trip's headsign (from narrative data; falls back to the name of the last stop if no headsign is recorded).
   - Accumulates the trip's stop sequence for later merging.
   - Collects all per-stop scheduled times.
6. If no trips are active on the requested date and no service for this route is scheduled on any future date, the system returns a service-date-out-of-range response (see Extension 3b).
7. If no trips are active on the requested date, the system returns a no-service-that-day response (see Extension 3c).
8. For each direction group the system derives a canonical stop order using a topological sort across all collected stop sequences for that direction. This heuristic works well for simple linear routes but may produce an imperfect ordering for loops or routes with complex branching. ([`RouteScheduleBeanServiceImpl.java#L411-L436`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteScheduleBeanServiceImpl.java#L411-L436))
9. Trip IDs within each direction group are sorted in ascending order by their combined ID string. ([`BeanFactoryV2.java#L977`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L977))
10. Stop times within each direction group are sorted by trip ID, then by arrival time within each trip, and then assembled into per-trip `tripsWithStopTimes` entries. ([`BeanFactoryV2.java#L961-L983`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L961-L983))
11. The references block is populated with all agencies, routes, stops, trips, service alerts, and a flat list of all stop-time instances.
12. The system returns HTTP 200 with the complete route schedule entry.

## Extensions

**3a. Route not found**

At step 3, if the route collection ID is not present in the transit graph, the system returns HTTP 404 with `code: 404` and no data body.

**3b. Date is after all service has ended**

After step 5, if no block trips have any scheduled service on the requested date *and* no service is scheduled for any later date, the system returns HTTP 200 with `code: 510`, `text: "ServiceDateOutOfRange"`, and no data body. This occurs when querying a date that falls entirely after the route's service window has closed. ([`RouteScheduleBeanServiceImpl.java#L204-L208`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteScheduleBeanServiceImpl.java#L204-L208), [`ScheduleForRouteAction.java#L86-L89`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForRouteAction.java#L86-L89))

**3c. No trips run on the requested date (but future service exists)**

After step 5, if no trips are active on the requested date but the route does have service scheduled on later dates (including the case where the requested date pre-dates the route's first service day), the system returns HTTP 200 with `code: 510`, `text: "NoServiceThatDay"`, and a partial data body. The entry includes the route ID, the schedule date, and empty `serviceIds` and `stopTripGroupings` arrays. The references block contains the agency and route objects but no stops, trips, or stop times. ([`ScheduleForRouteAction.java#L90-L92`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForRouteAction.java#L90-L92))

**4a. `date` parameter cannot be parsed**

If the `date` value is not a valid `YYYY-MM-DD` string and not a pure decimal integer, the framework raises a type-conversion error. The system returns HTTP 400 with `code: 400` and a validation-error body.

## Suspected Defects

### Defects that affect the use case

**Both 510 responses return HTTP 200 instead of HTTP 510.**

[`ScheduleForRouteAction.java#L104-L108`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForRouteAction.java#L104-L108), [`ScheduleForRouteAction.java#L110-L114`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForRouteAction.java#L110-L114)

Both `setNoServiceResponse` and `setNoServiceThatDayResponse` return `new DefaultHttpHeaders()` without calling `.withStatus(510)`. Every other error response in `ApiActionSupport` (404, 400, 500) correctly propagates the status code to the HTTP layer. The two 510 paths do not. Clients that check the HTTP status code rather than the body's `code` field cannot distinguish a 510 error from a 200 success. The Go implementation should return HTTP 510 for both conditions.

**`setNoServiceResponse` silently discards the route schedule data it computed.**

[`ScheduleForRouteAction.java#L86-L89`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForRouteAction.java#L86-L89), [`ScheduleForRouteAction.java#L104-L108`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForRouteAction.java#L104-L108)

The "ServiceDateOutOfRange" path calls `factory.getResponse(routeSchedule)` to compute a full response object, passes it to `setNoServiceResponse(data)`, but the method signature accepts `data` only to discard it — the response is constructed with `null`. The "NoServiceThatDay" path (Extension 3c) does the same computation and *does* include the data. The inconsistency appears unintentional: the service-date-out-of-range case should probably include a partial data body similar to the no-service-that-day case, or at minimum not waste the computation. The Go implementation should decide whether to include a body for this condition.

### Implementation defects only

**`arrivalEnabled` and `departureEnabled` use time-value comparisons instead of GTFS pickup/drop-off type.**

[`RouteScheduleBeanServiceImpl.java#L392-L395`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteScheduleBeanServiceImpl.java#L392-L395)

`arrivalEnabled` is set to `stopTimeEntry.getArrivalTime() > 0` and `departureEnabled` to `stopTimeEntry.getDepartureTime() > 0`. The correct check is whether the GTFS `drop_off_type` (for arrivals) and `pickup_type` (for departures) are not equal to 1 (no service). A trip whose first or last stop has a scheduled time of exactly midnight (0 seconds) would incorrectly receive `arrivalEnabled: false` or `departureEnabled: false`. In a Go reimplementation, these flags should be derived from the pickup/drop-off types.

**`serviceId` and `stopHeadsign` fields in stop-time entries are never populated.**

[`BeanFactoryV2.java#L761-L772`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L761-L772)

The `getStopTime(StopTimeInstanceBeanExtendedWithStopId)` factory method transfers `arrivalTime`, `departureTime`, `arrivalEnabled`, `departureEnabled`, `tripId`, and `stopId` to the output bean, but never sets `serviceId` or `stopHeadsign`. Both fields serialise as empty strings in every response. A clean Go implementation need not include these fields.

**`TripId` field name is capitalised incorrectly in `TripWithStopTimesV2Bean`.**

[`TripWithStopTimesV2Bean.java#L28`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/TripWithStopTimesV2Bean.java#L28)

The private field is named `TripId` (capital T). Jackson derives the JSON property name from the getter `getTripId()`, producing the correct `tripId` (lowercase) in JSON. However, XML serialisers that use field-name access would produce `TripId` (capital T), which diverges from JSON consumers' expectations. The Go implementation should use `tripId` (lowercase) consistently.

## Open Questions

None.

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Combined route ID in {agencyId}_{routeId} format, encoded in the URL path"
    },
    "date": {
      "type": "string",
      "description": "Service date in YYYY-MM-DD format, or a Unix epoch millisecond value as a decimal string. Defaults to the current date."
    },
    "key": {
      "type": "string",
      "description": "API authentication key"
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "Whether to include the references block. Note: for this endpoint the references block is always populated regardless of this value."
    }
  }
}
```

**`id`** — Combined route identifier in `{agencyId}_{routeId}` form (e.g., `1_102718`). The agency ID is everything before the first underscore; the route entity ID is everything after. Required; absence causes a 404 at the routing layer before the action is reached.

**`date`** — The service date for which to retrieve the schedule. Accepts `YYYY-MM-DD` (e.g., `2026-05-08`) or a raw decimal millisecond epoch value. Defaults to the current wall-clock time, from which the system derives today's service date. An invalid format causes HTTP 400.

**`key`** — API key for client identification. Required by the framework; requests without a key may fail authentication depending on server configuration.

**`includeReferences`** — Controls whether the response envelope includes a references block. For `schedule-for-route` the references are assembled unconditionally in the response-construction layer; this parameter has no observable effect on the output.

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "version": { "type": "integer" },
    "code": { "type": "integer" },
    "text": { "type": "string" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "data": { "type": "object" }
  }
}
```

**`version`** — API version number (always `2` for this endpoint).

**`code`** — Response status: `200` on success, `404` if the route is not found, `510` if no service is available for the requested date.

**`text`** — Human-readable status: `"OK"`, `"resource not found"`, `"ServiceDateOutOfRange"`, or `"NoServiceThatDay"`.

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`data`** — Present on `200` responses and on `510 NoServiceThatDay` responses. Absent (null) on `404` and `510 ServiceDateOutOfRange` responses.

---

### data.entry

```json
{
  "type": "object",
  "properties": {
    "routeId":          { "type": "string" },
    "scheduleDate":     { "type": "integer", "description": "Unix ms" },
    "serviceIds":       { "type": "array", "items": { "type": "string" } },
    "stopTripGroupings": { "type": "array", "items": { "$ref": "#/stopTripGrouping" } }
  }
}
```

**`data.entry.routeId`** — Combined route ID in `{agencyId}_{routeId}` form (mirrors the request parameter).

**`data.entry.scheduleDate`** — The service date for which the schedule was computed, expressed as a Unix millisecond timestamp representing midnight at the start of that day in the feed's configured timezone (America/Los_Angeles for the KCM feed). For example, 2026-05-08 in PDT (UTC-7) is returned as `1778223600000`.

**`data.entry.serviceIds`** — List of combined service IDs (`{agencyId}_{serviceId}`) for service calendars that are active on the requested date and include trips for this route. Empty when no trips run on that date. These correspond to GTFS `service_id` values prefixed with the agency ID.

**`data.entry.stopTripGroupings`** — One entry per direction of travel observed among the active trips. See `data.entry.stopTripGroupings[]` below.

---

### data.entry.stopTripGroupings[]

```json
{
  "type": "object",
  "properties": {
    "directionId":       { "type": "string" },
    "tripHeadsigns":     { "type": "array", "items": { "type": "string" } },
    "stopIds":           { "type": "array", "items": { "type": "string" } },
    "tripIds":           { "type": "array", "items": { "type": "string" } },
    "tripsWithStopTimes": { "type": "array", "items": { "$ref": "#/tripWithStopTimes" } }
  }
}
```

**`data.entry.stopTripGroupings[].directionId`** — GTFS direction identifier for this group (`"0"` or `"1"`). The meaning of each direction is agency-defined. Serialised as a string, not an integer.

**`data.entry.stopTripGroupings[].tripHeadsigns`** — Array of distinct headsign strings observed across all trips in this direction. Most routes yield a single headsign; routes with short-turn variants or multiple destinations may yield several. The set is unordered. If a trip has no recorded headsign, the name of its last stop is used as a fallback.

**`data.entry.stopTripGroupings[].stopIds`** — Ordered list of combined stop IDs (`{agencyId}_{stopId}`) representing the canonical stop sequence for this direction. The ordering is derived by a topological sort across all trip variants; it approximates travel order for linear routes but may be imprecise for loops or branching routes.

**`data.entry.stopTripGroupings[].tripIds`** — List of combined trip IDs (`{agencyId}_{tripId}`) for all trips operating in this direction on the requested date. Sorted in ascending lexicographic order by the combined ID string.

**`data.entry.stopTripGroupings[].tripsWithStopTimes`** — Per-trip stop-time detail, one entry per trip ID. See `data.entry.stopTripGroupings[].tripsWithStopTimes[]` below.

---

### data.entry.stopTripGroupings[].tripsWithStopTimes[]

```json
{
  "type": "object",
  "properties": {
    "tripId":    { "type": "string" },
    "stopTimes": { "type": "array", "items": { "$ref": "#/scheduleStopTime" } }
  }
}
```

**`data.entry.stopTripGroupings[].tripsWithStopTimes[].tripId`** — Combined trip ID (`{agencyId}_{tripId}`) identifying this trip.

**`data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes`** — Ordered list of scheduled stop times for this trip. Each entry describes one stop call; stops appear in the order the vehicle visits them. See `data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes[]` below.

---

### data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes[]

```json
{
  "type": "object",
  "properties": {
    "stopId":          { "type": "string" },
    "tripId":          { "type": "string" },
    "arrivalTime":     { "type": "integer" },
    "departureTime":   { "type": "integer" },
    "arrivalEnabled":  { "type": "boolean" },
    "departureEnabled":{ "type": "boolean" },
    "stopHeadsign":    { "type": "string" },
    "serviceId":       { "type": "string" }
  }
}
```

**`data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes[].stopId`** — Combined stop ID (`{agencyId}_{stopId}`) for the stop at this call.

**`data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes[].tripId`** — Combined trip ID repeated at each stop-time entry.

**`data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes[].arrivalTime`** — Scheduled arrival time expressed as seconds elapsed since midnight of the service date in the feed's timezone. For trips that cross midnight, values exceed 86 400.

**`data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes[].departureTime`** — Scheduled departure time, same unit as `arrivalTime`. To compute the absolute Unix millisecond time of either value: `scheduleDate + time * 1000`.

**`data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes[].arrivalEnabled`** — `true` when passengers may alight at this stop on this trip. See Suspected Defects for a caveat about midnight times.

**`data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes[].departureEnabled`** — `true` when passengers may board at this stop on this trip. See Suspected Defects for the same caveat.

**`data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes[].stopHeadsign`** — Per-stop headsign override. Always an empty string in current responses; the field is defined in the schema but never populated by the Java implementation.

**`data.entry.stopTripGroupings[].tripsWithStopTimes[].stopTimes[].serviceId`** — The combined service ID for this stop time. Always an empty string in current responses; the field is defined in the schema but never populated by the Java implementation.

---

### data.references

The references block contains full resolved objects for all entities cited in the entry. For `schedule-for-route` the references are always populated regardless of the `includeReferences` request parameter.

```json
{
  "type": "object",
  "properties": {
    "agencies":   { "type": "array" },
    "routes":     { "type": "array" },
    "stops":      { "type": "array" },
    "trips":      { "type": "array" },
    "situations": { "type": "array" },
    "stopTimes":  { "type": "array" }
  }
}
```

**`data.references.agencies`** — Full agency objects for all agencies that operate trips on this route. Each entry follows the standard agency structure (id, name, timezone, url, phone, etc.).

**`data.references.routes`** — Full route objects for the queried route (and any additional routes referenced by the stops, via the stop-to-route membership list). Each stop in `references.stops` carries a `routes` array listing the routes that serve it, which may introduce additional route entries here.

**`data.references.stops`** — Full stop objects for every stop called by any active trip on the route. Each stop object includes its coordinates, name, code, direction label, and a list of routes serving it. If a stop has a parent station, that station is also included.

**`data.references.trips`** — Full trip objects for every active trip, including trip headsign, direction ID, service ID, block ID, shape ID, and route short name.

**`data.references.situations`** — Service alert objects for alerts that apply to the queried route or any of its stops. Empty array when no alerts are active.

**`data.references.stopTimes`** — Flat list of all scheduled stop-time instances across all active trips, using the same per-stop structure as `tripsWithStopTimes[].stopTimes[]`. This list duplicates the stop times that are already present within `stopTripGroupings[].tripsWithStopTimes[].stopTimes[]`; both representations contain the same data.
