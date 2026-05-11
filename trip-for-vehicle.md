# trip-for-vehicle

## Goal in Context

A rider's client app has a vehicle ID (for example, from a nearby vehicle list or a real-time display) and wants to see the current trip that vehicle is executing, together with its real-time status and — optionally — the stop schedule and static trip record.

## Scope

OneBusAway public REST API, v2.

## Level

User goal.

## Primary Actor

Rider.

## Stakeholders and Interests

**Rider** — wants to know which trip a specific vehicle is currently operating, where the vehicle is right now, and which stops it will serve so they can decide whether to board.

## Preconditions

- The caller supplies a valid API key.
- The vehicle ID supplied is in the combined `{agencyId}_{vehicleId}` form.

## Minimal Guarantees

The API always returns a well-formed JSON (or XML) envelope containing a `code`, `text`, and `currentTime`.

## Success Guarantees

- A 200 response is returned whose `data.entry` describes the trip the vehicle is currently executing.
- The entry always includes `tripId` and `serviceDate`.
- The `status` block is present by default and contains the vehicle's real-time or schedule-derived position.
- Referenced entities (trip, route, agency, stops, alerts) are included in `data.references` according to what sections are present in the entry.

## Trigger

An HTTP GET to `/api/where/trip-for-vehicle/{id}.json` (or `.xml`).

## Main Success Scenario

1. The caller issues a GET request with the required `{id}` path segment set to the vehicle's combined ID (`{agencyId}_{vehicleId}`). The agency portion is everything before the first underscore; the entity portion is everything after. [Source: `AgencyAndIdLibrary.convertFromString` — the combined-ID parsing rule.]
2. The system looks up the block that the vehicle was executing at the requested time (defaulting to now). This lookup is based on real-time tracking data, not the static schedule: the vehicle must be actively tracked. If the vehicle has no active block at the given time, the lookup returns nothing.
3. From the block, the system identifies which trip the vehicle is currently serving (the active trip).
4. The response entry is assembled from the active trip's data:
   - `tripId` and `serviceDate` are always included.
   - The `status` block is included by default (unless `includeStatus=false`) and contains the vehicle's current real-time or schedule-derived position, phase, schedule deviation, occupancy, and service alerts.
   - The `schedule` block (stop times for the active trip) is included only when `includeSchedule=true`.
   - The scheduled trip record is added to the references block from the status path — the active trip's full trip element (route, agency) is always added to `data.references` when the status block is present; when `includeTrip=true` the same trip is additionally added explicitly.
5. Service alerts that apply to this vehicle journey at the requested time are collected and their IDs are included in both `data.entry.situationIds` and (redundantly) inside `data.entry.status.situationIds` when `includeStatus=true`. The full alert objects are added to `data.references`.
6. The system returns HTTP 200 with the assembled entry.

## Extensions

**2a. Vehicle not found or not actively tracked at the given time:**
  The system returns HTTP 404 with `code: 404` and `text: "resource not found"`. This occurs whenever the vehicle ID is unrecognised or the vehicle has no block location record at the requested time (including when querying a past or future time for which no real-time snapshot exists). [Source: [`TripForVehicleAction.java#L104–L105`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripForVehicleAction.java#L104-L105)]

**2b. Vehicle has a block location but no active trip at the given time:**
  The service returns null for the block location (treated identically to 2a): HTTP 404. [Source: [`TripStatusBeanServiceImpl.java#L213–L218`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L213-L218)]

**2c. Unsupported API version requested via `version` parameter:**
  The system returns HTTP 500 with `code: 500` and `text: "unknown version: N"`. Only version 2 is supported. [Source: [`TripForVehicleAction.java#L87–L88`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripForVehicleAction.java#L87-L88)]

**4a. `includeStatus=false` is passed:**
  The `status` block is omitted from the entry. The active trip is no longer added to references via the status path, and if `includeTrip` is also false (the default), the references block will contain no trip or route elements. [Source: [`TripStatusBeanServiceImpl.java#L476–L480`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L476-L480)]

**4b. `includeTrip=true` is passed:**
  The full trip record is explicitly fetched and added to `data.references.trips`. Because the status block also adds the active trip to references when present, this parameter only makes a difference when `includeStatus=false`. [Source: [`TripStatusBeanServiceImpl.java#L464–L468`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L464-L468)]

**4c. `includeSchedule=true` is passed:**
  The `schedule` block is populated with the active trip's stop times, its timezone, and (when applicable) IDs for the preceding and following trips in the same block. Each stop in the schedule is added to `data.references.stops`. [Source: [`TripStatusBeanServiceImpl.java#L470–L475`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L470-L475)]

**4d. Trip is part of a frequency-based (headway) service:**
  A `frequency` object is included in both `data.entry` and `data.entry.status` (if present), describing the headway and the time window during which the frequency applies.

**5a. No service alerts apply:**
  `data.entry.situationIds` is absent from the response (not an empty array).

**5b. Vehicle is executing a trip on a different route than the block was originally selected for (interlining):**
  The entry's `tripId` and `status.activeTripId` always reflect the trip the vehicle is currently executing — they may belong to a route different from any route that caused this vehicle to appear in a prior query. This is expected behaviour: a single block can contain trips across multiple routes.

## Suspected Defects

### Defects that affect the use case

**`occupancyCount` and `occupancyCapacity` sentinel values expose implementation details**

When no real-time occupancy data is available for the vehicle, `TripStatusBeanServiceImpl` sets `occupancyCount = -1` and `occupancyCapacity = -1` ([source: `TripStatusBeanServiceImpl.java#L404–L407`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L404-L407)). These sentinel integers are published in the JSON response — the fields are present with a value of `-1` rather than being absent. The same applies when a vehicle occupancy record exists but has no capacity value ([source: `TripStatusBeanServiceImpl.java#L402–L407`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L402-L407)). A caller cannot distinguish "no data" from a legitimate count of −1 passengers. The intended behaviour is to omit the field when data is unavailable.

The Go implementation should omit `occupancyCount` and `occupancyCapacity` from the response when no occupancy data is available, and omit `occupancyCapacity` specifically when the record exists but has no capacity.

### Implementation defects only

**`TripStatusBeanServiceImpl` sets `lastKnownLocation` twice**

`getBlockLocationAsStatusBean` calls `bean.setLastKnownLocation(blockLocation.getLastKnownLocation())` on lines 271 and 276 with the same value ([source: `TripStatusBeanServiceImpl.java#L270–L278`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L270-L278)). The second assignment is a duplicate with no observable effect.

**`getBlockLocationsAsTripDetails` null-checks the wrong variable**

The list-version method (used by other endpoints, not `trip-for-vehicle`) checks `if (tripDetails != null)` instead of `if (details != null)` ([source: `TripStatusBeanServiceImpl.java#L424`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L424)). The list is never null at that point, so the guard is vacuous and `null` can be appended to the result list when a block location has no active trip. This does not affect `trip-for-vehicle`, which calls the single-result `getTripForVehicle` path.

**`@RequiredFieldValidator` on `setId` is unreachable**

The `@RequiredFieldValidator` annotation on `TripForVehicleAction.setId` is dead code ([source: `TripForVehicleAction.java#L59–L62`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripForVehicleAction.java#L59-L62)). The vehicle ID is embedded in the URL path; when it is absent, the Struts2 framework returns a container-level 404 before the action is reached, so `hasErrors()` is never checked for this parameter.

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Path parameter. Combined vehicle ID in {agencyId}_{vehicleId} form."
    },
    "key": {
      "type": "string",
      "description": "API key."
    },
    "includeTrip": {
      "type": "boolean",
      "default": false
    },
    "includeSchedule": {
      "type": "boolean",
      "default": false
    },
    "includeStatus": {
      "type": "boolean",
      "default": true
    },
    "time": {
      "type": "string",
      "description": "Point in time for the lookup. Accepts Unix milliseconds (integer string) or yyyy-MM-dd_HH-mm-ss. Defaults to server current time."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "When false, the references block is omitted from the response."
    },
    "version": {
      "type": "integer",
      "default": 2,
      "description": "API version. Only 2 is supported; other values produce a 500 error."
    }
  }
}
```

**`id`** — Combined vehicle identifier, encoded directly in the URL path: `/api/where/trip-for-vehicle/{id}.json`. The agency portion is everything before the first underscore; the rest is the entity ID. Example: `1_4210`.

**`key`** — API authentication key. Required on all requests.

**`includeTrip`** — When `true`, the full trip element for the active trip is added to `data.references.trips` (together with its route and agency). When `false` (the default), the trip is still added to references via the status block if `includeStatus=true`, so this flag only matters when `includeStatus=false`.

**`includeSchedule`** — When `true`, the `data.entry.schedule` block is populated with the active trip's stop times. Defaults to `false`.

**`includeStatus`** — When `true` (the default), the `data.entry.status` block is populated with the vehicle's real-time or schedule-derived position and related data. Set to `false` to suppress real-time data entirely.

**`time`** — The point in time used for the vehicle lookup. Accepts either a Unix timestamp in milliseconds or the formatted string `yyyy-MM-dd_HH-mm-ss`. When omitted, the server's current time is used. Useful for testing against historical snapshots.

**`includeReferences`** — When `false`, the `data.references` object is omitted from the response.

**`version`** — Must be `2`. Any other value results in a 500 response.

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
    "data": {
      "type": "object",
      "properties": {
        "entry":      { "$ref": "#/definitions/tripDetails" },
        "references": { "$ref": "#/definitions/references" }
      }
    }
  }
}
```

**`code`** — HTTP-style status code mirrored inside the body (200 on success).

**`text`** — Human-readable status description.

**`version`** — API version of the response (always 2).

**`currentTime`** — Server wall-clock time at response generation, in Unix milliseconds.

**`data.entry`** — The trip details object; see below.

**`data.references`** — Dereferencing map for IDs that appear in the entry; absent when `includeReferences=false`.

### data.entry

```json
{
  "type": "object",
  "properties": {
    "tripId":       { "type": "string" },
    "serviceDate":  { "type": "integer", "description": "Unix ms" },
    "frequency":    { "type": "object", "description": "Present only for frequency-based trips." },
    "status":       { "type": "object", "description": "Present unless includeStatus=false." },
    "schedule":     { "type": "object", "description": "Present only when includeSchedule=true." },
    "situationIds": { "type": "array",  "items": { "type": "string" }, "description": "Absent when no alerts apply." }
  }
}
```

**`data.entry.tripId`** — Combined trip ID (`{agencyId}_{tripId}`) of the trip the vehicle is currently executing.

**`data.entry.serviceDate`** — Unix milliseconds for midnight at the start of the service date on which this trip runs. For trips that extend past midnight, the service date is the previous calendar day.

**`data.entry.frequency`** — For frequency-based (headway) schedules: an object with `startTime`, `endTime`, and `headway` (all in seconds since midnight of the service date). Absent for timetable-based trips.

**`data.entry.status`** — Real-time vehicle status object. Present unless `includeStatus=false`; see `data.entry.status` below.

**`data.entry.schedule`** — Stop-time schedule for the active trip. Present only when `includeSchedule=true`; see `data.entry.schedule` below.

**`data.entry.situationIds`** — Array of combined service-alert IDs that currently apply to this vehicle journey. Absent (not an empty array) when no alerts apply. The corresponding alert objects are in `data.references.situations`.

### data.entry.status

```json
{
  "type": "object",
  "properties": {
    "activeTripId":                { "type": "string" },
    "blockTripSequence":           { "type": "integer" },
    "serviceDate":                 { "type": "integer", "description": "Unix ms" },
    "frequency":                   { "type": "object" },
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
    "situationIds":                { "type": "array", "items": { "type": "string" } }
  }
}
```

**`data.entry.status.activeTripId`** — Combined trip ID of the trip the vehicle is currently executing. For a vehicle undergoing interlining, this is the currently active trip, which may belong to a different route than the block was originally looked up for.

**`data.entry.status.blockTripSequence`** — Zero-based index of the active trip within the ordered sequence of trips in the vehicle's block for this service day. Absent when no active trip can be determined.

**`data.entry.status.serviceDate`** — Unix milliseconds for midnight of the service date. Duplicates `data.entry.serviceDate`.

**`data.entry.status.frequency`** — For frequency-based trips: the applicable headway window. Absent for timetable-based trips.

**`data.entry.status.scheduledDistanceAlongTrip`** — Metres the vehicle is scheduled to have progressed along the active trip at the requested time. Absent when no position can be computed.

**`data.entry.status.totalDistanceAlongTrip`** — Total length of the active trip in metres.

**`data.entry.status.position`** — Current vehicle position as `{lat, lon}`. For vehicles with real-time tracking this reflects the latest GPS fix (possibly extrapolated forward); for untracked vehicles this reflects the scheduled position. Absent when no position can be computed.

**`data.entry.status.orientation`** — Vehicle heading in degrees, where 0° is east, 90° is north, 180° is west, and 270° is south. Absent when orientation data is unavailable.

**`data.entry.status.closestStop`** — Combined stop ID of the stop on the active trip's sequence that is geographically or temporally nearest to the vehicle's current position.

**`data.entry.status.closestStopTimeOffset`** — Seconds between the vehicle's current position and the closest stop's scheduled time. Positive means the stop is still ahead; negative means it has been passed.

**`data.entry.status.nextStop`** — Combined stop ID of the next stop the vehicle will serve. Absent once the vehicle has passed the last stop on the trip.

**`data.entry.status.nextStopTimeOffset`** — Seconds until the vehicle reaches the next stop according to schedule.

**`data.entry.status.phase`** — The vehicle's current operational phase as a lowercase string. Possible values: `at_base`, `deadhead_before`, `layover_before`, `in_progress`, `deadhead_during`, `layover_during`, `deadhead_after`, `layover_after`, `unknown`. [Source: [`EVehiclePhase.java#L105–L107`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-realtime-api/src/main/java/org/onebusaway/realtime/api/EVehiclePhase.java#L105-L107)]

**`data.entry.status.status`** — A modifier string. Normally `"default"`; `"canceled"` when the trip has been cancelled.

**`data.entry.status.predicted`** — `true` when real-time tracking data is available and used for position, schedule deviation, and time estimates; `false` when values are derived from the static schedule.

**`data.entry.status.lastUpdateTime`** — Unix milliseconds of the most recent vehicle update of any kind. Absent when no update has been received.

**`data.entry.status.lastLocationUpdateTime`** — Unix milliseconds of the most recent vehicle update that contained a location fix. Absent when no location update has been received.

**`data.entry.status.lastKnownDistanceAlongTrip`** — Metres along the active trip as reported in the most recent real-time update, before any forward extrapolation. Absent when not supplied by the real-time source.

**`data.entry.status.lastKnownLocation`** — `{lat, lon}` of the most recent confirmed GPS fix, before any extrapolation. May be absent.

**`data.entry.status.lastKnownOrientation`** — Heading in degrees as reported in the most recent real-time update. Absent when not supplied.

**`data.entry.status.scheduleDeviation`** — Seconds the vehicle is running late (positive) or early (negative) relative to schedule. Absent when no real-time data is available (in that case the vehicle is assumed on time).

**`data.entry.status.distanceAlongTrip`** — Metres the vehicle has actually progressed along the active trip, potentially extrapolated forward from the last known position. Absent when not determinable.

**`data.entry.status.vehicleId`** — Combined vehicle ID (`{agencyId}_{vehicleId}`).

**`data.entry.status.occupancyStatus`** — GTFS-RT `OccupancyStatus` name string (e.g., `MANY_SEATS_AVAILABLE`, `FEW_SEATS_AVAILABLE`, `FULL`). Absent when no occupancy data is available. **Note:** due to a defect, `occupancyCount` and `occupancyCapacity` are published as `-1` rather than being absent when data is unavailable; see Suspected Defects.

**`data.entry.status.occupancyCount`** — Raw passenger count from real-time data. Published as `-1` when no occupancy record exists (defect — should be absent).

**`data.entry.status.occupancyCapacity`** — Vehicle capacity from real-time data. Published as `-1` when no record exists or the record carries no capacity value (defect — should be absent).

**`data.entry.status.vehicleFeatures`** — List of feature identifiers reported for this vehicle (e.g., accessibility features). Absent when the real-time source supplies none.

**`data.entry.status.situationIds`** — Combined IDs of service alerts applicable to this vehicle journey at the requested time. Absent when none apply.

### data.entry.schedule

Present only when `includeSchedule=true`.

```json
{
  "type": "object",
  "properties": {
    "timeZone":      { "type": "string" },
    "stopTimes":     { "type": "array", "items": { "$ref": "#/definitions/tripStopTime" } },
    "previousTripId":{ "type": "string" },
    "nextTripId":    { "type": "string" },
    "frequency":     { "type": "object" }
  }
}
```

**`data.entry.schedule.timeZone`** — IANA timezone identifier for the trip (e.g., `America/Los_Angeles`). Times in `stopTimes` are expressed in this timezone.

**`data.entry.schedule.stopTimes`** — Ordered list of stop times for each stop the active trip serves.

**`data.entry.schedule.previousTripId`** — Combined trip ID of the preceding trip in the same block (the one whose vehicle becomes this trip's vehicle). Present only when a preceding block trip exists on a different route (i.e., when the vehicle is interlining in from another route).

**`data.entry.schedule.nextTripId`** — Combined trip ID of the following trip in the same block. Present only when a following block trip exists on a different route (interlining out to another route).

**`data.entry.schedule.frequency`** — For frequency-based trips: the headway window object. Absent for timetable-based trips.

### data.entry.schedule.stopTimes[]

```json
{
  "type": "object",
  "properties": {
    "arrivalTime":          { "type": "integer" },
    "departureTime":        { "type": "integer" },
    "stopId":               { "type": "string" },
    "stopHeadsign":         { "type": "string" },
    "distanceAlongTrip":    { "type": "number" },
    "historicalOccupancy":  { "type": "string" }
  }
}
```

**`data.entry.schedule.stopTimes[].arrivalTime`** — Scheduled arrival time as seconds elapsed since midnight of the service date. Values may exceed 86,400 for trips that extend past midnight.

**`data.entry.schedule.stopTimes[].departureTime`** — Scheduled departure time in the same units. These are the static scheduled times and do not incorporate real-time predictions.

**`data.entry.schedule.stopTimes[].stopId`** — Combined stop ID (`{agencyId}_{stopId}`). The corresponding stop element is in `data.references.stops`.

**`data.entry.schedule.stopTimes[].stopHeadsign`** — Per-stop headsign override, when the GTFS data specifies a different destination text for this particular stop. Absent when no per-stop headsign is defined.

**`data.entry.schedule.stopTimes[].distanceAlongTrip`** — Distance in metres from the start of the trip to this stop.

**`data.entry.schedule.stopTimes[].historicalOccupancy`** — Historical (not real-time) occupancy level at this stop as a GTFS-RT `OccupancyStatus` name string. Absent when no historical data is available.
