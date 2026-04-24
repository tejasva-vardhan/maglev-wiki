**DRAFT - DO NOT IMPLEMENT**

## Goal in Context

A rider wants to see a departure board for a specific stop — every approaching vehicle, its scheduled and predicted arrival and departure times, and any service disruptions — so they can decide when to travel. A secondary, more targeted goal is to retrieve a single arrival-and-departure entry for a specific trip instance at a stop, enabling a client to refresh one entry without reloading the entire board.

## Scope

OBA REST API, covering two endpoints:

- `GET /api/where/arrivals-and-departures-for-stop/{id}` — departure board for a stop (list)
- `GET /api/where/arrival-and-departure-for-stop/{id}` — single arrival/departure entry

## Level

User goal

## Primary Actor

Rider

## Stakeholders and Interests

- **Rider** — wants accurate, real-time departure information for a specific stop, including which vehicles are approaching, how late or early they are, and any service alerts that apply.

## Preconditions

- The stop ID is known and refers to a stop in the loaded transit data bundle.
- For the single-entry endpoint: the trip ID and service date of the specific arrival are known (typically obtained from a prior departure-board call).

## Minimal Guarantees

- The response envelope always contains a numeric HTTP status code, a version number, and the current server time.
- If the stop is unknown, HTTP 404 is returned with no data.
- If required parameters are absent, HTTP 400 is returned with a field-level error map.

## Success Guarantees

- The response contains one entry per scheduled or predicted vehicle visit to the stop that falls within the requested time window.
- Each entry carries scheduled times, predicted times (when real-time data is available), vehicle distance from the stop, the number of intermediate stops away, the trip's current status, and any applicable service alerts.
- The queried stop itself, nearby stops, referenced trips, routes, agencies, and service alerts are all included in the references block.

## Trigger

A client issues a `GET` request to one of the two endpoints.

## Main Success Scenario

### Departure board (`arrivals-and-departures-for-stop/{id}`)

1. The system resolves the stop from the path parameter `id`. If the stop is not found, the scenario ends with HTTP 404.
2. The system determines the reference time `T`: the value of the `time` parameter if supplied, otherwise the current server time.
3. The system determines two separate time windows:
   - For schedule-based trips: `[T − minutesBefore, T + minutesAfter]` (defaults: 5 min before, 35 min after).
   - For frequency-based trips: `[T − frequencyMinutesBefore, T + frequencyMinutesAfter]` (defaults: 2 min before, 30 min after).
4. The system retrieves all block instances that include a stop time at this stop within the union of both windows.
5. For each retrieved instance:
   a. If the agency's per-agency "hide schedule info" flag is set and the instance has no real-time prediction, the entry is skipped.
   b. The instance is tested against the appropriate window for its type (frequency-based or schedule-based). If neither its scheduled nor predicted arrival/departure time falls within that window, the entry is skipped.
   c. A departure board entry is assembled containing the scheduled and predicted times, arrival/departure enabled flags, sequence within the trip, block trip sequence, headsign, route names, service date, and distance/stop-count from the vehicle to this stop (derived from the vehicle's current block location).
   d. If the vehicle has a known block location, vehicle status (predicted vs. schedule-derived position, schedule deviation, position, vehicle ID, occupancy) is attached as a nested trip status object.
   e. Service alerts applicable to the stop call are attached.
   f. The entry is passed through the system-level and instance-level filter chains. If it does not match, it is discarded. (In standard deployments, this chain excludes entries whose GTFS-RT schedule relationship is SKIPPED.)
6. The entries are sorted by best arrival time: predicted arrival time if one is set, otherwise scheduled arrival time.
7. The system fetches all stops within 100 metres of the queried stop. These are included as nearby stop IDs.
8. The system fetches service alerts for the stop ID and for every route serving the stop (including routes with no current active service). Duplicate alerts are deduplicated by alert ID.
9. The response is assembled as an entry-with-references object. Routes referenced by the response entries are sorted by a custom route ordering defined at deployment time, with the primary agency's routes listed first. The response is returned with HTTP 200.

### Single entry (`arrival-and-departure-for-stop/{id}`)

1. The system validates that `tripId` and `serviceDate` are present. If either is absent, the scenario ends with HTTP 400 and a field-level error map.
2. The system determines the reference time `T` (same as above).
3. The system looks up the specific arrival-and-departure instance identified by the combination of stop ID, trip ID, and service date. `vehicleId` and `stopSequence`, when supplied, help disambiguate when a vehicle visits the same stop more than once within a trip. `stopSequence` also improves lookup performance.
4. If no matching instance is found, the scenario ends with HTTP 404.
5. The instance is assembled into a departure entry following the same logic as steps 5c–5e above.
6. If schedule deviation history is available for this trip instance, a histogram of historical deviations is added to the internal record (but is not surfaced in the serialised API response; see Suspected Defects).
7. The response is assembled as an entry-with-references object and returned with HTTP 200.

## Extensions

**2a. `time` parameter supplied as Unix milliseconds**

The time parameter accepts both Unix milliseconds and human-readable strings (handled by the DateTimeConverter). When 0 is passed, it is treated as an explicit request for the Unix epoch rather than as "no time given", because the query defaults to the current time at construction and there is no separate "time was not set" flag.

**5b-alt. Frequency-based trip scheduled times**

For a frequency-based trip (one defined by a headway rather than a fixed timetable), when a real-time predicted time is available, the scheduled arrival and departure times in the entry are overwritten with the predicted value. Clients that display "scheduled" time for frequency-based trips should use the predicted time directly rather than relying on a schedule/prediction distinction.

**5d-alt. Interlining — active trip differs from the entry's trip**

A block may contain trips on different routes. If the block is currently executing a trip that precedes the queried trip in the block sequence, the vehicle is counted because it will eventually serve this stop's trip. In this case, `tripStatus.activeTripId` is the trip the vehicle is currently on, while the entry's `tripId` is the trip that will call at this stop. This is expected behaviour and does not indicate a data error.

**5d-alt2. No block location available**

When no block location has been established for the instance (no real-time feed match and the vehicle has not been seen), the trip status sub-object is omitted. `predicted` remains false. `distanceFromStop` and `numberOfStopsAway` are computed from the scheduled position rather than a real-time position.

**5e-alt. Trip is cancelled**

If a trip's GTFS-RT schedule relationship is CANCELED, the entry's `status` field is set to `"canceled"` and `predicted` is false. In the list endpoint these entries pass through unless a server-side "hide canceled trips" flag is set, in which case they are excluded from the list silently. In the single-entry endpoint, a canceled trip can still be returned (with `predicted=false` and no trip status sub-object), unless the server flag is set, in which case the endpoint returns HTTP 404.

**8-alt. `includeReferences=false`**

When the caller passes `includeReferences=false`, the single-entry endpoint returns the full entry structure with empty reference arrays. The list endpoint crashes with an unhandled NullPointerException and returns HTTP 200 with body `null` instead of the expected data. See Suspected Defects.

---

## Suspected Defects

### Defects that affect the use case

**`BeanFactoryV2` — NullPointerException when `includeReferences=false` on list endpoint**

`BeanFactoryV2.getResponse(StopWithArrivalsAndDeparturesBean)` calls `response.getReferences().getRoutes().sort(…)` unconditionally. `ReferencesBean.getRoutes()` returns `null` when no routes have been added (which happens when `includeReferences=false` suppresses all reference additions). The resulting NullPointerException propagates out of the response builder — which is not wrapped in a try-catch in `ArrivalsAndDeparturesForStopAction.show()` — leaving `_response` unset. Struts2 serialises the null model as the four-byte JSON body `null`, returned with HTTP 200. The single-entry endpoint is unaffected because its response builder (`getResponse(ArrivalAndDepartureBean)`) does not call the route-sort code.

**Intended behaviour:** `includeReferences=false` should return the full data structure with empty (or absent) reference arrays, as it does for the single-entry endpoint.

---

**`ArrivalAndDepartureV2Bean.hasPredictedDepartureTime()` — checks wrong field**

```java
// ArrivalAndDepartureV2Bean, line 311
public boolean hasPredictedDepartureTime() {
    return this.predictedArrivalTime > 0;   // should be predictedDepartureTime
}
```

`hasPredictedDepartureTime()` tests `predictedArrivalTime` rather than `predictedDepartureTime`. `computeBestDepartureTime()` (line 367) delegates to this method, so it returns the predicted departure time only when there is a predicted arrival time — and falls back to the scheduled departure time whenever there is no predicted arrival, even if a predicted departure exists. Callers that use `computeBestDepartureTime()` to decide which time to display can show a stale scheduled departure time when a real prediction is available.

---

**`BeanFactoryV2.isSituationExcludedForApplication()` — always returns false**

```java
// BeanFactoryV2, line 1381
return !_applicationKey.contains(_applicationKey);
```

A string always contains itself, so this expression is always `false`, meaning no service alerts are ever excluded based on application key. The correct comparison would be `!applicationIds.contains(_applicationKey)`. The effect is that all service alerts are returned regardless of which application key the request was made with.

---

### Implementation defects only

**`ArrivalAndDepartureComparator.compare()` — integer overflow on timestamp subtraction**

```java
// ArrivalsAndDeparturesBeanServiceImpl, line 554
return (int) (t1 - t2);
```

Timestamps are Unix milliseconds. Casting their difference to `int` silently overflows when two times are more than roughly 24 days apart, producing a wrong sort order. The fix is to use `Long.compare(t1, t2)`.

---

**`ArrivalsAndDeparturesBeanServiceImpl.getArrivalsAndDeparturesByStopId()` — unused local variable**

Line 214 declares `HashSet<String> agenciesExcludingScheduled = query.getAgenciesExcludingScheduled()` but the variable is never read; the agency check on line 216 re-reads the set directly from `query`. This is dead code with no behavioural effect.

---

**`ArrivalsAndDeparturesForStopAction.show()` — dead code in `serviceDateFiltering` branch**

```java
// ArrivalsAndDeparturesForStopAction, line 121
if(serviceDateFilterOn && _query.getTime() == 0)
    _query.setTime(SystemTime.currentTimeMillis());
```

`ArrivalsAndDeparturesQueryBean` initialises `time` to `SystemTime.currentTimeMillis()` at construction, so `_query.getTime()` is never zero unless the caller explicitly passes `time=0` (the Unix epoch). The `serviceDateFiltering` branch intended to substitute the current time when no time was supplied, but has no effect because the bean already defaults to current time.

---

## Open Questions

None. All behaviours were resolved by static analysis and live server verification.

---

## Request Parameters

### `arrivals-and-departures-for-stop/{id}`

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Stop ID, embedded in the URL path."
    },
    "key": {
      "type": "string",
      "description": "API key."
    },
    "time": {
      "type": "integer",
      "description": "Unix ms. Reference time for the query. Defaults to server current time."
    },
    "minutesBefore": {
      "type": "integer",
      "default": 5,
      "description": "How many minutes before the reference time to include schedule-based arrivals."
    },
    "minutesAfter": {
      "type": "integer",
      "default": 35,
      "description": "How many minutes after the reference time to include schedule-based departures."
    },
    "frequencyMinutesBefore": {
      "type": "integer",
      "default": 2,
      "description": "Look-back window in minutes for frequency-based trips."
    },
    "frequencyMinutesAfter": {
      "type": "integer",
      "default": 30,
      "description": "Look-ahead window in minutes for frequency-based trips."
    },
    "version": {
      "type": "integer",
      "default": 2,
      "description": "API response version. Only version 2 is fully supported."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "Whether to populate the references block. Setting false causes a null response body — see Suspected Defects."
    }
  }
}
```

**`id`** — The stop ID as it appears in the transit data (agency-prefixed, e.g. `1_75403`). Encoded directly in the URL path.

**`key`** — API authentication key.

**`time`** — Reference time in Unix milliseconds. The window of arrivals/departures returned is relative to this time. Defaults to the current server time. Primarily useful for testing historical or future states.

**`minutesBefore`** — Width of the look-back window in minutes for schedule-based trips. An arrival with a scheduled (or predicted) arrival or departure time at or after `T − minutesBefore` and at or before `T + minutesAfter` is included.

**`minutesAfter`** — Width of the look-ahead window in minutes for schedule-based trips.

**`frequencyMinutesBefore`** — Separate look-back window applied only to frequency-based (headway) trips.

**`frequencyMinutesAfter`** — Separate look-ahead window applied only to frequency-based trips.

**`version`** — Response schema version. Must be 2 (the default) for a well-formed response.

**`includeReferences`** — When true (the default), the `references` block is populated with the full objects for every agency, route, stop, trip, and service alert referenced by ID elsewhere in the response. When false, the list endpoint returns a null body due to a defect; the single-entry endpoint returns empty reference arrays correctly.

---

### `arrival-and-departure-for-stop/{id}`

```json
{
  "type": "object",
  "required": ["id", "tripId", "serviceDate", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Stop ID, embedded in the URL path."
    },
    "tripId": {
      "type": "string",
      "description": "ID of the trip for the specific arrival."
    },
    "serviceDate": {
      "type": "integer",
      "description": "Unix ms. Midnight of the service date for the trip."
    },
    "vehicleId": {
      "type": "string",
      "description": "Vehicle ID. Optional; helps disambiguate when multiple vehicles serve the same trip instance."
    },
    "stopSequence": {
      "type": "integer",
      "default": -1,
      "description": "0-indexed position of this stop within the trip. -1 means unspecified. Improves disambiguation and lookup performance."
    },
    "time": {
      "type": "integer",
      "description": "Unix ms. Reference time for real-time status. Defaults to server current time."
    },
    "key": {
      "type": "string",
      "description": "API key."
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

**`id`** — Stop ID, embedded in the URL path.

**`tripId`** — Trip ID for the specific arrival. Required.

**`serviceDate`** — Unix milliseconds representing midnight of the service date for the trip. This anchors the trip to a specific operating day. Required.

**`vehicleId`** — When a trip is operated by a known vehicle, supplying the vehicle ID helps the system select the correct instance when the same trip runs under multiple vehicles on the same service date (unusual but possible with substitutions).

**`stopSequence`** — The 0-indexed position of this stop within the trip's stop-time sequence. Passing this value improves both disambiguation (for trips that visit the same stop more than once) and lookup speed. Defaults to −1 (unspecified).

**`time`** — Reference time for computing real-time status (schedule deviation, vehicle position). Defaults to current server time.

---

## Response Structure

The outer envelope is shared by both endpoints.

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

**`version`** — Response schema version number (typically 2).

**`code`** — HTTP status code mirrored inside the body (200, 400, 404, 500).

**`text`** — Human-readable status string ("OK", "resource not found", etc.).

**`currentTime`** — Server time at the moment the response was generated, in Unix milliseconds.

**`data`** — The payload object. Structure differs between the two endpoints; see below.

---

### `data` (arrivals-and-departures-for-stop)

```json
{
  "type": "object",
  "properties": {
    "entry": {
      "type": "object",
      "properties": {
        "stopId":                { "type": "string" },
        "arrivalsAndDepartures": { "type": "array",  "items": { "type": "object" } },
        "nearbyStopIds":         { "type": "array",  "items": { "type": "string" } },
        "situationIds":          { "type": "array",  "items": { "type": "string" } }
      }
    },
    "references": { "type": "object" }
  }
}
```

**`data.entry.stopId`** — The stop ID that was queried.

**`data.entry.arrivalsAndDepartures[]`** — Ordered list of arrival-and-departure entries (see `arrivalsAndDepartures[]` schema below). Sorted by best arrival time: predicted arrival time when one is set, otherwise scheduled arrival time.

**`data.entry.nearbyStopIds[]`** — IDs of stops within 100 metres of the queried stop. Useful for helping a rider see nearby boarding locations. The referenced stop objects are included in `data.references.stops`.

**`data.entry.situationIds[]`** — IDs of service alerts applicable to this stop and to all routes currently or statically serving it. The referenced alert objects are included in `data.references.situations`.

---

### `data` (arrival-and-departure-for-stop)

```json
{
  "type": "object",
  "properties": {
    "entry":      { "type": "object" },
    "references": { "type": "object" }
  }
}
```

**`data.entry`** — A single arrival-and-departure object (same schema as one element of `data.entry.arrivalsAndDepartures[]` above).

---

### `data.entry.arrivalsAndDepartures[]`

```json
{
  "type": "object",
  "properties": {
    "routeId":                    { "type": "string" },
    "tripId":                     { "type": "string" },
    "serviceDate":                { "type": "integer", "description": "Unix ms" },
    "vehicleId":                  { "type": "string" },
    "stopId":                     { "type": "string" },
    "stopSequence":               { "type": "integer" },
    "totalStopsInTrip":           { "type": "integer" },
    "blockTripSequence":          { "type": "integer" },
    "routeShortName":             { "type": "string" },
    "routeLongName":              { "type": "string" },
    "tripHeadsign":               { "type": "string" },
    "arrivalEnabled":             { "type": "boolean" },
    "departureEnabled":           { "type": "boolean" },
    "scheduledArrivalTime":       { "type": "integer", "description": "Unix ms" },
    "scheduledDepartureTime":     { "type": "integer", "description": "Unix ms" },
    "scheduledArrivalInterval":   { "type": "object" },
    "scheduledDepartureInterval": { "type": "object" },
    "predictedArrivalTime":       { "type": "integer", "description": "Unix ms; 0 when no prediction" },
    "predictedDepartureTime":     { "type": "integer", "description": "Unix ms; 0 when no prediction" },
    "predictedArrivalInterval":   { "type": "object" },
    "predictedDepartureInterval": { "type": "object" },
    "frequency":                  { "type": "object" },
    "predicted":                  { "type": "boolean" },
    "lastUpdateTime":             { "type": "integer", "description": "Unix ms" },
    "distanceFromStop":           { "type": "number" },
    "numberOfStopsAway":          { "type": "integer" },
    "status":                     { "type": "string" },
    "occupancyStatus":            { "type": "string" },
    "historicalOccupancy":        { "type": "string" },
    "scheduledTrack":             { "type": "string" },
    "actualTrack":                { "type": "string" },
    "tripStatus":                 { "type": "object" },
    "situationIds":               { "type": "array", "items": { "type": "string" } }
  }
}
```

**`routeId`** — The route ID for the trip calling at this stop. Referenced in `data.references.routes`.

**`tripId`** — The trip ID that will call at this stop. May differ from `tripStatus.activeTripId` when the vehicle is currently executing a preceding trip in the same block (interlining). Referenced in `data.references.trips`.

**`serviceDate`** — Unix milliseconds for midnight of the service date on which the trip operates. For trips that run past midnight, this is midnight of the previous calendar day.

**`vehicleId`** — The vehicle currently operating this block instance, if a real-time match has been established. Empty string when unknown.

**`stopId`** — The stop at which this arrival/departure is scheduled. For this endpoint, always the queried stop. Referenced in `data.references.stops`.

**`stopSequence`** — 0-indexed position of this stop within the trip's ordered stop sequence (internally computed, not the GTFS `stop_sequence` field). The first stop has sequence 0; the last stop has sequence `totalStopsInTrip − 1`.

**`totalStopsInTrip`** — Total number of stop visits in this trip, counting each visit separately if a stop is visited more than once.

**`blockTripSequence`** — 0-indexed position of this trip within the block's ordered trip sequence. Compare to `tripStatus.blockTripSequence` to determine whether the vehicle has yet to reach this trip, is currently on it, or has passed it.

**`routeShortName`** — The route's short display name (e.g. "67"). Resolved from the stop-time narrative, then the trip narrative, then the route's own short name, in that priority order. May be absent for routes with only a long name.

**`routeLongName`** — The route's long display name.

**`tripHeadsign`** — Destination text for this specific service call. Resolved from the stop-time headsign first, then the trip headsign.

**`arrivalEnabled`** — True if a rider can alight from this vehicle at this stop. False for the first stop of the trip (the vehicle originates there; passengers board but do not arrive).

**`departureEnabled`** — True if a rider can board this vehicle at this stop. False for the last stop of the trip (the vehicle terminates there).

**`scheduledArrivalTime`** — Scheduled arrival time in Unix milliseconds. For frequency-based trips when a real-time prediction is available, this field is overwritten with the predicted time.

**`scheduledDepartureTime`** — Scheduled departure time in Unix milliseconds. Subject to the same frequency overwrite as `scheduledArrivalTime`.

**`scheduledArrivalInterval`** — An optional `{ "from": <ms>, "to": <ms> }` time range representing the confidence window on the scheduled arrival. Present only when the service uses range-based scheduling.

**`scheduledDepartureInterval`** — Same as `scheduledArrivalInterval` but for departure.

**`predictedArrivalTime`** — Real-time predicted arrival time in Unix milliseconds. `0` when no prediction is available.

**`predictedDepartureTime`** — Real-time predicted departure time in Unix milliseconds. `0` when no prediction is available.

**`predictedArrivalInterval`** — Optional confidence range for the predicted arrival.

**`predictedDepartureInterval`** — Optional confidence range for the predicted departure.

**`frequency`** — Populated only for frequency-based (headway) trips. Contains `startTime`, `endTime` (seconds since midnight of service date), `headway` (seconds between departures), and `exactTimes` (boolean: true if departures are at exact multiples of the headway). Absent for schedule-based trips.

**`predicted`** — True if a real-time block location was available when computing this entry. False for cancelled trips even when a location was previously known.

**`lastUpdateTime`** — Unix milliseconds of the most recent real-time update from the vehicle. `0` when no update has been received.

**`distanceFromStop`** — Distance in metres from the vehicle's current position to this stop along the block path. Positive: vehicle has not yet reached this stop. Negative: vehicle has already passed this stop. Computed from real-time position when available, otherwise from the scheduled position.

**`numberOfStopsAway`** — Number of stops between the vehicle's next stop and this stop, not counting the vehicle's current next stop. Negative when the vehicle is beyond this stop in the block.

**`status`** — Operational status string. `"default"` indicates normal operation. `"canceled"` indicates the trip has been cancelled. Other values are set from real-time data.

**`occupancyStatus`** — Current vehicle occupancy as a GTFS-RT `OccupancyStatus` name string (e.g. `"MANY_SEATS_AVAILABLE"`, `"FULL"`). Empty string when unknown.

**`historicalOccupancy`** — Historical average occupancy for this trip at this stop and service date, derived from ridership data. Same vocabulary as `occupancyStatus`. Empty string when not available.

**`scheduledTrack`** — Scheduled platform or track identifier. Empty string when not applicable or unknown.

**`actualTrack`** — Actual (real-time) platform or track identifier. Empty string when not applicable or unknown.

**`tripStatus`** — Real-time status of the vehicle serving this block instance. Absent when no block location has been established. See `tripStatus` schema below.

**`situationIds[]`** — IDs of service alerts applicable to this specific stop call (stop + trip + vehicle). Referenced in `data.references.situations`.

---

### `data.entry.arrivalsAndDepartures[].tripStatus`

```json
{
  "type": "object",
  "properties": {
    "activeTripId":                  { "type": "string" },
    "blockTripSequence":             { "type": "integer" },
    "serviceDate":                   { "type": "integer", "description": "Unix ms" },
    "frequency":                     { "type": "object" },
    "scheduledDistanceAlongTrip":    { "type": "number" },
    "totalDistanceAlongTrip":        { "type": "number" },
    "position":                      { "type": "object", "properties": { "lat": { "type": "number" }, "lon": { "type": "number" } } },
    "orientation":                   { "type": "number" },
    "closestStop":                   { "type": "string" },
    "closestStopTimeOffset":         { "type": "integer" },
    "nextStop":                      { "type": "string" },
    "nextStopTimeOffset":            { "type": "integer" },
    "phase":                         { "type": "string" },
    "status":                        { "type": "string" },
    "predicted":                     { "type": "boolean" },
    "lastUpdateTime":                { "type": "integer", "description": "Unix ms" },
    "lastLocationUpdateTime":        { "type": "integer", "description": "Unix ms" },
    "lastKnownLocation":             { "type": "object", "properties": { "lat": { "type": "number" }, "lon": { "type": "number" } } },
    "lastKnownDistanceAlongTrip":    { "type": "number" },
    "lastKnownOrientation":          { "type": "number" },
    "distanceAlongTrip":             { "type": "number" },
    "scheduleDeviation":             { "type": "integer" },
    "vehicleId":                     { "type": "string" },
    "occupancyStatus":               { "type": "string" },
    "occupancyCount":                { "type": "integer" },
    "occupancyCapacity":             { "type": "integer" },
    "vehicleFeatures":               { "type": "array", "items": { "type": "string" } },
    "situationIds":                  { "type": "array", "items": { "type": "string" } }
  }
}
```

**`tripStatus.activeTripId`** — The trip the vehicle is currently executing. When the vehicle is between trips in a block or on a preceding interlined trip, this differs from the parent entry's `tripId`.

**`tripStatus.blockTripSequence`** — Position of the active trip within the block. Compare to the parent entry's `blockTripSequence` to determine relative position.

**`tripStatus.serviceDate`** — Unix milliseconds for midnight of the active trip's service date.

**`tripStatus.frequency`** — Frequency information for the active trip if it is headway-based; absent otherwise.

**`tripStatus.scheduledDistanceAlongTrip`** — How far along the active trip the vehicle is scheduled to have travelled, in metres, at the reference time.

**`tripStatus.totalDistanceAlongTrip`** — Total path length of the active trip in metres.

**`tripStatus.position`** — Current vehicle position. When real-time data is available, reflects the actual GPS position; otherwise extrapolated from the scheduled position. Absent if position is unknown.

**`tripStatus.orientation`** — Vehicle heading in degrees, where 0° = east, 90° = north, 180° = west, 270° = south.

**`tripStatus.closestStop`** — ID of the stop nearest to the vehicle's current position, whether it has been passed or not. Referenced in `data.references.stops`.

**`tripStatus.closestStopTimeOffset`** — Seconds from the vehicle's current position to the closest stop. Negative if the stop has already been passed.

**`tripStatus.nextStop`** — ID of the next stop the vehicle has not yet departed. Absent when the vehicle has passed the last stop on the active trip. Referenced in `data.references.stops`.

**`tripStatus.nextStopTimeOffset`** — Seconds until the vehicle reaches `nextStop` according to the schedule or real-time estimate.

**`tripStatus.phase`** — Journey phase string describing the vehicle's current operational state.

**`tripStatus.status`** — Status modifier string for the active trip (e.g. `"default"`, `"canceled"`).

**`tripStatus.predicted`** — True if the position information is derived from real-time GPS data. False if it is schedule-derived.

**`tripStatus.lastUpdateTime`** — Unix milliseconds of the most recent real-time record from the vehicle (of any type). `0` if no update has been received.

**`tripStatus.lastLocationUpdateTime`** — Unix milliseconds of the most recent real-time record that included a location update. May be earlier than `lastUpdateTime` if the most recent updates were non-positional.

**`tripStatus.lastKnownLocation`** — The last GPS-confirmed position, without any extrapolation. Absent if no GPS fix has been received.

**`tripStatus.lastKnownDistanceAlongTrip`** — Distance along the active trip at the time of the last known location, in metres.

**`tripStatus.lastKnownOrientation`** — Heading at the time of the last known location.

**`tripStatus.distanceAlongTrip`** — Current distance along the active trip in metres, potentially extrapolated from the last known position. Absent if not computable.

**`tripStatus.scheduleDeviation`** — Current deviation from schedule in seconds. Positive = late; negative = early. `0` when no real-time data is available.

**`tripStatus.vehicleId`** — ID of the vehicle. Empty string when unknown.

**`tripStatus.occupancyStatus`** — Vehicle occupancy as a GTFS-RT status name. Empty string when unknown.

**`tripStatus.occupancyCount`** — Number of passengers on board. `-1` when unknown.

**`tripStatus.occupancyCapacity`** — Passenger capacity of the vehicle. `-1` when unknown.

**`tripStatus.vehicleFeatures[]`** — List of feature strings describing this vehicle (e.g. accessibility features).

**`tripStatus.situationIds[]`** — IDs of service alerts currently active for this trip. Referenced in `data.references.situations`.

---

### `data.references`

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

**`data.references.agencies[]`** — Full agency objects for every agency referenced in this response. Sorted with the primary agency first.

**`data.references.routes[]`** — Full route objects for every route referenced in `routeId` fields. Sorted by the deployment's custom route ordering.

**`data.references.stops[]`** — Full stop objects for the queried stop, nearby stops, and any stops referenced in trip status sub-objects.

**`data.references.trips[]`** — Full trip objects for every trip referenced in `tripId` or `tripStatus.activeTripId` fields.

**`data.references.situations[]`** — Full service alert objects for every alert ID referenced in `situationIds` fields.

**`data.references.stopTimes[]`** — Reserved; empty for these endpoints.
