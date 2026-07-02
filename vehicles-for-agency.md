# Use Case: Get Vehicles for Agency

## Goal in Context

A rider's client application retrieves the current real-time status of every tracked vehicle operating under a specified agency, optionally filtered to only those with recent position updates.

## Scope

OneBusAway REST API — `GET /api/where/vehicles-for-agency/{id}.json`

## Level

User goal

## Primary Actor

Rider (via a client application)

## Stakeholders and Interests

- **Rider** — wants to see which vehicles are currently active, where they are, and which trips they are serving, in order to plan or track a journey.

## Preconditions

- The agency ID is a known, valid agency served by this server instance.
- The client supplies a valid API key.

## Minimal Guarantees

- The response is always a valid JSON envelope with a 200 HTTP status code (no server errors propagate to the client as 5xx).
- Out-of-service-area conditions are indicated by `outOfRange: true` rather than an error status.

## Success Guarantees

- The response lists all vehicles the system is currently tracking for the requested agency (subject to any age filter).
- Each entry includes the vehicle's last known position, operational phase, and — where a matching trip is found — the trip it is serving at the requested time and its real-time status.
- Trips, routes, agencies, stops, and service alerts referenced in the list are populated in the references block.

## Trigger

The client issues a GET request to `/api/where/vehicles-for-agency/{id}.json` with a valid agency ID and API key.

## Main Success Scenario

1. The client supplies the agency ID as the `{id}` path segment.
2. The server determines the effective reference time: if `time` is supplied it is used as-is; otherwise the server's current clock is used.
3. The server retrieves all vehicle location records currently held in the real-time store, retaining only those whose agency matches `{id}`.
4. If `ageInSeconds` is a positive integer, the server discards any vehicle whose last update time (per [`VehicleStatusBeanServiceImpl.java#L349–353`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/VehicleStatusBeanServiceImpl.java#L349)) is more than `ageInSeconds` seconds before the reference time. When `ageInSeconds` is absent, zero, or negative, no staleness filter is applied.
5. For each retained vehicle the server resolves the trip the vehicle is serving at the reference time and constructs a vehicle status entry (see Response Structure).
6. All trips, routes, agencies, stops, and service alerts referenced by any entry are collected into the references block.
7. The server returns HTTP 200 with `code: 200`, `limitExceeded: false`, `outOfRange: false`, and the list of vehicle status entries.

## Extensions

**4a. Agency is not in the service area of this server instance:**
- The server catches the out-of-service-area error and returns HTTP 200 with an empty list and `outOfRange: true`.

**3a. Agency ID is unknown but not out-of-service-area:**
- The agency filter finds no matching vehicles; the server returns HTTP 200 with an empty list and `outOfRange: false`. No error is raised.

**1a. The `id` path segment is missing entirely:**
- The server returns HTTP 404 (routing failure, not an API-level error response).

**5a. No matching trip is found for a vehicle at the reference time:**
- The vehicle entry is included, but `tripId` and `tripStatus` are absent.

**5b. A vehicle is executing a trip on a different route than the one it was nominally assigned to (interlining):**
- The outer `tripId` field refers to the nominal trip; the `tripStatus.activeTripId` field refers to the trip actually being executed at the reference time. Both trips appear in the references block. This is expected behaviour, not an error.

## Suspected Defects

### Defects that affect the use case

**Occupancy count and capacity use a sentinel integer instead of absent fields.**
[`BeanFactoryV2.java#L872–886`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L872)

When no occupancy count or capacity data is available, the response includes `occupancyCount: -1` and `occupancyCapacity: -1` rather than omitting those fields. The same -1 sentinel is used when `occupancyCapacity` is reported as zero or negative. Clients cannot rely on JSON field absence to detect unavailability; they must treat -1 as "no data". The intended behaviour was probably to omit these fields when no data is available.

**`ageInSeconds=0` silently disables the age filter.**
[`VehiclesForAgencyAction.java#L89`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/VehiclesForAgencyAction.java#L89)

The condition `_ageInSeconds != null && _ageInSeconds > 0` treats zero (and negative values) as equivalent to "no filter". A caller supplying `ageInSeconds=0` expecting to receive only vehicles updated at the exact reference time will instead receive all tracked vehicles regardless of age. The intended behaviour of `ageInSeconds=0` is unclear.

### Implementation defects only

None identified.

## Implementation Decisions

**`ageInSeconds=0` applies a strict zero-second cutoff, not "no filter".**

The legacy Java condition `_ageInSeconds != null && _ageInSeconds > 0` treated `ageInSeconds=0` as equivalent to absent — no staleness filter was applied. Maglev deliberately deviates from this: supplying `ageInSeconds=0` now applies a strict cutoff at the reference time, excluding any vehicle whose last update timestamp is before that instant. Only vehicles timestamped at or after the reference time are retained.

This was chosen because treating zero the same as "absent" makes the parameter non-monotonic — a caller cannot use `ageInSeconds=0` to express "give me only vehicles with a position update right now." The legacy behaviour reflects an implementation oversight (`> 0` instead of `>= 0`) rather than a deliberate API contract. Negative values and an absent parameter continue to disable the filter, preserving backward compatibility for the common case.

Closed by PR [#1108](https://github.com/OneBusAway/maglev/pull/1108).

**`outOfRange: true` (Extension 4a) is not implemented; `outOfRange` is always `false`.**

The spec's Extension 4a describes a federated multi-server scenario from the legacy Java stack, where a regional broker could receive a request for an agency it did not serve and return `outOfRange: true` to signal that the client should redirect to the correct regional server. Maglev has no equivalent concept: agency presence is binary — an agency is either loaded from the GTFS feed (and therefore served) or it is not in the database. There is no state in which Maglev knows about an agency but considers it "out of service area." Accordingly, `outOfRange` is hardcoded to `false` in all response paths. If Maglev ever gains multi-instance or federated routing support, this decision should be revisited.

Closed by PR [#1114](https://github.com/OneBusAway/maglev/pull/1114).

## Open Questions

None.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Agency ID — supplied as the {id} path segment. Plain agency ID; not in combined agencyId_entityId form."
    },
    "key": {
      "type": "string",
      "description": "API authentication key."
    },
    "time": {
      "type": "string",
      "description": "Reference time. Accepts Unix milliseconds as a plain integer string, or a formatted timestamp in yyyy-MM-dd_HH-mm-ss. Defaults to the server's current time."
    },
    "ageInSeconds": {
      "type": "integer",
      "description": "Maximum age of vehicle location data to include, in seconds. Only applied when the value is a positive integer. Non-positive values and absence are treated identically (no filter)."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "When false, the references block is omitted from the response."
    }
  }
}
```

**`id`** — The plain agency ID (e.g., `1` or `40`). Supplied as the URL path segment: `/api/where/vehicles-for-agency/{id}.json`. This is the bare agency identifier, not the combined `agencyId_entityId` form used for routes, stops, and trips.

**`key`** — API authentication key. Required for all requests.

**`time`** — Reference time used for two purposes: (1) determining which trip each vehicle is serving at that moment; and (2) computing the cutoff for the `ageInSeconds` filter. Accepts a plain integer (Unix milliseconds) or a string in `yyyy-MM-dd_HH-mm-ss` format. Defaults to the server's current time. Note that supplying a historical time retrieves current real-time vehicle records but resolves their trip associations at the historical time, producing a mixed-time-state response.

**`ageInSeconds`** — When a positive integer, vehicles whose last recorded update is older than this many seconds relative to the reference time are excluded. When absent, zero, or negative, all tracked vehicles are returned regardless of how stale their last update is.

**`includeReferences`** — Controls whether the `references` block is populated. Defaults to `true`. Set to `false` to reduce payload size when reference data is not needed.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "code":        { "type": "integer", "description": "HTTP status code; 200 on success." },
    "text":        { "type": "string" },
    "version":     { "type": "integer", "description": "Always 2." },
    "currentTime": { "type": "integer", "description": "Unix ms — server time at the moment the response was generated." },
    "data":        { "type": "object" }
  }
}
```

**`code`** — `200` in all cases where the API processes the request. Error conditions at the API level (e.g., out-of-service-area) are still returned with `code: 200`.

**`currentTime`** — Server wall-clock time in Unix milliseconds at the moment the response was produced.

**`data`** — Contains the list, pagination flags, and references.

---

### data

```json
{
  "type": "object",
  "properties": {
    "limitExceeded": { "type": "boolean", "description": "Always false; this endpoint does not cap result count." },
    "outOfRange":    { "type": "boolean", "description": "True when the agency is not within the service area of this server instance." },
    "list":          { "type": "array" },
    "references":    { "type": "object" }
  }
}
```

**`data.limitExceeded`** — Always `false`. The endpoint returns all matching vehicles without applying a count ceiling.

**`data.outOfRange`** — `true` only when the requested agency falls outside the server's service area. In that case `list` is empty. For unknown agency IDs that are simply not tracked, this is `false` and the list is empty.

**`data.list`** — Array of vehicle status entries; see below.

**`data.references`** — Populated when `includeReferences` is `true` (the default). Contains the trips, routes, agencies, stops, and situations referenced anywhere in `data.list`.

---

### data.list[]

Each element describes one tracked vehicle.

```json
{
  "type": "object",
  "properties": {
    "vehicleId":              { "type": "string" },
    "lastUpdateTime":         { "type": "integer", "description": "Unix ms — time the vehicle's record was last updated by any source." },
    "lastLocationUpdateTime": { "type": "integer", "description": "Unix ms — time the vehicle's GPS position was last updated. Absent when no location update has been received." },
    "location": {
      "type": "object",
      "properties": {
        "lat": { "type": "number" },
        "lon": { "type": "number" }
      }
    },
    "phase":           { "type": "string" },
    "status":          { "type": "string" },
    "occupancyStatus": { "type": "string" },
    "occupancyCount":  { "type": "integer" },
    "occupancyCapacity": { "type": "integer" },
    "tripId":          { "type": "string" },
    "tripStatus":      { "type": "object" }
  }
}
```

**`data.list[].vehicleId`** — Opaque vehicle identifier in combined `agencyId_entityId` form (e.g., `1_4321`). Links the physical vehicle to its real-time records.

**`data.list[].lastUpdateTime`** — Unix milliseconds. The timestamp of the most recent update from any real-time source for this vehicle. This is the value compared against `ageInSeconds`.

**`data.list[].lastLocationUpdateTime`** — Unix milliseconds. The timestamp of the most recent GPS position update specifically. Absent when no GPS update has ever been received for this vehicle.

**`data.list[].location`** — The vehicle's last known GPS coordinates (`lat`, `lon`). May be absent if no position has been reported.

**`data.list[].phase`** — The vehicle's operational phase as a lower-case string. Possible values: `at_base`, `deadhead_before`, `layover_before`, `in_progress`, `deadhead_during`, `layover_during`, `deadhead_after`, `layover_after`, `unknown`. `in_progress` means the vehicle is actively serving a passenger trip; the deadhead and layover phases describe non-revenue or idle portions of the block.

**`data.list[].status`** — An implementation-defined string describing the vehicle's scheduling status. Typically used for short text codes such as `default` or `deviated`. May be absent.

**`data.list[].occupancyStatus`** — A string from the GTFS-RT occupancy vocabulary indicating how full the vehicle is. Possible values: `EMPTY`, `MANY_SEATS_AVAILABLE`, `FEW_SEATS_AVAILABLE`, `STANDING_ROOM_ONLY`, `CRUSHED_STANDING_ROOM_ONLY`, `FULL`, `NOT_ACCEPTING_PASSENGERS`. Absent when no occupancy data is available or when the value is unknown. See Suspected Defects regarding the inconsistent treatment of this field between the outer entry and the nested `tripStatus`.

**`data.list[].occupancyCount`** — Raw passenger count reported by the vehicle. `-1` when not available. See Suspected Defects.

**`data.list[].occupancyCapacity`** — Total passenger capacity of the vehicle. `-1` when not available or when the reported capacity is zero or negative. See Suspected Defects.

**`data.list[].tripId`** — Combined `agencyId_tripId` identifier of the trip the vehicle is nominally assigned to. Absent when no trip association is found for the reference time. For interlining vehicles, this may differ from `tripStatus.activeTripId`.

**`data.list[].tripStatus`** — Nested real-time status of the trip being served. Absent when no trip association is found. See the `data.list[].tripStatus` schema below.

---

### data.list[].tripStatus

```json
{
  "type": "object",
  "properties": {
    "activeTripId":                { "type": "string" },
    "blockTripSequence":           { "type": "integer" },
    "serviceDate":                 { "type": "integer", "description": "Unix ms — midnight of the service date in the agency's local timezone." },
    "frequency":                   { "type": "object", "description": "Present only for frequency-based (headway) trips." },
    "scheduledDistanceAlongTrip":  { "type": "number", "description": "Metres along the trip the vehicle is scheduled to be at the reference time." },
    "totalDistanceAlongTrip":      { "type": "number", "description": "Total length of the trip in metres." },
    "position": {
      "type": "object",
      "properties": {
        "lat": { "type": "number" },
        "lon": { "type": "number" }
      }
    },
    "orientation":              { "type": "number" },
    "closestStop":              { "type": "string" },
    "closestStopTimeOffset":    { "type": "integer" },
    "nextStop":                 { "type": "string" },
    "nextStopTimeOffset":       { "type": "integer" },
    "phase":                    { "type": "string" },
    "status":                   { "type": "string" },
    "predicted":                { "type": "boolean" },
    "lastUpdateTime":           { "type": "integer", "description": "Unix ms. Absent when zero." },
    "lastLocationUpdateTime":   { "type": "integer", "description": "Unix ms. Absent when zero." },
    "lastKnownDistanceAlongTrip": { "type": "number", "description": "Metres. Absent when no real-time distance is known." },
    "lastKnownLocation": {
      "type": "object",
      "properties": {
        "lat": { "type": "number" },
        "lon": { "type": "number" }
      }
    },
    "lastKnownOrientation": { "type": "number" },
    "scheduleDeviation":    { "type": "integer", "description": "Seconds; positive = late, negative = early. Absent when not determinable." },
    "distanceAlongTrip":    { "type": "number", "description": "Metres along the current trip. Absent when not set." },
    "vehicleId":            { "type": "string" },
    "occupancyStatus":      { "type": "string" },
    "occupancyCount":       { "type": "integer" },
    "occupancyCapacity":    { "type": "integer" },
    "vehicleFeatures":      { "type": "array", "items": { "type": "string" } },
    "situationIds":         { "type": "array", "items": { "type": "string" } }
  }
}
```

**`data.list[].tripStatus.activeTripId`** — Combined `agencyId_tripId` of the trip the vehicle is actively executing at the reference time. For interlining blocks this may differ from the outer `tripId`. The corresponding trip record appears in `references.trips`.

**`data.list[].tripStatus.blockTripSequence`** — Zero-based position of the active trip within the vehicle's block for this service day. `-1` if unavailable.

**`data.list[].tripStatus.serviceDate`** — Unix milliseconds representing midnight of the service date (in the agency's local timezone). Times within the trip are expressed as seconds elapsed since this value and may exceed 86,400 for trips that run past midnight.

**`data.list[].tripStatus.frequency`** — For headway-based services, describes the service frequency interval. Absent for fixed-schedule trips.

**`data.list[].tripStatus.scheduledDistanceAlongTrip`** — Metres from the start of the trip to where the vehicle is expected to be at the reference time, based on the static schedule.

**`data.list[].tripStatus.totalDistanceAlongTrip`** — Total length of the trip's path in metres.

**`data.list[].tripStatus.position`** — Vehicle's position (lat/lon), derived from real-time data when available or estimated from the schedule when not.

**`data.list[].tripStatus.orientation`** — Vehicle heading in degrees using a non-standard convention: 0° = East, 90° = North, 180° = West, 270° = South. Absent when orientation is not determinable.

**`data.list[].tripStatus.closestStop`** — Combined stop ID of the stop nearest to the vehicle's current position along the trip. The stop record appears in `references.stops`.

**`data.list[].tripStatus.closestStopTimeOffset`** — Seconds between the vehicle's position and the closest stop according to the schedule. Positive means the stop is ahead; negative means it has been passed.

**`data.list[].tripStatus.nextStop`** — Combined stop ID of the next stop the vehicle will serve. The stop record appears in `references.stops`.

**`data.list[].tripStatus.nextStopTimeOffset`** — Seconds until the vehicle reaches the next stop, based on the schedule.

**`data.list[].tripStatus.phase`** — Operational phase string (same vocabulary as the outer `phase` field).

**`data.list[].tripStatus.status`** — Scheduling status string.

**`data.list[].tripStatus.predicted`** — `true` when real-time GPS or prediction data was used to derive this status; `false` when the position and timing are estimated from the static schedule only.

**`data.list[].tripStatus.lastUpdateTime`** — Unix milliseconds of the most recent real-time update. Absent when zero.

**`data.list[].tripStatus.lastLocationUpdateTime`** — Unix milliseconds of the most recent GPS position update. Absent when zero.

**`data.list[].tripStatus.lastKnownDistanceAlongTrip`** — Metres along the trip at the time of the most recent known real-time position. Absent when no real-time distance has been recorded.

**`data.list[].tripStatus.lastKnownLocation`** — Coordinates of the vehicle's most recently confirmed real-time position. May be absent.

**`data.list[].tripStatus.lastKnownOrientation`** — Heading in degrees (same convention as `orientation`) at the time of the last known real-time fix. Absent when not available.

**`data.list[].tripStatus.scheduleDeviation`** — How many seconds early (negative) or late (positive) the vehicle is running relative to its schedule. Absent when not determinable.

**`data.list[].tripStatus.distanceAlongTrip`** — Metres the vehicle has travelled along the current trip. Absent when not set.

**`data.list[].tripStatus.vehicleId`** — Combined vehicle ID, same value as the outer `vehicleId` field.

**`data.list[].tripStatus.occupancyStatus`** — Occupancy category string (GTFS-RT vocabulary). Unlike the outer entry, this field is not filtered for unknown values — it may contain `UNKNOWN` if that is what the real-time data source reports.

**`data.list[].tripStatus.occupancyCount`** — Raw passenger count at the trip level. `-1` when not available.

**`data.list[].tripStatus.occupancyCapacity`** — Vehicle capacity at the trip level. `-1` when not available.

**`data.list[].tripStatus.vehicleFeatures`** — List of feature strings reported for this vehicle (e.g., accessibility or equipment descriptors). Empty array or absent when none are reported.

**`data.list[].tripStatus.situationIds`** — IDs of active service alerts affecting this trip. Each ID corresponds to a situation in `references.situations`. Absent when there are no active alerts.
