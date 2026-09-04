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
- The references block contains full objects for all agencies, routes, stops, and trips referenced by the entry. Legacy Java also includes service-alert situation objects that apply to the queried route or its stops, even though the entry has no `situationIds` field. Maglev intentionally does not preserve those ambient situation references; see [Implementation Decisions](#implementation-decisions) and [[Situation-References]].

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
6. If no trips are active on the requested date and no service for this route is scheduled on any future effective service date, the legacy Java implementation returns a service-date-out-of-range response (see Extension 3b). Maglev intentionally uses a planned service-span rule instead; see [Implementation Decisions](#implementation-decisions).
7. If no trips are active on the requested date but a future effective service date exists, the legacy Java implementation returns a no-service-that-day response (see Extension 3c). Maglev intentionally uses a planned service-span rule instead; see [Implementation Decisions](#implementation-decisions).
8. For each direction group the system derives a canonical stop order using a topological sort across all collected stop sequences for that direction. This heuristic works well for simple linear routes but may produce an imperfect ordering for loops or routes with complex branching. ([`RouteScheduleBeanServiceImpl.java#L411-L436`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteScheduleBeanServiceImpl.java#L411-L436))
9. Trip IDs within each direction group are sorted in ascending order by their combined ID string. ([`BeanFactoryV2.java#L977`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L977))
10. Stop times within each direction group are sorted by trip ID, then by arrival time within each trip, and then assembled into per-trip `tripsWithStopTimes` entries. ([`BeanFactoryV2.java#L961-L983`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L961-L983))
11. The references block is populated with all agencies, routes, stops, trips, and a flat list of all stop-time instances. Legacy Java also populates `references.situations` with service alerts that apply to the queried route or any of its stops, despite there being no `situationIds` field in `data.entry`; Maglev intentionally differs and leaves `references.situations` empty for this endpoint under the shared [[Situation-References]] rule.
12. The system returns HTTP 200 with the complete route schedule entry.

## Extensions

**3a. Route not found**

At step 3, if the route collection ID is not present in the transit graph, the system returns HTTP 404 with `code: 404` and no data body.

**3b. Date is outside the route's service span / no later effective service**

After step 5, the legacy Java implementation checks whether any effective service date exists after the requested date. If no block trips have any scheduled service on the requested date *and* no later effective service date exists, it returns HTTP 200 with `code: 510`, `text: "ServiceDateOutOfRange"`, and no data body. This primarily occurs when querying a date after the route's service window has closed, but the future-effective-date rule can also classify a date inside a planned service span as out of range if every later date has been removed by exceptions. ([`RouteScheduleBeanServiceImpl.java#L204-L208`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteScheduleBeanServiceImpl.java#L204-L208), [`RouteScheduleBeanServiceImpl.java#L233-L241`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteScheduleBeanServiceImpl.java#L233-L241), [`ExtendedCalendarServiceImpl.java#L139-L164`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/ExtendedCalendarServiceImpl.java#L139-L164), [`ScheduleForRouteAction.java#L86-L89`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForRouteAction.java#L86-L89))

Maglev intentionally differs for this distinction: `ServiceDateOutOfRange` means the requested date is outside the route's planned service span, either before the first planned service date or after the last planned service date. See [Implementation Decisions](#implementation-decisions).

**3c. No trips run on the requested date**

After step 5, if no trips are active on the requested date but a later effective service date exists, the legacy Java implementation returns HTTP 200 with `code: 510`, `text: "NoServiceThatDay"`, and a partial data body. Under this legacy rule, a date before the route's first service day is also classified as `NoServiceThatDay` when later service exists. The entry includes the route ID, the schedule date, and empty `serviceIds` and `stopTripGroupings` arrays. The references block contains the agency and route objects but no stops, trips, or stop times. ([`ScheduleForRouteAction.java#L90-L92`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForRouteAction.java#L90-L92))

Maglev intentionally differs for this distinction: `NoServiceThatDay` means the requested date is inside the route's planned service span, but no trips operate on that date after applying calendar exceptions. See [Implementation Decisions](#implementation-decisions).

**4a. `date` parameter cannot be parsed**

If the `date` value is not a valid `YYYY-MM-DD` string and not a pure decimal integer, the framework raises a type-conversion error. The system returns HTTP 400 with `code: 400` and a validation-error body.

## Suspected Defects

### Defects that affect the use case

**The empty-schedule reason uses an asymmetric future-effective-service rule.**

[`RouteScheduleBeanServiceImpl.java#L204-L208`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteScheduleBeanServiceImpl.java#L204-L208), [`RouteScheduleBeanServiceImpl.java#L233-L241`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteScheduleBeanServiceImpl.java#L233-L241), [`ExtendedCalendarServiceImpl.java#L139-L164`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/ExtendedCalendarServiceImpl.java#L139-L164)

Legacy Java distinguishes `NoServiceThatDay` from `ServiceDateOutOfRange` by asking whether any effective service date exists after the requested date. That makes the rule asymmetric: a date before the route starts can be reported as `NoServiceThatDay`, while a date inside the planned service span can be reported as `ServiceDateOutOfRange` if every later planned date is removed by `calendar_dates.exception_type = 2`. The intended rider-facing distinction is simpler: `ServiceDateOutOfRange` means the requested date is outside the route's planned service span, while `NoServiceThatDay` means the date is in span but no trips operate that day. Maglev's intended contract corrects this; see [Implementation Decisions](#implementation-decisions).

**Both 510 responses return HTTP 200 instead of HTTP 510.**

[`ScheduleForRouteAction.java#L104-L108`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForRouteAction.java#L104-L108), [`ScheduleForRouteAction.java#L110-L114`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForRouteAction.java#L110-L114)

Both `setNoServiceResponse` and `setNoServiceThatDayResponse` return `new DefaultHttpHeaders()` without calling `.withStatus(510)`. Every other error response in `ApiActionSupport` (404, 400, 500) correctly propagates the status code to the HTTP layer. The two 510 paths do not. Clients that check the HTTP status code rather than the body's `code` field cannot distinguish a 510 error from a 200 success.

**`setNoServiceResponse` silently discards the route schedule data it computed.**

[`ScheduleForRouteAction.java#L86-L89`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForRouteAction.java#L86-L89), [`ScheduleForRouteAction.java#L104-L108`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForRouteAction.java#L104-L108)

The "ServiceDateOutOfRange" path calls `factory.getResponse(routeSchedule)` to compute a full response object, passes it to `setNoServiceResponse(data)`, but the method signature accepts `data` only to discard it — the response is constructed with `null`. The "NoServiceThatDay" path (Extension 3c) does the same computation and *does* include the data. The inconsistency appears unintentional: the service-date-out-of-range case should probably include a partial data body similar to the no-service-that-day case, or at minimum not waste the computation.

### Implementation defects only

**`arrivalEnabled` and `departureEnabled` use time-value comparisons instead of GTFS pickup/drop-off type.**

[`RouteScheduleBeanServiceImpl.java#L392-L395`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RouteScheduleBeanServiceImpl.java#L392-L395)

`arrivalEnabled` is set to `stopTimeEntry.getArrivalTime() > 0` and `departureEnabled` to `stopTimeEntry.getDepartureTime() > 0`. GTFS defines pickup and drop-off availability via `pickup_type` and `drop_off_type` respectively, which this implementation does not consult. A trip whose first or last stop has a scheduled time of exactly midnight (0 seconds) would receive `arrivalEnabled: false` or `departureEnabled: false` regardless of what the GTFS data specifies. The intended check was probably whether `drop_off_type` (for arrivals) and `pickup_type` (for departures) are not equal to 1 (no service).

**`serviceId` and `stopHeadsign` fields in stop-time entries are never populated.**

[`BeanFactoryV2.java#L761-L772`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L761-L772)

The `getStopTime(StopTimeInstanceBeanExtendedWithStopId)` factory method transfers `arrivalTime`, `departureTime`, `arrivalEnabled`, `departureEnabled`, `tripId`, and `stopId` to the output bean, but never sets `serviceId` or `stopHeadsign`. Both fields serialise as empty strings in every response. The intended behaviour was probably to populate them from their respective data sources.

**`TripId` field name is capitalised incorrectly in `TripWithStopTimesV2Bean`.**

[`TripWithStopTimesV2Bean.java#L28`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/TripWithStopTimesV2Bean.java#L28)

The private field is named `TripId` (capital T). Jackson derives the JSON property name from the getter `getTripId()`, producing the correct `tripId` (lowercase) in JSON. However, XML serialisers that use field-name access would produce `TripId` (capital T), which diverges from JSON consumers' expectations. The intended serialisation was probably `tripId` (lowercase) in all output formats.

## Implementation Decisions

**Extension 4a — validation-error body wraps field errors in OBA response envelope**

The legacy Java implementation returns a bare Struts body `{"fieldErrors":{"date":[...]}}` with no OBA envelope fields. Maglev returns the standard OBA response envelope with field errors nested under `data`, matching the format used by `validationErrorResponse` across all other endpoints. This is a deliberate deviation from the legacy body format to maintain a consistent response structure across all Maglev error responses (see DC-4a).

**Extensions 3b/3c — "no service" responses use `code: 200` with descriptive `text`, not `code: 510` (DC-2)**

The legacy Java implementation returns `code: 510` in the response body for both `ServiceDateOutOfRange` and `NoServiceThatDay`. HTTP 510 ("Not Extended") is a protocol-level negotiation code unrelated to transit scheduling, and the legacy implementation does not propagate it to the HTTP transport layer anyway (both responses are HTTP 200). Maglev returns `code: 200` in the body (matching the HTTP status) and uses the `text` field (`"ServiceDateOutOfRange"` or `"NoServiceThatDay"`) to communicate the reason. These are not error conditions — the route exists, the request succeeded, the schedule for the requested date is simply empty. Static analysis of all three clients (Wayfinder + JS SDK, iOS, Android) confirms none inspect `code: 510` or branch on the body `code` field for this endpoint.

**Extension 3b — `ServiceDateOutOfRange` includes a partial data body (DC-3)**

The legacy Java implementation returns `data: null` for `ServiceDateOutOfRange` while returning a partial data body (route ID, schedule date, empty arrays, agency+route references) for `NoServiceThatDay`. Since DC-2 reclassifies both as successful responses with an empty schedule, Maglev returns the same partial data body for both cases. This makes the two responses structurally consistent — the only difference is the `text` label — and gives clients useful context (route metadata) even when no trips run. For iOS specifically, this eliminates a `DecodingError` that occurred when `data` was null, allowing the app to display an empty schedule instead of a generic error.

**Extensions 3b/3c — empty-schedule reason uses planned service-span semantics (SD-01)**

Legacy Java distinguishes `NoServiceThatDay` from `ServiceDateOutOfRange` by asking whether any effective service date exists after the requested service date. That makes the rule asymmetric: dates before a route starts can be reported as `NoServiceThatDay`, while dates inside a planned service span can be reported as `ServiceDateOutOfRange` if every later date is removed by `calendar_dates.exception_type = 2`.

Maglev's intended contract instead uses planned service-span semantics. `ServiceDateOutOfRange` means the requested date is outside the route's known planned service span, either before the first planned service date or after the last planned service date. `NoServiceThatDay` means the requested date is inside that span, but no trips operate on that date after applying calendar exceptions.

This treats the service span symmetrically and reflects the product distinction a rider sees: outside the route's schedule era versus an in-scope date with no scheduled trips. Removal exceptions can represent holidays, staffing cancellations, or other date-specific removals, and do not by themselves mean the route is outside its planned service span.

Client impact is expected to be low: the response shape is unchanged, and the affected value is the wrapper `text` for empty-schedule edge cases. Static review of Wayfinder, the JS SDK, iOS, and Android found no client dependency on the exact `NoServiceThatDay` vs `ServiceDateOutOfRange` labels for this endpoint.

Implementation note: this entry defines the intended Maglev contract. Any Maglev code path that still follows the legacy future-effective-service rule for before-start dates or all-later-dates-removed cases should be treated as needing follow-up implementation/test coverage.

**Ambient `references.situations` is omitted because no `situationIds` field references those alerts (SD-02)**

Legacy Java is unusual for this endpoint: after building the static route schedule, it queries service alerts for the route and every stop in the schedule (`RouteScheduleBeanServiceImpl.addSituations`) and writes them directly into `data.references.situations` (`BeanFactoryV2.getRouteSchedule`). `RouteScheduleV2Bean` itself has no `situationIds` field, so these alert objects are ambient reference-block data rather than dereference targets. The alert query is also not scoped to the requested schedule date in the way the static schedule is; it uses the service-alert query's default time, which means publication-window filtering is not applied. See [[Situation-References]] for citations.

Maglev intentionally does not preserve this one-off legacy behavior. `schedule-for-route` is a planning-oriented static schedule response, and active or published real-time/service-alert state should not appear in `references.situations` unless the response data explicitly includes IDs that point to those situations. Callers that need current alerts for a route should use an alert/situation-oriented endpoint or a real-time endpoint whose data section includes `situationIds`.

Client impact is expected to be low: no known client depends on the legacy ambient situation array for route schedules, and removing it makes `schedule-for-route` follow the shared invariant that reference-block situations exist only to resolve explicit `situationIds` values.

If this is reconsidered, the fallback is to reintroduce Java's ambient route/stop alert query and include those full objects in `references.situations`, but that would deliberately violate the shared [[Situation-References]] rule and should be documented as a new endpoint-specific exception.

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

**`includeReferences`** — Controls whether the response envelope includes a references block. In legacy Java `schedule-for-route` assembles references unconditionally in the response-construction layer, so this parameter has no observable effect on the output. Maglev follows the endpoint-specific reference behavior described above, but does not include ambient situation references; see [Implementation Decisions](#implementation-decisions) and [[Situation-References]].

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

**`code`** — Response status. Legacy Java uses `200` on success, `404` if the route is not found, and body `code: 510` for the two empty-schedule responses. Maglev intentionally uses body `code: 200` for both empty-schedule responses and carries the reason in `text`; see [Implementation Decisions](#implementation-decisions).

**`text`** — Human-readable status: `"OK"`, `"resource not found"`, `"ServiceDateOutOfRange"`, or `"NoServiceThatDay"`. For Maglev, `ServiceDateOutOfRange` means the requested date is outside the route's planned service span, while `NoServiceThatDay` means the date is inside that span but no trips operate; see [Implementation Decisions](#implementation-decisions).

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`data`** — Present on successful schedule responses. Legacy Java also includes a partial data body on `510 NoServiceThatDay` but returns null for `510 ServiceDateOutOfRange`. Maglev intentionally includes the same partial empty-schedule body for both empty-schedule reasons; see [Implementation Decisions](#implementation-decisions).

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

The references block contains full resolved objects for all entities cited in the entry. For legacy Java `schedule-for-route`, references are always populated regardless of the `includeReferences` request parameter, and `references.situations` is additionally populated with route/stop alerts that are not cited by any `situationIds` field. Maglev intentionally omits those ambient situation references; see [Implementation Decisions](#implementation-decisions) and [[Situation-References]].

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

**`data.references.situations`** — In legacy Java, service alert objects for alerts that apply to the queried route or any of its stops, even though `data.entry` has no `situationIds` field. Maglev intentionally returns an empty array for this endpoint because no situation IDs are explicitly referenced in the response data; see [Implementation Decisions](#implementation-decisions) and [[Situation-References]].

**`data.references.stopTimes`** — Flat list of all scheduled stop-time instances across all active trips, using the same per-stop structure as `tripsWithStopTimes[].stopTimes[]`. This list duplicates the stop times that are already present within `stopTripGroupings[].tripsWithStopTimes[].stopTimes[]`; both representations contain the same data.
