## Goal in Context

A rider's client application wants to show all transit vehicles currently active on a specific route — their real-time positions, schedule adherence, and upcoming stops — so the rider can decide which trip to board.

## Scope

OneBusAway REST API — `trips-for-route` endpoint.

## Level

User goal.

## Primary Actor

Rider (via a client application).

## Stakeholders and Interests

- **Rider**: wants an accurate, up-to-date picture of every active trip on the route so they can plan their journey.

## Preconditions

- The caller supplies a valid API key.
- The route ID identifies a route known to the system.

## Minimal Guarantees

- The response envelope is always returned with an appropriate HTTP status code and a machine-readable status code in the body.
- If the route is outside the service area, an empty list is returned (HTTP 200) rather than an error.

## Success Guarantees

- The response contains one entry per active trip on the route at the requested time.
- Each entry carries the trip's scheduled service date, and optionally its real-time status, its full stop schedule, and any active service alerts.
- Referenced entities (trips, routes, agencies, stops, situations) are gathered into the references block.

## Trigger

A GET request to `/api/where/trips-for-route/{id}.{format}`.

## Main Success Scenario

1. The caller issues a GET request with a route ID embedded in the URL path and optional query parameters.
2. The system resolves the requested time (defaulting to the current server time if none is supplied).
3. The system identifies all scheduled block instances for the route whose service window intersects the interval from 30 minutes before the requested time to 10 minutes after it. The backward extension captures trips that may be running late; the forward extension captures trips that may be running early. Both windows are operator-configurable.
4. For each matched block instance, the system computes the real-time state of the vehicle at the requested time.
5. For each active block, the system assembles a trip details entry containing:
   - The trip ID and the service date (midnight of the operating day, in Unix milliseconds).
   - A frequency block if the trip runs on a frequency-based schedule.
   - The full real-time status of the vehicle (if `includeStatus` is `true`).
   - The complete stop schedule for the trip (if `includeSchedule` is `true`).
   - IDs of any service alerts currently affecting the trip.
6. If `includeTrip` is `true`, the trip record is added to the references block, which transitively pulls in the route and agency.
7. Stops referenced by the status (closest stop, next stop) and by the schedule are added to the references block.
8. Active service alert records are added to the references block.
9. The system returns HTTP 200 with the list of trip detail entries and the populated references block.

## Extensions

**2a. `time` is supplied as a Unix millisecond integer or a formatted date string (`yyyy-MM-dd_HH-mm-ss`).**
The system uses that value in place of the current server time. Continue from step 3.

**4a. A trip is marked as canceled and the system is configured to suppress canceled trips.**
The status object for that trip is omitted from its entry even when `includeStatus` is `true`. The trip entry is still present in the list. See Suspected Defects.

**4b. A trip's data is incomplete (its trip record or stop times cannot be loaded).**
That trip is silently excluded from the result list.

**5a. The route is outside the service area.**
The system catches the out-of-area condition and returns HTTP 200 with an empty list and `outOfRange: true`.

**5b. The `id` path parameter is absent.**
The system returns HTTP 400 with a validation error body.

**5c. A `version` other than `2` is supplied.**
The system returns HTTP 500 with an "unknown version" error body.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["id"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Route ID, embedded in the URL path: /api/where/trips-for-route/{id}.{format}"
    },
    "key": {
      "type": "string",
      "description": "API key."
    },
    "time": {
      "type": "string",
      "description": "The point in time for which active trips are computed. Accepts a Unix millisecond timestamp (integer string) or a formatted date-time string (yyyy-MM-dd_HH-mm-ss). Defaults to the server's current time."
    },
    "includeTrip": {
      "type": "boolean",
      "default": true,
      "description": "Whether to load each trip's record into the references block."
    },
    "includeStatus": {
      "type": "boolean",
      "default": true,
      "description": "Whether to include the real-time status block in each trip entry."
    },
    "includeSchedule": {
      "type": "boolean",
      "default": true,
      "description": "Whether to include the stop-time schedule in each trip entry."
    },
    "maxCount": {
      "type": "integer",
      "description": "Accepted for compatibility but not applied — see Suspected Defects. All matching trips are always returned."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "Whether to populate the references block."
    },
    "version": {
      "type": "integer",
      "default": 2,
      "description": "API version. Only 2 is valid. Any other value produces an error response."
    }
  }
}
```

**`id`** — The route's compound identifier in `{agencyId}_{routeId}` form, URL-encoded in the path. Required.

**`key`** — API access key. Required by the API infrastructure.

**`time`** — Point in time used to determine which trips are active. Accepts either a plain integer (Unix milliseconds) or a date-time string in `yyyy-MM-dd_HH-mm-ss` format. When omitted, the server's current time is used.

**`includeTrip`** — When `true` (the default), each trip's record is loaded and placed in the `references.trips` block, which transitively adds the route and agency. When `false`, trip records are not loaded and do not appear in references.

**`includeStatus`** — When `true` (the default), each trip entry includes a `status` object containing the vehicle's real-time position and schedule deviation. When `false`, the `status` field is absent.

**`includeSchedule`** — When `true` (the default), each trip entry includes a `schedule` object with the full ordered list of stop times. When `false`, the `schedule` field is absent.

**`maxCount`** — Accepted and parsed but never applied. The result set is never capped regardless of this value. See Suspected Defects.

**`includeReferences`** — When `false`, the `references` block is returned empty. Default is `true`.

**`version`** — Selects the response format version. The only supported value is `2`. Omitting it is equivalent to passing `2`. Any other value returns an HTTP 500 error.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "version": { "type": "integer" },
    "code": { "type": "integer" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "text": { "type": "string" },
    "data": { "type": "object" }
  }
}
```

**`version`** — Always `2`.

**`code`** — HTTP-style status code: `200` on success, `400` for a missing or invalid `id`, `500` for an unknown version.

**`currentTime`** — The server's wall-clock time at the moment the response was generated, in Unix milliseconds.

**`text`** — Human-readable status phrase corresponding to `code`.

**`data`** — The payload object; see below.

---

### data

```json
{
  "type": "object",
  "properties": {
    "list": {
      "type": "array",
      "items": { "type": "object" }
    },
    "limitExceeded": { "type": "boolean" },
    "outOfRange": { "type": "boolean" },
    "references": { "type": "object" }
  }
}
```

**`data.list`** — Array of trip detail entries; see `data.list[]` below.

**`data.limitExceeded`** — Always `false`. The service layer does not apply a cap to results for this endpoint, so the flag is never set. See Suspected Defects (`maxCount`).

**`data.outOfRange`** — `true` when the route falls outside the system's service area (the list is empty in this case). Otherwise `false`.

**`data.references`** — Collected referenced entities. Contains sub-objects `agencies`, `routes`, `trips`, `stops`, and `situations`, each an array of the corresponding entity records. Populated according to the `includeTrip`, `includeStatus`, and `includeSchedule` flags. Empty when `includeReferences=false`.

---

### data.list[]

One entry per active trip on the route.

```json
{
  "type": "object",
  "properties": {
    "tripId": { "type": "string" },
    "serviceDate": { "type": "integer", "description": "Unix ms" },
    "frequency": { "type": "object" },
    "status": { "type": "object" },
    "schedule": { "type": "object" },
    "situationIds": {
      "type": "array",
      "items": { "type": "string" }
    }
  }
}
```

**`data.list[].tripId`** — The trip's compound identifier, matching an entry in `data.references.trips` when `includeTrip` is `true`.

**`data.list[].serviceDate`** — Unix millisecond timestamp of midnight at the start of the operating day for this trip instance.

**`data.list[].frequency`** — Present only for frequency-based trips. See `data.list[].frequency` below.

**`data.list[].status`** — Present when `includeStatus` is `true` and the trip has a computable real-time state. Absent when `includeStatus` is `false`, or when a canceled trip is hidden by operator configuration. See `data.list[].status` below.

**`data.list[].schedule`** — Present when `includeSchedule` is `true`. See `data.list[].schedule` below.

**`data.list[].situationIds`** — IDs of service alerts currently affecting this trip. Absent when there are none. Each ID matches an entry in `data.references.situations`.

---

### data.list[].frequency

```json
{
  "type": "object",
  "properties": {
    "startTime": { "type": "integer", "description": "Unix ms" },
    "endTime": { "type": "integer", "description": "Unix ms" },
    "headway": { "type": "integer" },
    "exactTimes": { "type": "integer" }
  }
}
```

**`data.list[].frequency.startTime`** — Start of the frequency window, in Unix milliseconds.

**`data.list[].frequency.endTime`** — End of the frequency window, in Unix milliseconds.

**`data.list[].frequency.headway`** — Scheduled headway in seconds.

**`data.list[].frequency.exactTimes`** — `1` if departures are locked to exact times within the headway; `0` otherwise.

---

### data.list[].status

Present when `includeStatus` is `true`.

```json
{
  "type": "object",
  "properties": {
    "activeTripId": { "type": "string" },
    "blockTripSequence": { "type": "integer" },
    "serviceDate": { "type": "integer", "description": "Unix ms" },
    "frequency": { "type": "object" },
    "scheduledDistanceAlongTrip": { "type": "number" },
    "totalDistanceAlongTrip": { "type": "number" },
    "position": { "type": "object" },
    "orientation": { "type": "number" },
    "closestStop": { "type": "string" },
    "closestStopTimeOffset": { "type": "integer" },
    "nextStop": { "type": "string" },
    "nextStopTimeOffset": { "type": "integer" },
    "phase": { "type": "string" },
    "status": { "type": "string" },
    "predicted": { "type": "boolean" },
    "lastUpdateTime": { "type": "integer", "description": "Unix ms" },
    "lastLocationUpdateTime": { "type": "integer", "description": "Unix ms" },
    "lastKnownDistanceAlongTrip": { "type": "number" },
    "lastKnownLocation": { "type": "object" },
    "lastKnownOrientation": { "type": "number" },
    "scheduleDeviation": { "type": "integer" },
    "distanceAlongTrip": { "type": "number" },
    "vehicleId": { "type": "string" },
    "situationIds": {
      "type": "array",
      "items": { "type": "string" }
    },
    "occupancyStatus": { "type": "string" },
    "occupancyCount": { "type": "integer" },
    "occupancyCapacity": { "type": "integer" },
    "vehicleFeatures": {
      "type": "array",
      "items": { "type": "string" }
    }
  }
}
```

**`data.list[].status.activeTripId`** — The trip the vehicle is currently serving. May differ from the entry's `tripId` when the vehicle is running a block that spans multiple trips. Present only when the vehicle has an active trip assignment.

**`data.list[].status.blockTripSequence`** — Zero-based index of the active trip within its block's ordered trip sequence. `-1` when not set.

**`data.list[].status.serviceDate`** — Unix millisecond timestamp of midnight at the start of the operating day for the active trip.

**`data.list[].status.frequency`** — Frequency window for the active trip if it runs on a frequency-based schedule; absent otherwise.

**`data.list[].status.scheduledDistanceAlongTrip`** — How far, in metres, the vehicle is scheduled to have travelled along the active trip at the requested time. Absent when the trip is not in progress.

**`data.list[].status.totalDistanceAlongTrip`** — Total length of the active trip in metres.

**`data.list[].status.position`** — Current position of the vehicle as `{ "lat": number, "lon": number }`. Present when a position can be computed (from real-time data or schedule extrapolation). Absent when the vehicle's location is unknown.

**`data.list[].status.orientation`** — Compass heading in degrees, where 0° is east, 90° is north, 180° is west, 270° is south. Absent when unknown.

**`data.list[].status.closestStop`** — ID of the stop nearest to the vehicle's current location. Absent when no stop is identified.

**`data.list[].status.closestStopTimeOffset`** — Seconds between the vehicle's current position and the closest stop. Positive means the stop is ahead; negative means the stop has been passed.

**`data.list[].status.nextStop`** — ID of the next stop the vehicle will serve. Absent when the vehicle has passed the last stop in the trip.

**`data.list[].status.nextStopTimeOffset`** — Seconds until the vehicle reaches the next stop.

**`data.list[].status.phase`** — A string describing the current operational phase of the trip (e.g. `in_progress`, `layover_before`, `deadhead_before`).

**`data.list[].status.status`** — A modifier string for the trip. `"default"` for normal operation; `"canceled"` when the trip has been canceled (only present when the operator has not configured canceled trips to be hidden).

**`data.list[].status.predicted`** — `true` if real-time arrival information is available for this trip.

**`data.list[].status.lastUpdateTime`** — Unix millisecond timestamp of the most recent real-time update received from the vehicle. Absent (not included in the response) when no update has been received (i.e. value is zero).

**`data.list[].status.lastLocationUpdateTime`** — Unix millisecond timestamp of the most recent real-time location update. Absent when no location update has been received.

**`data.list[].status.lastKnownDistanceAlongTrip`** — The last raw distance-along-trip reading received from the vehicle, in metres. Absent when not available.

**`data.list[].status.lastKnownLocation`** — The vehicle's last directly reported position as `{ "lat": number, "lon": number }`. Differs from `position`, which may be extrapolated forward. Absent when no location report has been received.

**`data.list[].status.lastKnownOrientation`** — The vehicle's last directly reported heading in degrees. Absent when not available.

**`data.list[].status.scheduleDeviation`** — Seconds by which the trip is deviating from schedule. Positive means late; negative means early. Absent when no real-time data is available.

**`data.list[].status.distanceAlongTrip`** — How far, in metres, the vehicle has actually travelled along the active trip. May be extrapolated from the last known reading. Absent when not available.

**`data.list[].status.vehicleId`** — Compound identifier of the vehicle serving the trip. Absent when real-time data is unavailable or no vehicle is assigned.

**`data.list[].status.situationIds`** — IDs of service alerts currently affecting the vehicle's active trip. Absent when there are none.

**`data.list[].status.occupancyStatus`** — GTFS-Realtime `OccupancyStatus` enum name (e.g. `MANY_SEATS_AVAILABLE`, `FEW_SEATS_AVAILABLE`, `FULL`). Absent when occupancy data is unavailable.

**`data.list[].status.occupancyCount`** — Raw passenger count from an automatic passenger counter. `-1` when no data is available.

**`data.list[].status.occupancyCapacity`** — Declared vehicle capacity. `-1` when not configured.

**`data.list[].status.vehicleFeatures`** — List of feature strings describing equipment on the vehicle (e.g. accessibility features). Absent when none are reported.

---

### data.list[].schedule

Present when `includeSchedule` is `true`.

```json
{
  "type": "object",
  "properties": {
    "timeZone": { "type": "string" },
    "stopTimes": {
      "type": "array",
      "items": { "type": "object" }
    },
    "previousTripId": { "type": "string" },
    "nextTripId": { "type": "string" },
    "frequency": { "type": "object" }
  }
}
```

**`data.list[].schedule.timeZone`** — IANA time zone identifier for the agency operating this trip (e.g. `America/Los_Angeles`). Arrival and departure times in `stopTimes` are expressed in seconds from midnight in this zone.

**`data.list[].schedule.stopTimes`** — Ordered list of stop visits for the trip. See `data.list[].schedule.stopTimes[]` below.

**`data.list[].schedule.previousTripId`** — If this trip is part of a block and is preceded by another trip, the ID of that preceding trip. Absent when there is no predecessor.

**`data.list[].schedule.nextTripId`** — If this trip is part of a block and is followed by another trip, the ID of that following trip. Absent when there is no successor.

**`data.list[].schedule.frequency`** — Frequency information for the trip if it runs on a frequency-based schedule; absent otherwise.

---

### data.list[].schedule.stopTimes[]

```json
{
  "type": "object",
  "properties": {
    "arrivalTime": { "type": "integer" },
    "departureTime": { "type": "integer" },
    "stopId": { "type": "string" },
    "stopHeadsign": { "type": "string" },
    "distanceAlongTrip": { "type": "number" },
    "historicalOccupancy": { "type": "string" }
  }
}
```

**`data.list[].schedule.stopTimes[].arrivalTime`** — Scheduled arrival time, in seconds elapsed since midnight of the service date (in the trip's time zone). These are static scheduled times; real-time adjustments are not reflected here.

**`data.list[].schedule.stopTimes[].departureTime`** — Scheduled departure time, in seconds elapsed since midnight of the service date.

**`data.list[].schedule.stopTimes[].stopId`** — Compound identifier of the stop. Matches an entry in `data.references.stops`.

**`data.list[].schedule.stopTimes[].stopHeadsign`** — An override headsign for this specific stop visit, if the GTFS data specifies one. Absent when the trip's default headsign applies.

**`data.list[].schedule.stopTimes[].distanceAlongTrip`** — Distance in metres from the start of the trip to this stop.

**`data.list[].schedule.stopTimes[].historicalOccupancy`** — Historical occupancy level at this stop, expressed as a GTFS-Realtime `OccupancyStatus` name, if available. Absent when no historical data exists.

---

## Suspected Defects

### Defects that affect the use case

**`maxCount` is accepted but never applied — `TripsForRouteAction`, line 109.**
The action parses `maxCount` and writes it to the query bean, but the service layer (`TripStatusBeanServiceImpl.getTripsForRoute`) never reads the query's count field when calling the block calendar. All matching trips are always returned, and `data.limitExceeded` is always `false`. A Go implementation should either honour the parameter as documented or remove it from the parameter list entirely.

**Canceled trips remain in the list when hidden by operator configuration — `TripStatusBeanServiceImpl.getBlockLocationAsStatusBean`, around line 254.**
When the operator configures the system to suppress canceled trips, `getBlockLocationAsStatusBean` returns `null` for those trips. The calling code treats a null status as absent rather than as a signal to exclude the trip from results. Consequently, canceled trips still appear in the list with no `status` block even when `includeStatus` is `true`. The likely intent is that hidden canceled trips should be excluded from the list entirely. A Go implementation should decide whether to replicate this behaviour (for client compatibility) or exclude them.

**Documented defaults for `includeStatus`, `includeSchedule`, and `includeTrip` are wrong.**
The existing Markdown documentation (`src/site/markdown/api/where/methods/trips-for-route.md`) states that `includeStatus` and `includeSchedule` default to `false`, and does not mention `includeTrip`. The code (`TripsForRouteAction`, lines 54–58) initialises all three to `true`. The code is authoritative.

### Implementation defects only

**Null check targets the wrong variable in `TripStatusBeanServiceImpl.getBlockLocationsAsTripDetails`, around line 424.**
The loop body reads `if (tripDetails != null)` — which tests the accumulator list (never null) — where `if (details != null)` was intended. As a result, a null `TripDetailsBean` can be appended to the list. The downstream method `BeanFactoryV2.getTripDetailsResponse` explicitly skips null entries, so there is no user-visible effect. A clean reimplementation would naturally avoid this mistake.

---

## Open Questions

**Trip selection window defaults.** The backward window (30 minutes) and forward window (10 minutes) are annotated as `@ConfigurationParameter`, meaning operators can override them at deployment time. The spec documents the compiled-in defaults. Actual production window values depend on the deployment configuration and may differ.
