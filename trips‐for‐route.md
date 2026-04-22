## Goal in Context

A rider-facing client wants to display which vehicles are currently running (or are about to run) on a specific route, along with their real-time positions, schedule adherence, and stop-time schedules, so passengers can decide when and where to board.

## Scope

OneBusAway REST API — `GET /api/where/trips-for-route/{id}.json`

## Level

User-goal

## Primary Actor

Rider (via a client application)

## Stakeholders and Interests

- **Rider**: wants an accurate, up-to-date list of vehicles currently operating or starting soon on a chosen route, including predicted positions and how late or early each vehicle is running.

## Preconditions

- The caller supplies a valid API key.
- The server has a loaded GTFS data bundle.

## Minimal Guarantees

- The response is always HTTP 200 with a well-formed JSON envelope.
- When a request targets a route that is outside the server's service area, an empty list is returned with `data.outOfRange` set to `true`.

## Success Guarantees

- Every block that has a scheduled trip overlapping the active window around the requested time is represented by one entry in `data.list`.
- Each entry reflects the trip the block is **currently executing** at the requested time, including real-time position and deviation when available.
- Referenced trips, stops, routes, agencies, and service alerts are fully populated in `data.references` (unless the caller suppresses them with `includeReferences=false`).

## Trigger

`GET /api/where/trips-for-route/{id}.json?key=…`

## Main Success Scenario

1. The caller issues a GET request supplying a route ID (path parameter) and optional query parameters for time, inclusion flags, and API key.
2. The server resolves the reference time: the supplied `time` value if provided; otherwise the current server time.
3. The server computes an active window extending 30 minutes before and 10 minutes after the reference time. (These defaults are configurable at deployment time.)
4. The server identifies every block that has at least one trip on the queried route scheduled to overlap that active window.
5. For each such block, the server determines which trip within that block is currently active at the reference time. This trip becomes the entry's `tripId` and the status `activeTripId`. **Due to interlining, the active trip may belong to a different route than the one queried** — this is expected, intended behaviour (see note below).
6. For each block, the server assembles a trip detail entry:
   - `tripId` and `serviceDate` are always included.
   - The full trip record (headsign, route, direction, etc.) is added to `data.references.trips` when `includeTrip` is `true`.
   - The real-time `status` block is included when `includeStatus` is `true`.
   - The `schedule` block (ordered stop times for the active trip, plus adjacent block-trip IDs) is included when `includeSchedule` is `true`.
   - Service alert IDs applicable to the vehicle journey are listed in `situationIds`; the full alert records appear in `data.references.situations`.
7. All referenced stops, routes, and agencies are populated in `data.references`.
8. The server returns HTTP 200 with the assembled list. `data.limitExceeded` is always `false`. The order of entries in `data.list` is non-deterministic.

**Interlining note:** The endpoint selects blocks whose scheduled trips overlap the active window for the queried route. A selected block may, at the moment of the request, be executing a trip that belongs to a different route — because the vehicle is running a sequence of trips across multiple routes without returning to the depot. In that case the entry's `tripId` and status `activeTripId` will reference a trip on the other route, and the `schedule.stopTimes` will cover that other trip's stops. This is not a defect; it accurately reflects what the vehicle is doing.

## Extensions

**2a. Route not found / no active blocks:**
The server finds no blocks matching the queried route and time window. It returns HTTP 200 with `data.list` empty and `data.limitExceeded: false`. `data.outOfRange` is `false`.

**2b. `time` parameter supplied:**
The server uses the provided timestamp as the reference time instead of the current time. All window calculations proceed from that time. This is useful for testing and replay.

**5a. Block has no active trip at the reference time:**
If the block location cannot resolve an active trip instance (e.g. the block is between trips), the entry is silently omitted from the result.

**6a. `includeStatus=false`:**
The `status` key is absent from the entry (not null — absent entirely).

**6b. `includeSchedule=false`:**
The `schedule` key is absent from the entry.

**6c. `includeTrip=false`:**
The trip record is not added to `data.references.trips`. The `tripId` field is still present in the entry.

**6d. No real-time data available for a vehicle:**
`status.predicted` is `false`. Position and orientation are computed from the static schedule. `status.scheduleDeviation` is `0`. `status.vehicleId` is an empty string.

**6e. Trip is frequency-based:**
A `frequency` object is included at both the entry level and within `status`, describing the headway interval.

**6f. Active trip has a preceding trip in the same block:**
`schedule.previousTripId` is populated with that preceding trip's ID. The preceding trip is added to `data.references.trips`.

**6g. Active trip has a following trip in the same block:**
`schedule.nextTripId` is populated. The following trip is added to `data.references.trips`.

**6h. `includeReferences=false`:**
`data.references` is still present in the response but all arrays within it are empty.

**Error — missing route ID:**
The Struts framework returns HTTP 404 before the action is reached.

**Error — wrong API version:**
If the `version` parameter is present and not `2`, the server returns HTTP 500 with code `500` and `text: "unknown version: N"`.

**Error — validation failure (e.g. bad field type):**
The server returns HTTP 400 with code `400` and a validation error body.

## Suspected Defects

### Defects that affect the use case

**`maxCount` parameter is accepted but has no effect**
`TripsForRouteAction.java` (lines 78–80) accepts a `maxCount` parameter and passes it into the query bean via `query.setMaxCount(...)`. However, `TripStatusBeanServiceImpl.getTripsForRoute()` (line 231) reads only the inclusion and time fields from the query; it never reads `getMaxCount()`. No result capping is applied regardless of the value the caller supplies. The `data.limitExceeded` flag is hardcoded to `false`. A Go reimplementation should either remove the parameter or implement actual capping.

**`includeStatus` and `includeSchedule` documented defaults are wrong**
The existing method documentation (`src/site/markdown/api/where/methods/trips-for-route.md`) states that `includeStatus` and `includeSchedule` default to `false`. In the Java code (`TripsForRouteAction.java` lines 54–58), all three inclusion flags — `includeTrip`, `includeStatus`, and `includeSchedule` — are initialised to `true`. The live server confirms all three are included by default. A Go implementation should use `true` as the default for all three.

### Implementation defects only

**Wrong null guard in `TripStatusBeanServiceImpl.getBlockLocationsAsTripDetails()` (~line 424)**
The method checks `if (tripDetails != null)` before adding to the result list, where `tripDetails` is the accumulator list itself — it is never null. The intended check was `if (details != null)`. As a result, null `TripDetailsBean` values (produced when a block has no active trip instance) are added to the list. `BeanFactoryV2.getTripDetailsResponse()` filters them out with its own `if (trip != null)` guard, so the observable output is unaffected. A clean reimplementation would simply omit the null guard on the list or place it correctly.

**`lastKnownLocation` set twice in `TripStatusBeanServiceImpl.getBlockLocationAsStatusBean()` (lines 270 and 276)**
`bean.setLastKnownLocation(blockLocation.getLastKnownLocation())` is called at line 270 and again at line 276–278 (inside a conditional block that also sets orientation). The second call produces the same value as the first and overwrites it without effect. No observable difference at the API boundary.

## Open Questions

None. All observable behaviours were resolved by static analysis of the full call chain and confirmed with live server requests against KCM data.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Agency-qualified route ID, encoded in the URL path: /api/where/trips-for-route/{id}.json"
    },
    "key": {
      "type": "string",
      "description": "API key"
    },
    "time": {
      "type": "integer",
      "description": "Unix ms. Reference time for the query. Defaults to server current time."
    },
    "includeTrip": {
      "type": "boolean",
      "default": true,
      "description": "Whether to populate trip records in data.references.trips."
    },
    "includeStatus": {
      "type": "boolean",
      "default": true,
      "description": "Whether to include the real-time status block in each entry."
    },
    "includeSchedule": {
      "type": "boolean",
      "default": true,
      "description": "Whether to include the stop-time schedule in each entry."
    },
    "maxCount": {
      "type": "integer",
      "description": "Accepted but has no effect. No result capping is applied. See Suspected Defects."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "If false, data.references is returned with all arrays empty."
    },
    "version": {
      "type": "integer",
      "default": 2,
      "description": "API version. Only 2 is supported; other values produce HTTP 500."
    }
  }
}
```

**`id`** — Agency-qualified route ID in the URL path, e.g. `1_100018`. The agency prefix and the underscore separator are required.

**`key`** — API key. Required for all requests.

**`time`** — Unix timestamp in milliseconds. Shifts the reference point for the active-window calculation. Primarily useful for testing historical snapshots.

**`includeTrip`** — When `true` (the default), full trip records (headsign, route ID, direction, shape ID, etc.) are placed in `data.references.trips` for every trip referenced by the response — including the active trips and any preceding/following block trips. When `false`, the trip bean is not fetched and `data.references.trips` will be empty.

**`includeStatus`** — When `true` (the default), each entry contains a `status` block with the vehicle's current position, deviation, and occupancy. When `false`, the `status` key is absent from the entry.

**`includeSchedule`** — When `true` (the default), each entry contains a `schedule` block with the full ordered stop sequence for the active trip, plus IDs for adjacent block trips. When `false`, the `schedule` key is absent.

**`maxCount`** — Intended to cap the number of results. Has no effect due to a defect; all matching blocks are always returned. See Suspected Defects.

**`includeReferences`** — Controls whether `data.references` is populated. When `false`, the `references` object is still present but all arrays inside it are empty.

**`version`** — Only `2` is accepted. Any other value produces HTTP 500 with `text: "unknown version: N"`.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "code":        { "type": "integer" },
    "text":        { "type": "string" },
    "version":     { "type": "integer" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "data":        { "type": "object" }
  }
}
```

**`code`** — HTTP-equivalent status code. Always `200` for a successful response.

**`text`** — Human-readable status. `"OK"` on success.

**`version`** — Always `2`.

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`data`** — The response payload object.

---

### data

```json
{
  "type": "object",
  "properties": {
    "list":           { "type": "array", "items": { "type": "object" } },
    "limitExceeded":  { "type": "boolean" },
    "outOfRange":     { "type": "boolean" },
    "references":     { "type": "object" }
  }
}
```

**`data.list`** — Array of trip detail entries, one per active block. Order is non-deterministic.

**`data.limitExceeded`** — Always `false`. No result cap is applied (see Suspected Defects).

**`data.outOfRange`** — `true` only if the server throws an out-of-service-area error, which does not occur for unknown or inactive routes in practice. A route with no active blocks returns an empty list with `outOfRange: false`.

**`data.references`** — Lookup tables for all entities referenced by ID within `data.list`. Populated when `includeReferences=true` (the default). When `includeReferences=false`, the object is present but all arrays are empty.

---

### data.list[]

```json
{
  "type": "object",
  "properties": {
    "tripId":       { "type": "string" },
    "serviceDate":  { "type": "integer", "description": "Unix ms" },
    "frequency":    { "type": "object" },
    "situationIds": { "type": "array", "items": { "type": "string" } },
    "status":       { "type": "object" },
    "schedule":     { "type": "object" }
  }
}
```

**`data.list[].tripId`** — ID of the trip the block is executing at the reference time. This is the *active* trip — which may belong to a route other than the one queried if the block is interlined.

**`data.list[].serviceDate`** — Unix millisecond timestamp of midnight on the service date for this block instance. Times in the schedule are expressed in seconds elapsed since this moment.

**`data.list[].frequency`** — Frequency-based service descriptor. Present only for trips that run on a headway schedule; `null` for fixed-schedule trips.

**`data.list[].situationIds`** — Array of service alert IDs applicable to this vehicle journey. Full alert records are in `data.references.situations`. Empty array when no alerts apply.

**`data.list[].status`** — Real-time status of the vehicle. Absent (key not present) when `includeStatus=false`. See `data.list[].status` schema below.

**`data.list[].schedule`** — Scheduled stop-time sequence for the active trip. Absent (key not present) when `includeSchedule=false`. See `data.list[].schedule` schema below.

---

### data.list[].status

```json
{
  "type": "object",
  "properties": {
    "activeTripId":                { "type": "string" },
    "blockTripSequence":           { "type": "integer" },
    "serviceDate":                 { "type": "integer", "description": "Unix ms" },
    "scheduledDistanceAlongTrip":  { "type": "number" },
    "totalDistanceAlongTrip":      { "type": "number" },
    "position": {
      "type": "object",
      "properties": {
        "lat": { "type": "number" },
        "lon": { "type": "number" }
      }
    },
    "orientation":                 { "type": "number" },
    "closestStop":                 { "type": "string" },
    "closestStopTimeOffset":       { "type": "integer" },
    "nextStop":                    { "type": "string" },
    "nextStopTimeOffset":          { "type": "integer" },
    "phase":                       { "type": "string" },
    "status":                      { "type": "string" },
    "predicted":                   { "type": "boolean" },
    "lastUpdateTime":              { "type": "integer", "description": "Unix ms" },
    "lastLocationUpdateTime":      { "type": "integer", "description": "Unix ms" },
    "lastKnownDistanceAlongTrip":  { "type": "number" },
    "lastKnownLocation": {
      "type": "object",
      "properties": {
        "lat": { "type": "number" },
        "lon": { "type": "number" }
      }
    },
    "lastKnownOrientation":        { "type": "number" },
    "scheduleDeviation":           { "type": "integer" },
    "distanceAlongTrip":           { "type": "number" },
    "vehicleId":                   { "type": "string" },
    "occupancyStatus":             { "type": "string" },
    "occupancyCount":              { "type": "integer" },
    "occupancyCapacity":           { "type": "integer" },
    "vehicleFeatures":             { "type": "array", "items": { "type": "string" } },
    "situationIds":                { "type": "array", "items": { "type": "string" } },
    "frequency":                   { "type": "object" }
  }
}
```

**`data.list[].status.activeTripId`** — Trip ID of the trip the vehicle is actively serving. Identical to the entry-level `tripId`.

**`data.list[].status.blockTripSequence`** — Zero-based index of the active trip within its block's ordered trip sequence. Useful for comparing against block data.

**`data.list[].status.serviceDate`** — Unix milliseconds of midnight on the service date.

**`data.list[].status.scheduledDistanceAlongTrip`** — Meters the vehicle is scheduled to have progressed along the active trip at the reference time. Computed from the static schedule when real-time data is unavailable.

**`data.list[].status.totalDistanceAlongTrip`** — Total length of the active trip in meters.

**`data.list[].status.position`** — Current latitude and longitude. When `predicted=false`, this is the schedule-derived position; when `predicted=true`, it reflects real-time GPS data (potentially extrapolated forward from the last known fix).

**`data.list[].status.orientation`** — Compass bearing in degrees, where 0° is east, 90° is north, 180° is west, and 270° is south. May be absent if not available.

**`data.list[].status.closestStop`** — Stop ID of the stop closest to the vehicle's current position along the trip. Added to `data.references.stops`.

**`data.list[].status.closestStopTimeOffset`** — Seconds between the scheduled time at the closest stop and the vehicle's current position. Positive means the stop is ahead; negative means it has been passed.

**`data.list[].status.nextStop`** — Stop ID of the next upcoming stop. Absent if the vehicle has passed the last stop. Added to `data.references.stops`.

**`data.list[].status.nextStopTimeOffset`** — Seconds to the scheduled time at the next stop.

**`data.list[].status.phase`** — A label describing the vehicle's current journey phase (e.g. in-progress, layover). Empty string when no real-time data is available.

**`data.list[].status.status`** — A status modifier for the trip. `"default"` for normally operating trips; `"canceled"` for cancelled trips (only present when cancelled-trip reporting is enabled).

**`data.list[].status.predicted`** — `true` if real-time GPS data was used to compute the vehicle position; `false` if the position is derived from the static schedule.

**`data.list[].status.lastUpdateTime`** — Unix milliseconds of the most recent real-time update received from the vehicle. `0` if no real-time data has ever been received.

**`data.list[].status.lastLocationUpdateTime`** — Unix milliseconds of the most recent real-time update that contained a GPS location. `0` if no location update has been received.

**`data.list[].status.lastKnownDistanceAlongTrip`** — Distance in meters along the active trip as reported in the most recent real-time fix. Absent if no real-time location has been received.

**`data.list[].status.lastKnownLocation`** — Latitude and longitude from the most recent real-time GPS fix. This differs from `position`, which may be extrapolated forward; `lastKnownLocation` is the raw last report. `null` if no location update has been received.

**`data.list[].status.lastKnownOrientation`** — Bearing from the most recent real-time fix. Absent if not available.

**`data.list[].status.scheduleDeviation`** — How many seconds late (positive) or early (negative) the vehicle is running at the reference time. `0` when `predicted=false`.

**`data.list[].status.distanceAlongTrip`** — Meters the vehicle has actually progressed along the active trip. May be extrapolated from the last known position when real-time data is stale.

**`data.list[].status.vehicleId`** — Opaque agency-assigned vehicle identifier. Empty string when no real-time data is available.

**`data.list[].status.occupancyStatus`** — GTFS-RT `OccupancyStatus` enum name (e.g. `"MANY_SEATS_AVAILABLE"`, `"FULL"`). Empty string when occupancy data is unavailable.

**`data.list[].status.occupancyCount`** — Raw passenger count reported by the vehicle. `-1` when unavailable.

**`data.list[].status.occupancyCapacity`** — Total passenger capacity of the vehicle. `-1` when unavailable.

**`data.list[].status.vehicleFeatures`** — Array of feature strings reported for the vehicle (e.g. accessibility features). Empty array when none are reported.

**`data.list[].status.situationIds`** — Service alert IDs specifically applicable to this vehicle's journey at the reference time. Full alert records are in `data.references.situations`.

**`data.list[].status.frequency`** — Frequency descriptor for the active trip, if it runs on a headway schedule. `null` for fixed-schedule trips.

---

### data.list[].schedule

```json
{
  "type": "object",
  "properties": {
    "timeZone":      { "type": "string" },
    "stopTimes":     { "type": "array", "items": { "type": "object" } },
    "previousTripId": { "type": "string" },
    "nextTripId":     { "type": "string" },
    "frequency":      { "type": "object" }
  }
}
```

**`data.list[].schedule.timeZone`** — IANA timezone identifier for the agency that operates this trip (e.g. `"America/Los_Angeles"`). All times in `stopTimes` are seconds elapsed since midnight in this timezone on the service date.

**`data.list[].schedule.stopTimes`** — Ordered sequence of stops for the active trip.

**`data.list[].schedule.previousTripId`** — Trip ID of the trip that immediately precedes the active trip in the block sequence. Present regardless of whether the preceding trip is on the same or a different route. Absent if the active trip is the first in the block. The trip record is added to `data.references.trips`.

**`data.list[].schedule.nextTripId`** — Trip ID of the trip that immediately follows the active trip in the block sequence. Absent if the active trip is the last in the block. The trip record is added to `data.references.trips`.

**`data.list[].schedule.frequency`** — Frequency descriptor for the active trip if it runs on a headway schedule.

---

### data.list[].schedule.stopTimes[]

```json
{
  "type": "object",
  "properties": {
    "stopId":             { "type": "string" },
    "arrivalTime":        { "type": "integer" },
    "departureTime":      { "type": "integer" },
    "distanceAlongTrip":  { "type": "number" },
    "stopHeadsign":       { "type": "string" },
    "historicalOccupancy": { "type": "string" }
  }
}
```

**`data.list[].schedule.stopTimes[].stopId`** — Stop ID. The corresponding stop record is added to `data.references.stops`.

**`data.list[].schedule.stopTimes[].arrivalTime`** — Scheduled arrival time at this stop, in seconds elapsed since midnight on the service date. Values ≥ 86,400 indicate a time on the next calendar day.

**`data.list[].schedule.stopTimes[].departureTime`** — Scheduled departure time at this stop, in seconds elapsed since midnight on the service date.

**`data.list[].schedule.stopTimes[].distanceAlongTrip`** — Cumulative distance in meters from the trip's first stop to this stop, derived from the GTFS shape.

**`data.list[].schedule.stopTimes[].stopHeadsign`** — The headsign text shown on the vehicle at this specific stop, if the route-pattern narrative defines one. Empty string if not defined.

**`data.list[].schedule.stopTimes[].historicalOccupancy`** — Historical average load at this stop for this route, expressed as a GTFS-RT `OccupancyStatus` enum name. Empty string if no historical data is available.

---

### data.references

```json
{
  "type": "object",
  "properties": {
    "agencies":   { "type": "array", "items": { "type": "object" } },
    "routes":     { "type": "array", "items": { "type": "object" } },
    "trips":      { "type": "array", "items": { "type": "object" } },
    "stops":      { "type": "array", "items": { "type": "object" } },
    "situations": { "type": "array", "items": { "type": "object" } },
    "stopTimes":  { "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.references.agencies`** — Full agency records for every agency that operates a trip referenced in the response.

**`data.references.routes`** — Full route records for every trip referenced in the response.

**`data.references.trips`** — Full trip records for: each entry's active trip (when `includeTrip=true`), each `schedule.previousTripId`, and each `schedule.nextTripId`. When interlining is present, this array will contain trips belonging to routes other than the queried route.

**`data.references.stops`** — Full stop records for every stop in every `schedule.stopTimes` array (when `includeSchedule=true`), plus `status.closestStop` and `status.nextStop` (when `includeStatus=true`).

**`data.references.situations`** — Full service alert records for every situation ID referenced in entry-level `situationIds` or status-level `situationIds`.

**`data.references.stopTimes`** — Always an empty array for this endpoint. Present for schema consistency.
