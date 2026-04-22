**Endpoint:** `GET /api/where/trips-for-location.json`

---

## Goal in Context

A rider's client application wants to display all transit vehicles currently operating near a map viewport so the rider can see what is moving in real time (or, when real-time data is unavailable, where vehicles are scheduled to be).

---

## Scope

OBA REST API — `trips-for-location` endpoint.

---

## Level

User goal.

---

## Primary Actor

Rider (via a client application).

---

## Stakeholders and Interests

- **Rider** — wants an accurate, timely picture of active vehicles near their location so they can make boarding decisions.

---

## Preconditions

- The caller supplies a valid API key.
- The caller supplies a geographic centre point (`lat`, `lon`) and at least one of: a radius, or both a latitude span and a longitude span.
- The requested area falls within the service area covered by the loaded transit data.

---

## Minimal Guarantees

- The response is always well-formed JSON with an HTTP 200 status code (even for out-of-area or empty results).
- A request using an unsupported API version returns HTTP 500 with a descriptive error message.

---

## Success Guarantees

- Every entry in the list corresponds to a transit vehicle that was, at the query time, physically located within the requested bounds.
- Each entry's `tripId` is the ID of the trip the vehicle was actively serving at that moment.
- If `includeTrip` is true (the default), the referenced trip object is present in the `references` block.
- `outOfRange` is false.

---

## Trigger

An HTTP GET request to `/api/where/trips-for-location.json` with the required parameters.

---

## Main Success Scenario

1. The system receives the request and validates that the API version is 2 (the only supported version).
2. The system constructs a geographic bounding box from the request parameters:
   - If `radius` is provided: a circular area of that radius (in metres) centred on `lat`/`lon`, converted to an axis-aligned bounding box. Capped at 20,000 metres.
   - If `latSpan` and `lonSpan` are provided: a box of that span centred on `lat`/`lon`.
   - Both methods respect a maximum area equivalent to a 20,000-metre radius.
3. The system determines the query time: if `time` is provided, that value is used; otherwise the server's current wall-clock time.
4. The system identifies candidate scheduled blocks: all blocks whose scheduled geographic path passes through the bounding box during the window from 30 minutes before to 10 minutes after the query time. This wider window ensures that late-running or early-running vehicles are not missed.
5. For each candidate block the system computes its position at the query time:
   - If real-time GPS data is available for the block's vehicle, the position is derived from that data (`predicted` = true in the status object).
   - If no real-time data is available, the position is extrapolated from the static schedule (`predicted` = false).
6. The system retains only those blocks whose computed position falls inside the bounding box.
7. For each retained block, the system constructs a trip-details entry using the active trip instance — the trip the vehicle is currently serving at the query time.
8. Optional sub-objects are included according to the request flags:
   - `includeTrip=true` (default): the trip record is added to the `references` block; its route and agency are also added.
   - `includeStatus=true`: a `status` sub-object with position, deviation, occupancy, and related fields is embedded in each entry.
   - `includeSchedule=true`: a `schedule` sub-object with the full stop-time sequence, adjacent block trips, and timezone is embedded in each entry.
   - When `includeStatus=true` or `includeSchedule=true`, the stops referenced by those sub-objects are added to the `references` block.
9. The system assembles the response envelope and returns it with HTTP 200. `limitExceeded` is always false. `outOfRange` is false.

---

## Extensions

**2a. No radius or span supplied:**
The bounds default to a zero-size area at the centre point and no vehicles are returned (empty list, `outOfRange` false).

**4a. No candidate blocks found:**
No blocks are scheduled through the area during the search window. The list is empty; `outOfRange` is false.

**6a. All candidate blocks are outside the bounds at query time:**
After position computation, no block's location falls within the bounding box. The list is empty.

**6b. Query coordinates are outside the service area:**
The geospatial index raises an out-of-area exception. The system returns HTTP 200 with an empty list and `outOfRange` = true.

**7a. A block has no active trip at the query time:**
The block's active trip instance is null. The entry is silently dropped; no error is surfaced to the caller.

**8a. `includeReferences=false`:**
The `references` block is omitted from the response.

**B. Unsupported API version:**
If `version` is supplied and is not 2, the system returns HTTP 500 with `"unknown version: <n>"`. No data is returned.

---

## Suspected Defects

### Defects that affect the use case

**Max-area clamping is broken in `SearchBoundsFactory.createBounds()`.**

When the requested area exceeds the 20,000-metre maximum, the clamping logic calculates updated local span variables but then creates the replacement bounds from the original, unclamped instance variables (`_latSpan / 2` and `_lonSpan / 2`). Two observable failures result:

- *Radius-based queries that exceed the maximum*: `_latSpan` and `_lonSpan` are both zero (they are only set when the caller uses the span parameters), so the replacement bounds collapse to a zero-size area at the centre point. The caller receives an empty list instead of results capped to the maximum area.
- *Span-based queries that exceed the maximum in at least one dimension*: the clamped values are computed but discarded. The original, over-sized span is used, so the maximum is not enforced at all.

In a correct implementation, exceeding the maximum should cap the search to the largest permitted area, not collapse it to a point or ignore the cap entirely. File: `SearchBoundsFactory.java`, `createBounds()`, line 90.

**`maxCount` is accepted but never applied.**

`TripsForLocationAction` accepts a `maxCount` parameter, passes it to the query bean, and the query bean passes it to the service layer. However, `TripStatusBeanServiceImpl.getTripsForBounds()` passes only `bounds` and `time` to `BlockStatusServiceImpl.getBlocksForBounds()` — `maxCount` is not forwarded and is never used anywhere in the call chain. `limitExceeded` is hardcoded to false. Callers who set `maxCount` expecting to cap the result size will receive all matching vehicles regardless.

File: `TripStatusBeanServiceImpl.java`, `getTripsForBounds()`, lines 222–228.

**Existing documentation says `includeTrip` defaults to false; the code sets it to true.**

`TripsForLocationAction` initialises `_includeTrip = true`. The existing Markdown documentation at `src/site/markdown/api/where/methods/trips-for-location.md` states the default is false. The code is authoritative; the documentation is incorrect.

### Implementation defects only

**Wrong null check in `TripStatusBeanServiceImpl.getBlockLocationsAsTripDetails()`.**

Line 424: `if (tripDetails != null)` tests the list variable (which is never null — it is initialised with `new ArrayList<>()` on line 420) instead of `if (details != null)`. As a result, null `TripDetailsBean` objects can be added to the list. This does not cause a visible API defect because `BeanFactoryV2.getTripDetailsResponse()` (line 328) explicitly skips null entries before serialisation. The fix is to change the condition to check `details` rather than `tripDetails`.

File: `TripStatusBeanServiceImpl.java`, `getBlockLocationsAsTripDetails()`, line 424.

---

## Open Questions

None. All behaviours were determinable from static analysis and live server testing.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["key", "lat", "lon"],
  "properties": {
    "key":              { "type": "string" },
    "version":          { "type": "integer", "default": 2 },
    "lat":              { "type": "number" },
    "lon":              { "type": "number" },
    "radius":           { "type": "number" },
    "latSpan":          { "type": "number" },
    "lonSpan":          { "type": "number" },
    "time":             { "type": "string" },
    "maxCount":         { "type": "integer" },
    "includeTrip":      { "type": "boolean", "default": true },
    "includeStatus":    { "type": "boolean", "default": false },
    "includeSchedule":  { "type": "boolean", "default": false },
    "includeReferences":{ "type": "boolean", "default": true }
  }
}
```

**`key`** — API key; required for all requests.

**`version`** — API version. Only version 2 is supported; any other value produces an HTTP 500 error. Omitting the parameter defaults to version 2.

**`lat`** — Latitude of the search centre, in decimal degrees.

**`lon`** — Longitude of the search centre, in decimal degrees.

**`radius`** — Search radius in metres. Mutually exclusive with `latSpan`/`lonSpan`. The effective maximum is 20,000 metres; see Suspected Defects for how values exceeding the maximum are (incorrectly) handled.

**`latSpan`** — Latitude span of the search box in decimal degrees. Must be combined with `lonSpan`. Mutually exclusive with `radius`.

**`lonSpan`** — Longitude span of the search box in decimal degrees. Must be combined with `latSpan`. Mutually exclusive with `radius`.

**`time`** — Query time, expressed either as a Unix timestamp in milliseconds or as a string in `yyyy-MM-dd_HH-mm-ss` format (server local time). Defaults to the server's current wall-clock time.

**`maxCount`** — Accepted but not applied; the result set is always the complete set of vehicles within the bounds regardless of this value. See Suspected Defects.

**`includeTrip`** — When true (the default), each matching trip is added to the `references.trips` array and its route and agency are included in `references.routes` and `references.agencies`. When false, no trip, route, or agency data is populated in `references`.

**`includeStatus`** — When true, each list entry contains a `status` sub-object with real-time or schedule-derived vehicle position and related fields. Defaults to false.

**`includeSchedule`** — When true, each list entry contains a `schedule` sub-object with the full stop-time sequence for the active trip. Defaults to false.

**`includeReferences`** — When false, the `references` block is omitted from the response envelope. Defaults to true.

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

**`version`** — Always 2.

**`code`** — HTTP status code (200 for all success and out-of-area responses; 500 for version errors).

**`text`** — Human-readable status message (e.g. `"OK"`).

**`currentTime`** — Server's wall-clock time at the moment the response was generated, in Unix milliseconds.

**`data`** — The result payload; see below.

---

### data

```json
{
  "type": "object",
  "properties": {
    "limitExceeded": { "type": "boolean" },
    "outOfRange":    { "type": "boolean" },
    "list":          { "type": "array", "items": { "type": "object" } },
    "references":    { "type": "object" }
  }
}
```

**`data.limitExceeded`** — Always false for this endpoint. No result cap is enforced.

**`data.outOfRange`** — True when the query centre is outside the geographic area covered by the loaded transit data. The list will be empty.

**`data.list`** — Zero or more trip-details entries; see `data.list[]` below.

**`data.references`** — Shared entity objects referenced by ID from the list entries; see `data.references` below.

---

### data.list[]

```json
{
  "type": "object",
  "properties": {
    "tripId":       { "type": "string" },
    "serviceDate":  { "type": "integer", "description": "Unix ms" },
    "frequency":    { "type": "object" },
    "status":       { "type": "object" },
    "schedule":     { "type": "object" },
    "situationIds": { "type": "array", "items": { "type": "string" } }
  }
}
```

**`data.list[].tripId`** — The ID of the trip the vehicle is actively serving at the query time. This is also the key used to look up the trip in `data.references.trips` (when `includeTrip` is true). For vehicles executing an interlining block, this reflects the trip currently underway, which may belong to a different route than the one that caused the block to fall within the search bounds.

**`data.list[].serviceDate`** — Midnight of the service day for this trip, in Unix milliseconds. Times in the schedule sub-object are expressed as seconds elapsed since this instant.

**`data.list[].frequency`** — Present and non-null only for frequency-based (headway-based) trips. See `data.list[].frequency` below. Null for schedule-based trips.

**`data.list[].status`** — Present when `includeStatus=true`; null otherwise. See `data.list[].status` below.

**`data.list[].schedule`** — Present when `includeSchedule=true`; null otherwise. See `data.list[].schedule` below.

**`data.list[].situationIds`** — IDs of active service alerts that apply to this vehicle journey. References entries in `data.references.situations`.

---

### data.list[].frequency

```json
{
  "type": "object",
  "properties": {
    "startTime": { "type": "integer", "description": "Unix ms" },
    "endTime":   { "type": "integer", "description": "Unix ms" },
    "headway":   { "type": "integer" },
    "exactTimes":{ "type": "integer" }
  }
}
```

**`data.list[].frequency.startTime`** — Start of the frequency window, in Unix milliseconds.

**`data.list[].frequency.endTime`** — End of the frequency window, in Unix milliseconds.

**`data.list[].frequency.headway`** — Target headway in seconds: the expected interval between consecutive departures.

**`data.list[].frequency.exactTimes`** — `0` means departures are approximately spaced by the headway (no fixed timetable). `1` means departures occur at exact multiples of the headway from `startTime`.

---

### data.list[].status

Present only when `includeStatus=true`.

```json
{
  "type": "object",
  "properties": {
    "activeTripId":               { "type": "string" },
    "blockTripSequence":          { "type": "integer" },
    "serviceDate":                { "type": "integer", "description": "Unix ms" },
    "frequency":                  { "type": "object" },
    "scheduledDistanceAlongTrip": { "type": "number" },
    "totalDistanceAlongTrip":     { "type": "number" },
    "position":                   { "type": "object" },
    "orientation":                { "type": "number" },
    "closestStop":                { "type": "string" },
    "closestStopTimeOffset":      { "type": "integer" },
    "nextStop":                   { "type": "string" },
    "nextStopTimeOffset":         { "type": "integer" },
    "phase":                      { "type": "string" },
    "status":                     { "type": "string" },
    "predicted":                  { "type": "boolean" },
    "lastUpdateTime":             { "type": "integer", "description": "Unix ms" },
    "lastLocationUpdateTime":     { "type": "integer", "description": "Unix ms" },
    "lastKnownDistanceAlongTrip": { "type": "number" },
    "lastKnownLocation":          { "type": "object" },
    "lastKnownOrientation":       { "type": "number" },
    "distanceAlongTrip":          { "type": "number" },
    "scheduleDeviation":          { "type": "integer" },
    "vehicleId":                  { "type": "string" },
    "occupancyStatus":            { "type": "string" },
    "occupancyCount":             { "type": "integer" },
    "occupancyCapacity":          { "type": "integer" },
    "vehicleFeatures":            { "type": "array", "items": { "type": "string" } },
    "situationIds":               { "type": "array", "items": { "type": "string" } }
  }
}
```

**`data.list[].status.activeTripId`** — The ID of the trip the vehicle is currently executing. For a vehicle mid-block that is interlining onto a different route, this ID refers to the active trip (which may be on a different route than the queried area expected).

**`data.list[].status.blockTripSequence`** — Zero-based index of the active trip within the vehicle's block for the service day.

**`data.list[].status.serviceDate`** — Midnight of the service day, in Unix milliseconds. Redundant with the outer `serviceDate` field.

**`data.list[].status.frequency`** — Frequency details for frequency-based trips; null otherwise.

**`data.list[].status.scheduledDistanceAlongTrip`** — The distance in metres that the vehicle is scheduled to have travelled along the active trip at the query time. Present only when the trip is in progress.

**`data.list[].status.totalDistanceAlongTrip`** — Total length of the active trip in metres.

**`data.list[].status.position`** — Current vehicle position as `{"lat": number, "lon": number}`. Derived from real-time GPS data when available; otherwise extrapolated from the schedule. Absent if the vehicle has no determinable position.

**`data.list[].status.orientation`** — Vehicle heading in degrees. Convention: 0° = east, 90° = north, 180° = west, 270° = south. Absent if not determinable.

**`data.list[].status.closestStop`** — Stop ID of the stop nearest to the vehicle's current position along the trip. References an entry in `data.references.stops`.

**`data.list[].status.closestStopTimeOffset`** — Seconds between the nearest stop's scheduled time and the vehicle's current position. Positive means the stop is upcoming; negative means it has been passed.

**`data.list[].status.nextStop`** — Stop ID of the next stop the vehicle will serve. Absent if the vehicle has passed the last stop of the trip.

**`data.list[].status.nextStopTimeOffset`** — Seconds until the vehicle is scheduled to reach the next stop.

**`data.list[].status.phase`** — Operational phase of the vehicle's journey (e.g. `"in_progress"`, `"layover_before"`, `"deadhead_before"`). May be empty when not determinable.

**`data.list[].status.status`** — Status modifier string (e.g. `"default"`, `"canceled"`). `"default"` is the normal operating status.

**`data.list[].status.predicted`** — `true` if the position and schedule deviation were derived from real-time GPS data; `false` if derived from the static schedule.

**`data.list[].status.lastUpdateTime`** — Unix milliseconds of the most recent real-time update received from the vehicle. Zero when no real-time data is available.

**`data.list[].status.lastLocationUpdateTime`** — Unix milliseconds of the most recent real-time update that contained location data. Zero when no location update has been received.

**`data.list[].status.lastKnownDistanceAlongTrip`** — The last known distance along trip in metres, as reported by real-time data. Absent when no real-time data is available.

**`data.list[].status.lastKnownLocation`** — The last GPS-reported position as `{"lat": number, "lon": number}`. Differs from `position` in that `position` may be extrapolated forward from this value. Absent when no real-time location data is available.

**`data.list[].status.lastKnownOrientation`** — The last GPS-reported heading in degrees. Absent when no real-time data is available.

**`data.list[].status.distanceAlongTrip`** — Distance in metres the vehicle has travelled along the active trip. Potentially extrapolated forward from the last known position. Absent when not determinable.

**`data.list[].status.scheduleDeviation`** — How many seconds the vehicle is running behind schedule (positive) or ahead of schedule (negative). Zero when only schedule data is available.

**`data.list[].status.vehicleId`** — The agency-assigned vehicle identifier. Empty string when no real-time data is available.

**`data.list[].status.occupancyStatus`** — GTFS-RT occupancy category name (e.g. `"MANY_SEATS_AVAILABLE"`, `"FULL"`). Empty string when occupancy data is unavailable.

**`data.list[].status.occupancyCount`** — Number of passengers on board, if reported. `-1` when unknown.

**`data.list[].status.occupancyCapacity`** — Vehicle passenger capacity, if reported. `-1` when unknown.

**`data.list[].status.vehicleFeatures`** — List of feature tags reported for the vehicle (e.g. accessibility equipment). Empty list when no features are reported.

**`data.list[].status.situationIds`** — IDs of active service alerts applicable to this vehicle's journey. References entries in `data.references.situations`.

---

### data.list[].schedule

Present only when `includeSchedule=true`.

```json
{
  "type": "object",
  "properties": {
    "timeZone":      { "type": "string" },
    "stopTimes":     { "type": "array", "items": { "type": "object" } },
    "previousTripId":{ "type": "string" },
    "nextTripId":    { "type": "string" },
    "frequency":     { "type": "object" }
  }
}
```

**`data.list[].schedule.timeZone`** — IANA timezone identifier for the agency operating this trip (e.g. `"America/Los_Angeles"`). Schedule times are expressed in this timezone.

**`data.list[].schedule.stopTimes`** — Ordered array of scheduled stop visits for the active trip. See `data.list[].schedule.stopTimes[]` below.

**`data.list[].schedule.previousTripId`** — The ID of the preceding trip in the vehicle's block, if the vehicle is interlining (arriving from a different route). Null when there is no preceding interlining trip.

**`data.list[].schedule.nextTripId`** — The ID of the following trip in the vehicle's block, if the vehicle is interlining (continuing onto a different route). Null when there is no following interlining trip.

**`data.list[].schedule.frequency`** — Frequency details for frequency-based trips; null for schedule-based trips.

---

### data.list[].schedule.stopTimes[]

```json
{
  "type": "object",
  "properties": {
    "arrivalTime":         { "type": "integer" },
    "departureTime":       { "type": "integer" },
    "stopId":              { "type": "string" },
    "stopHeadsign":        { "type": "string" },
    "distanceAlongTrip":   { "type": "number" },
    "historicalOccupancy": { "type": "string" }
  }
}
```

**`data.list[].schedule.stopTimes[].arrivalTime`** — Scheduled arrival time at this stop, in seconds elapsed since midnight of the service date. May exceed 86,400 for trips that run past midnight.

**`data.list[].schedule.stopTimes[].departureTime`** — Scheduled departure time at this stop, in seconds elapsed since midnight of the service date.

**`data.list[].schedule.stopTimes[].stopId`** — Stop ID. References an entry in `data.references.stops`.

**`data.list[].schedule.stopTimes[].stopHeadsign`** — Override headsign for this specific stop visit, if any. Empty string when not overridden.

**`data.list[].schedule.stopTimes[].distanceAlongTrip`** — Cumulative distance along the trip's path to this stop, in metres.

**`data.list[].schedule.stopTimes[].historicalOccupancy`** — Historically observed occupancy category at this stop (e.g. `"MANY_SEATS_AVAILABLE"`). Empty string when no historical data is available.

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
    "situations": { "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.references.agencies`** — Agency objects for all agencies operating trips in the result list. Populated when `includeTrip=true`.

**`data.references.routes`** — Route objects for all routes serving trips in the result list. Populated when `includeTrip=true`.

**`data.references.trips`** — Trip objects for all entries in `data.list`. Only populated when `includeTrip=true` (the default). Each trip object contains `id`, `routeId`, `blockId`, `directionId`, `tripHeadsign`, `shapeId`, and related fields.

**`data.references.stops`** — Stop objects referenced by `closestStop` and `nextStop` in status sub-objects, and by stop times in schedule sub-objects. Populated when `includeStatus=true` or `includeSchedule=true`.

**`data.references.situations`** — Service alert objects for all situation IDs referenced in the list entries.
