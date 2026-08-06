# trips-for-location

## Goal in Context

A rider's client app needs to display active transit vehicles whose current positions fall within a specified geographic area, so that the rider can see which services are nearby right now.

## Scope

OneBusAway REST API — the `trips-for-location` endpoint.

## Level

User goal.

## Primary Actor

Rider (via a client application).

## Stakeholders and Interests

- **Rider**: wants a list of active trips whose vehicles are currently within a geographic area they care about, together with enough detail (route, schedule, real-time position) to plan their journey.

## Preconditions

- The server has a loaded transit data bundle.
- The caller supplies at least a centre coordinate (`lat`/`lon`).

## Minimal Guarantees

- The response always has HTTP status 200 for well-formed requests.
- If the query falls outside the service area, the response body contains an empty list and an `outOfRange` flag set to `true`.

## Success Guarantees

- The response contains one entry for every active trip whose vehicle's computed position at the query time is within the specified bounding box.
- If `includeTrip` is `true` (the default), the referenced trip beans are present in the `references` block.
- If `includeStatus` is `true`, each entry carries a real-time status object with the vehicle's computed position, schedule deviation, and related fields.
- If `includeSchedule` is `true`, each entry carries the full stop-time schedule for the active trip.

## Trigger

A GET request to `/api/where/trips-for-location.json` (or `.xml`).

## Main Success Scenario

1. Rider's app sends a GET request specifying a centre coordinate (`lat`/`lon`) and a search area (`radius`, or both `latSpan` and `lonSpan`), along with an optional `time`, `includeTrip`, `includeStatus`, and `includeSchedule`.

2. The server resolves the query time: the `time` parameter if supplied, otherwise the current server clock.

3. The server constructs a bounding box from the centre and the supplied bounds parameters:
   - If `radius` is given, the box encloses a circle of that radius around the centre. Maximum enforced radius is 20,000 m; see Suspected Defects for the broken clamping behaviour. ([`TripsForLocationAction.java#L46`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripsForLocationAction.java#L46), [`SearchBoundsFactory.java#L73–L96`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/impl/SearchBoundsFactory.java#L73-L96))
   - If `latSpan` and `lonSpan` are both given, the box is centred on the coordinate with half those spans as the offsets.
   - If neither form is supplied, the bounding box degenerates to a single point (zero area) and no vehicles are returned.

4. The server identifies candidate block instances (vehicle–day assignments) whose scheduled routes pass through the bounding box at any point within the window that spans 30 minutes before and 10 minutes after the query time. This window accommodates vehicles running late or early. ([`BlockStatusServiceImpl.java#L195–L196`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/blocks/BlockStatusServiceImpl.java#L195-L196))

5. For each candidate, the server computes the vehicle's position at the query time. When real-time GPS data is available the position is GPS-derived and `predicted` is `true`; otherwise the position is extrapolated from the static schedule and `predicted` is `false`.

6. Candidates whose computed position lies outside the bounding box are discarded. Only vehicles physically inside the box at the query time survive. ([`BlockStatusServiceImpl.java#L201–L207`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/blocks/BlockStatusServiceImpl.java#L201-L207))

7. For each surviving vehicle the server assembles a trip details entry containing the active trip ID (in combined `agencyId_entityId` form), service date, frequency info (for headway-based services), and any active service alert IDs.

8. If `includeStatus=true`, a status sub-object is attached to each entry with the vehicle's computed position, orientation, closest and next stop, schedule deviation, occupancy, and related fields.

9. If `includeSchedule=true`, a schedule sub-object is attached containing the full ordered list of stop times for the active trip, the trip's timezone, and the IDs of the adjacent trips in the block (if any).

10. If `includeTrip=true` (default), the trip bean for each active trip is added to the `references` block. Routes, stops (from the status's closest/next stop or from the schedule), agencies, and service alerts are also added to `references` as applicable.

11. The server returns HTTP 200 with a list body. `limitExceeded` is always `false` (see Suspected Defects).

## Extensions

**2a. Version parameter present and not equal to 2:**  
The server returns HTTP 500 with the text "unknown version: _n_".

**3a. `radius` exceeds 20,000 m:**  
Due to a defect in the clamping logic (see Suspected Defects), the search area collapses to a zero-size bounding box and the response contains 0 results rather than results clamped to the 20 km maximum. *Maglev intentionally corrects this — see [Implementation Decisions](#implementation-decisions).*

**3b. `latSpan` or `lonSpan` exceeds the equivalent of a 20 km radius:**  
Due to a defect in the clamping logic (see Suspected Defects), the maximum is not enforced and the unclamped spans are used. *Maglev intentionally corrects this — see [Implementation Decisions](#implementation-decisions).*

**3c. `lat` and `lon` are absent:**  
The server treats both as 0 (the default for a double), constructs a bounding box centred on (0, 0), and applies normal processing. If (0, 0) is outside the service area, `outOfRange` is `true` in the response.

**6a. The query lies outside the service area:**  
The `OutOfServiceAreaServiceException` is caught, and the server returns HTTP 200 with an empty list and `outOfRange: true`. ([`TripsForLocationAction.java#L137–L139`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripsForLocationAction.java#L137-L139))

**6b. No vehicles are found within the box:**  
The server returns HTTP 200 with an empty list, `limitExceeded: false`, and `outOfRange: false`.

## Suspected Defects

### Defects that affect the use case

**`maxCount` is accepted but never applied — `TripStatusBeanServiceImpl`**

The action class accepts a `maxCount` parameter and stores it in the query object, but [`TripStatusBeanServiceImpl.getTripsForBounds`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L222-L228) never reads `query.getMaxCount()`. All matching trips are returned regardless of the value supplied, and `limitExceeded` is always `false`. The Go implementation should decide whether to honour `maxCount` or treat it as unsupported.

**`radius` exceeding 20,000 m collapses the search area to a point — `SearchBoundsFactory`**

In [`SearchBoundsFactory.createBounds()`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/impl/SearchBoundsFactory.java#L87-L95), when the computed bounds exceed the maximum, the clamping branch rebuilds the bounds using `_latSpan / 2` and `_lonSpan / 2`. These fields hold the raw `latSpan`/`lonSpan` query parameters, which are zero for radius-based requests. The result is a bounding box of zero area, so no vehicles pass the position check and 0 results are returned. The intended behaviour is presumably to clamp to the 20 km maximum circle. The Go implementation should clamp the radius to 20 km and use that clamped circle. *Maglev intentionally corrects this — see [Implementation Decisions](#implementation-decisions).*

**`latSpan`/`lonSpan` exceeding the maximum are not clamped — `SearchBoundsFactory`**

In the same [`SearchBoundsFactory.createBounds()`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/impl/SearchBoundsFactory.java#L87-L95) clamping block, the clamped span values are computed into local variables (`latSpan`, `lonSpan`) but the subsequent call to `boundsFromLatLonOffset` uses `_latSpan / 2` and `_lonSpan / 2` — the original, unclamped field values — so the limit is silently bypassed. The Go implementation should apply the clamped span values when the supplied area exceeds the maximum. *Maglev intentionally corrects this — see [Implementation Decisions](#implementation-decisions).*

**`includeTrip` default is documented as `false` but the code defaults to `true` — `TripsForLocationAction`**

The field is initialised to `true` at [`TripsForLocationAction.java#L57`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripsForLocationAction.java#L57), so trip beans are included in references by default. The existing method documentation says "Defaults to false", which is incorrect. The Go implementation should default to `true` (matching the code, not the docs) to avoid breaking clients that rely on the current behaviour.

### Implementation defects only

**Wrong null guard in `getBlockLocationsAsTripDetails` — `TripStatusBeanServiceImpl`**

At [`TripStatusBeanServiceImpl.java#L424`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L424), the guard reads `if (tripDetails != null)` where the intent is clearly `if (details != null)`. The `tripDetails` list is never null (it was just initialised), so the check always passes and null `TripDetailsBean` entries can be added to the result list. This is harmless in practice because `BeanFactoryV2.getTripDetailsResponse` filters out null entries before constructing the response, but a clean reimplementation should check `details != null`.

## Implementation Decisions

**Bounding-box clamping is fixed, not replicated (deviates from legacy).**

The legacy implementation has two documented defects (see [Suspected Defects](#suspected-defects)): a `radius` over 20,000 m collapses the search area to a zero-size box instead of clamping to the 20 km maximum, and `latSpan`/`lonSpan` over the equivalent maximum are computed but silently not applied. Maglev intentionally corrects both in `BoundsFromParams` (`internal/gtfs/location_params.go`): when clamping is requested, a radius exceeding `MaxSearchRadiusInMeters` (20,000 m) is clamped to that maximum before computing the circle. For span-based requests, `latSpan`/`lonSpan` are clamped to the degree extents of that same 20,000 m circle at the request's latitude, derived via great-circle math rather than a fixed degree constant — at most latitudes this works out to roughly 0.3–0.4° of latitude, with the longitude equivalent scaling by `1/cos(latitude)` and widening toward the poles. Clients should expect results consistent with a 20 km cap, not the legacy zero-result or unclamped-span behaviour.

## Open Questions

None.

## Request Parameters

```json
{
  "type": "object",
  "required": ["lat", "lon"],
  "properties": {
    "lat":               { "type": "number", "description": "Latitude of the search centre" },
    "lon":               { "type": "number", "description": "Longitude of the search centre" },
    "radius":            { "type": "number", "description": "Search radius in metres. Maximum 20,000 m (see Suspected Defects for clamping behaviour). Mutually exclusive with latSpan/lonSpan." },
    "latSpan":           { "type": "number", "description": "Total latitude extent of the search box in degrees. Must be used together with lonSpan." },
    "lonSpan":           { "type": "number", "description": "Total longitude extent of the search box in degrees. Must be used together with latSpan." },
    "time":              { "type": "string", "description": "Query time. Accepts Unix ms as a plain integer, or the string format yyyy-MM-dd_HH-mm-ss. Defaults to the current server time." },
    "maxCount":          { "type": "integer", "description": "Accepted but not applied — all matching trips are always returned (see Suspected Defects)." },
    "includeTrip":       { "type": "boolean", "default": true,  "description": "When true, full trip beans are added to the references block." },
    "includeStatus":     { "type": "boolean", "default": false, "description": "When true, a real-time status object is attached to each trip details entry." },
    "includeSchedule":   { "type": "boolean", "default": false, "description": "When true, the full stop-time schedule is attached to each trip details entry." },
    "includeReferences": { "type": "boolean", "default": true,  "description": "When false, the references block is omitted from the response." },
    "key":               { "type": "string",  "description": "API key." },
    "version":           { "type": "integer", "default": 2,    "description": "API version. Only version 2 is supported; any other value returns HTTP 500." }
  }
}
```

**`lat`** — WGS-84 latitude of the geographic centre of the search.

**`lon`** — WGS-84 longitude of the geographic centre of the search.

**`radius`** — If supplied, the search area is the bounding box of a circle with this radius (in metres) centred on `lat`/`lon`. Maximum is 20,000 m (20 km), though the clamping is defective (see Suspected Defects). Cannot be combined with `latSpan`/`lonSpan`; if both are present, `radius` takes precedence.

**`latSpan`** — Total height of the search bounding box in degrees of latitude. Must be used with `lonSpan`. The box is centred on `lat`/`lon` so each side extends `latSpan / 2` degrees.

**`lonSpan`** — Total width of the search bounding box in degrees of longitude. Must be used with `latSpan`.

**`time`** — The point in time for which to query vehicle positions. Accepts a Unix millisecond timestamp (plain integer) or a `yyyy-MM-dd_HH-mm-ss` string. If omitted, the current server time is used.

**`maxCount`** — Nominally a cap on the number of results. Currently accepted and stored but never applied; all matching trips are returned and `limitExceeded` is always `false`.

**`includeTrip`** — Controls whether trip beans appear in the `references` block. Defaults to `true`.

**`includeStatus`** — Controls whether each list entry includes a `status` sub-object with the vehicle's real-time (or schedule-derived) position and timing. Defaults to `false`.

**`includeSchedule`** — Controls whether each list entry includes a `schedule` sub-object with the full ordered stop-time list. Defaults to `false`.

**`includeReferences`** — When `false`, the `references` block is omitted. Defaults to `true`.

**`key`** — API authentication key.

**`version`** — Must be 2 (or omitted). Any other value triggers an HTTP 500 error response.

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

**`version`** — API version echoed from the request (or the default of 2).

**`code`** — HTTP-style status code (`200` on success, `500` for unsupported version).

**`text`** — Human-readable status text (e.g. `"OK"`).

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`data`** — Container object described in the sections below.

---

### data

```json
{
  "type": "object",
  "properties": {
    "list":          { "type": "array",   "description": "Array of tripDetails objects" },
    "limitExceeded": { "type": "boolean", "description": "Always false — maxCount is never applied" },
    "outOfRange":    { "type": "boolean" },
    "references":    { "type": "object"  }
  }
}
```

**`data.list`** — Array of trip details entries; one entry per active trip whose vehicle is within the search area.

**`data.limitExceeded`** — Always `false`. The `maxCount` parameter is accepted but not enforced; this flag is never set to `true`.

**`data.outOfRange`** — `true` when the query coordinates fall entirely outside the service area covered by the loaded transit bundle.

**`data.references`** — Present when `includeReferences=true` (the default). Contains trips, routes, stops, agencies, and situations referenced by the list entries.

---

### data.list[]

```json
{
  "type": "object",
  "properties": {
    "tripId":       { "type": "string"  },
    "serviceDate":  { "type": "integer", "description": "Unix ms" },
    "frequency":    { "type": "object",  "description": "Null for fixed-schedule trips" },
    "status":       { "type": "object",  "description": "Present only when includeStatus=true" },
    "schedule":     { "type": "object",  "description": "Present only when includeSchedule=true" },
    "situationIds": { "type": "array", "items": { "type": "string" } }
  }
}
```

**`data.list[].tripId`** — Combined `agencyId_entityId` identifier of the scheduled trip the vehicle is executing.

**`data.list[].serviceDate`** — Unix millisecond timestamp of midnight at the start of the service day for this trip. For trips that extend past midnight, the service date is the previous calendar day: the block search resolves candidate service dates against the block's scheduled time range (which can exceed 24:00:00), not against the calendar date of the query time. [Source: [`BlockStatusServiceImpl.java#L192-L210`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/blocks/BlockStatusServiceImpl.java#L192-L210), [`BlockCalendarServiceImpl.java#L247-L265`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/blocks/BlockCalendarServiceImpl.java#L247-L265)]

**`data.list[].frequency`** — For headway-based (frequency) trips, a frequency object giving start time, end time, headway, and exactTimes. `null` for ordinary fixed-schedule trips.

**`data.list[].status`** — Real-time status object. Present only when `includeStatus=true`. See `data.list[].status` section below.

**`data.list[].schedule`** — Stop-time schedule object. Present only when `includeSchedule=true`. See `data.list[].schedule` section below.

**`data.list[].situationIds`** — Array of combined service-alert IDs currently affecting this trip. Corresponding alert objects appear in `data.references.situations`. Empty array when no alerts apply.

---

### data.list[].status

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
    "scheduleDeviation":             { "type": "integer" },
    "distanceAlongTrip":             { "type": "number" },
    "vehicleId":                     { "type": "string" },
    "occupancyStatus":               { "type": "string" },
    "occupancyCount":                { "type": "integer" },
    "occupancyCapacity":             { "type": "integer" },
    "vehicleFeatures":               { "type": "array", "items": { "type": "string" } },
    "situationIds":                  { "type": "array", "items": { "type": "string" } }
  }
}
```

**`data.list[].status.activeTripId`** — Combined ID of the trip the vehicle is currently executing. For ordinary trips this matches the outer `tripId`; for interlined blocks the vehicle may be serving a different trip than the one recorded in the outer entry.

**`data.list[].status.blockTripSequence`** — Zero-based index of the active trip within the ordered sequence of trips in the vehicle's block for the service day. `-1` if the active trip could not be determined.

**`data.list[].status.serviceDate`** — Unix milliseconds of midnight at the start of the service day. Duplicates `data.list[].serviceDate`, including the previous-calendar-day behaviour for trips extending past midnight.

**`data.list[].status.frequency`** — Frequency object for headway-based trips; `null` otherwise.

**`data.list[].status.scheduledDistanceAlongTrip`** — How far, in metres, the vehicle should have travelled along the active trip according to the schedule at the query time.

**`data.list[].status.totalDistanceAlongTrip`** — Total length of the active trip in metres.

**`data.list[].status.position`** — Current vehicle position (lat/lon). When real-time data is available this is the GPS position; otherwise it is extrapolated from the schedule. May be absent if the vehicle's position cannot be computed.

**`data.list[].status.orientation`** — Compass heading in degrees, where 0° is east, 90° is north, 180° is west, and 270° is south. Absent if unknown.

**`data.list[].status.closestStop`** — Combined ID of the stop nearest to the vehicle's current position along the trip's stop sequence. The corresponding stop bean is in `data.references.stops`.

**`data.list[].status.closestStopTimeOffset`** — Seconds between the current position and the closest stop in scheduled time. Positive means the stop is upcoming; negative means it has been passed.

**`data.list[].status.nextStop`** — Combined ID of the next stop the vehicle has not yet departed. Absent once the vehicle has passed the last stop.

**`data.list[].status.nextStopTimeOffset`** — Scheduled seconds until the vehicle reaches the next stop.

**`data.list[].status.phase`** — String describing the current journey phase (e.g. `"in_progress"`, `"layover_before"`, `"deadhead_before"`).

**`data.list[].status.status`** — String status modifier (e.g. `"default"`).

**`data.list[].status.predicted`** — `true` if real-time GPS data was used to compute the vehicle's position; `false` if the position was derived from the static schedule.

**`data.list[].status.lastUpdateTime`** — Unix milliseconds of the last real-time data message received from the vehicle. `0` if no real-time data has ever been received.

**`data.list[].status.lastLocationUpdateTime`** — Unix milliseconds of the last real-time message that included a GPS location. `0` if no location update has been received.

**`data.list[].status.lastKnownLocation`** — The last GPS position received from the vehicle, without any forward extrapolation.

**`data.list[].status.lastKnownDistanceAlongTrip`** — The last known distance-along-trip value received from the vehicle in real time.

**`data.list[].status.lastKnownOrientation`** — The last known heading received from the vehicle in real time.

**`data.list[].status.scheduleDeviation`** — Signed integer seconds of schedule deviation: positive means running late, negative means running early. `0` when `predicted` is `false`.

**`data.list[].status.distanceAlongTrip`** — Distance in metres the vehicle has progressed along the active trip. May be extrapolated forward from the last real-time reading.

**`data.list[].status.vehicleId`** — Combined ID of the physical vehicle, when known from real-time data.

**`data.list[].status.occupancyStatus`** — GTFS-RT `OccupancyStatus` value as a string (e.g. `"MANY_SEATS_AVAILABLE"`, `"FEW_SEATS_AVAILABLE"`, `"FULL"`). Empty string when occupancy data is unavailable.

**`data.list[].status.occupancyCount`** — Raw passenger count, or `-1` if not reported.

**`data.list[].status.occupancyCapacity`** — Vehicle capacity, or `-1` if not reported.

**`data.list[].status.vehicleFeatures`** — Array of feature strings reported by the vehicle (e.g. accessibility equipment). Empty array when no features are reported.

**`data.list[].status.situationIds`** — Service alert IDs applicable specifically to this vehicle journey (may overlap with the outer entry's `situationIds`).

---

### data.list[].schedule

```json
{
  "type": "object",
  "properties": {
    "timeZone":      { "type": "string" },
    "frequency":     { "type": "object" },
    "stopTimes":     { "type": "array" },
    "previousTripId":{ "type": "string" },
    "nextTripId":    { "type": "string" }
  }
}
```

**`data.list[].schedule.timeZone`** — IANA timezone identifier for the trip (e.g. `"America/Los_Angeles"`).

**`data.list[].schedule.frequency`** — Frequency object for headway-based trips; `null` otherwise.

**`data.list[].schedule.stopTimes`** — Ordered array of stop-time entries for every stop in the active trip. See `data.list[].schedule.stopTimes[]` below.

**`data.list[].schedule.previousTripId`** — Combined ID of the trip that directly precedes this trip in the block (i.e. the same vehicle's prior run). Absent when this is the first trip of the block or no preceding trip exists.

**`data.list[].schedule.nextTripId`** — Combined ID of the trip that directly follows this trip in the block (i.e. the same vehicle's next run). Absent when this is the last trip of the block.

---

### data.list[].schedule.stopTimes[]

```json
{
  "type": "object",
  "properties": {
    "stopId":             { "type": "string" },
    "arrivalTime":        { "type": "integer" },
    "departureTime":      { "type": "integer" },
    "stopHeadsign":       { "type": "string" },
    "distanceAlongTrip":  { "type": "number" },
    "historicalOccupancy":{ "type": "string" }
  }
}
```

**`data.list[].schedule.stopTimes[].stopId`** — Combined ID of the stop. The corresponding stop bean is in `data.references.stops`.

**`data.list[].schedule.stopTimes[].arrivalTime`** — Scheduled arrival time expressed as seconds elapsed since midnight of the service date. Values exceeding 86,400 indicate arrivals on the following calendar day that belong to this service day.

**`data.list[].schedule.stopTimes[].departureTime`** — Scheduled departure time in the same form as `arrivalTime`.

**`data.list[].schedule.stopTimes[].stopHeadsign`** — Destination text shown on the vehicle specifically at this stop, overriding the trip-level headsign. Empty string when no override applies.

**`data.list[].schedule.stopTimes[].distanceAlongTrip`** — Distance in metres from the start of the trip to this stop along the vehicle's path.

**`data.list[].schedule.stopTimes[].historicalOccupancy`** — Historical average occupancy at this stop, expressed as a GTFS-RT `OccupancyStatus` string. Empty string when no historical data is available.

---

### data.references

```json
{
  "type": "object",
  "properties": {
    "agencies":   { "type": "array" },
    "routes":     { "type": "array" },
    "stops":      { "type": "array" },
    "trips":      { "type": "array" },
    "situations": { "type": "array" }
  }
}
```

**`data.references.agencies`** — Agency beans for all agencies that own routes appearing in the response.

**`data.references.routes`** — Route beans for all routes whose trips appear in the response.

**`data.references.stops`** — Stop beans for all stops referenced by the response (closest stop and next stop from status objects; all stops in schedule stop-time arrays). Each bean's `direction` field is taken from the feed when defined, otherwise inferred from shape geometry — see [[Stop-Direction-Calculation]].

**`data.references.trips`** — Trip beans for all trips referenced in list entries. Populated only when `includeTrip=true` (the default).

**`data.references.situations`** — Service alert beans for all situation IDs referenced in the response.
