# trip-details

**GET** `/api/where/trip-details/{id}.json`

---

## Goal in Context

A rider's client application needs to display the full picture for a specific scheduled trip: the complete sequence of stops and times, any current real-time vehicle position and schedule deviation, and any active service disruption notices. The client supplies a trip ID and receives a single, self-contained response that covers all three concerns.

## Scope

OneBusAway public REST API — trip-details endpoint.

## Level

User goal.

## Primary Actor

Rider (accessing the system via a client application).

## Stakeholders and Interests

**Rider** — wants accurate scheduled stop times for a trip so they can plan their journey; wants to know whether the vehicle is on time and where it currently is; wants to be informed of any service disruptions affecting this trip.

## Preconditions

- The caller holds a valid API key.
- The trip ID being requested exists in the loaded transit data.

## Minimal Guarantees

- The response envelope always contains a `code` field and a `currentTime` timestamp in Unix milliseconds.
- An error response never exposes internal stack traces; it returns a structured envelope with an appropriate HTTP status code.

## Success Guarantees

- The response contains `data.entry.tripId` matching the requested trip ID in combined `{agencyId}_{entityId}` form.
- The response contains `data.entry.serviceDate` identifying the service date for which the trip details were resolved, as Unix milliseconds.
- If `includeSchedule` is true, `data.entry.schedule` contains the complete ordered sequence of stop times for the trip.
- If `includeStatus` is true and the system has a current tracking record for the trip's block, `data.entry.status` contains the vehicle's real-time or schedule-derived position.
- `data.references` contains full objects for every ID referenced in the entry (agencies, routes, trips, stops, situations) unless the caller suppressed references via `includeReferences=false`.

## Trigger

Client sends a GET request to `/api/where/trip-details/{id}.json` with a valid API key and a trip ID path parameter.

---

## Main Success Scenario

1. The caller provides a trip ID in the URL path. The system parses it as a combined `{agencyId}_{entityId}` identifier using the first underscore as the separator.

2. The system looks up the block that contains this trip. If `serviceDate` was supplied, only the block instance(s) operating on that service date are considered. If `vehicleId` was supplied, only block instances currently assigned to that vehicle are considered.

3. The system selects the block instance. If `serviceDate` was omitted, all currently active service dates for the block are eligible; when multiple instances match, the first one in iteration order is used ([`TripStatusBeanServiceImpl.java:150–153`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L150-L153)).

4. The system constructs the response entry:

   a. Sets `tripId` to the combined trip ID string and `serviceDate` to the Unix millisecond timestamp of midnight on the operating service date.

   b. If the trip is frequency-based, populates `frequency` with window start/end times (as Unix milliseconds) and the headway in seconds.

   c. If `includeTrip` is true, adds the full trip record for the *requested* trip (route ID, headsign, direction, block ID, shape ID, and service ID) to the references block. [Source: [`TripStatusBeanServiceImpl.java#L464–L468`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L464-L468)] This flag has no bearing on the *active* trip added in step e below: `getBlockLocationAsStatusBean` sets it on the status bean unconditionally, with no dependency on this flag, whenever a `status` block is built. [Source: [`TripStatusBeanServiceImpl.java#L294–L295`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L294-L295)]

   d. If `includeSchedule` is true, populates `schedule` with the trip's timezone, the complete ordered list of stop times (each with arrival and departure times expressed as seconds since midnight of the service date, the combined stop ID, the distance from the start of the trip in metres, and, when available, historical occupancy and a stop-level headsign override), and the combined IDs of the preceding and following trips in the block (if any). All stops referenced in the schedule are added to the references block.

   e. If `includeStatus` is true and the system has a current tracking record for the block, populates `status` with:
      - `activeTripId`: the trip the vehicle is actively serving at the query time (may differ from the requested trip if the vehicle has begun a subsequent interlining trip on the same block)
      - `blockTripSequence`: the zero-based index of the active trip within the block
      - `serviceDate`: same value as the entry-level service date, in Unix milliseconds
      - `position`: current estimated lat/lon of the vehicle
      - `orientation`: current heading in degrees (0° = east, 90° = north, 180° = west, 270° = south)
      - `closestStop` / `closestStopTimeOffset`: combined stop ID of the stop nearest the vehicle's current position, and the time offset in seconds from that stop's scheduled time (positive = stop is ahead, negative = already passed)
      - `nextStop` / `nextStopTimeOffset`: combined stop ID and offset for the very next stop the vehicle will serve
      - `scheduledDistanceAlongTrip`: how far the vehicle should have travelled along the active trip at the query time, in metres
      - `totalDistanceAlongTrip`: the full length of the active trip in metres
      - `distanceAlongTrip`: how far the vehicle has actually travelled along the active trip, in metres (present when real-time data provides this)
      - `scheduleDeviation`: the vehicle's lateness in seconds (positive = late, negative = early); present when real-time data is available, zero otherwise
      - `predicted`: `true` if the position and deviation were derived from real-time GPS data; `false` if they are computed from the static schedule
      - `lastUpdateTime` / `lastLocationUpdateTime`: Unix millisecond timestamps of the most recent real-time update and the most recent real-time location update respectively; absent if no updates have been received (Maglev emits `0` — see [Implementation Decisions](#implementation-decisions))
      - `lastKnownLocation`: the last GPS-confirmed lat/lon (as opposed to the extrapolated `position`)
      - `lastKnownDistanceAlongTrip` / `lastKnownOrientation`: the last known real-time distance and heading values
      - `phase`: the vehicle's current operating phase (e.g., `in_progress`, `layover_during`, `deadhead_before`)
      - `status`: a status modifier string, `"default"` under normal operation, `"CANCELED"` if the trip has been cancelled
      - `vehicleId`: combined vehicle ID, present when a vehicle is assigned
      - `occupancyStatus`: occupancy category string (e.g., `MANY_SEATS_AVAILABLE`), present when APC data is available
      - `occupancyCount` / `occupancyCapacity`: raw passenger count and vehicle capacity; `-1` when unavailable
      - `vehicleFeatures`: list of feature strings for the vehicle
      - `situationIds`: combined IDs of any service alerts that apply to the vehicle's current journey

      The active trip, closest stop, and next stop named by the status block are added to the references block independently of whether the schedule block is present. [Source: [`BeanFactoryV2.java#L533–L564`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L533-L564)]

   f. Collects all service alerts applicable to the trip at the query time. Any that are not restricted to a specific application (or whose restriction the system resolves as matching) have their IDs added to the entry-level `situationIds` and their full records added to `references.situations`. See [[Situation-References]] for the shared dereferencing and ID-formation rules.

5. The system returns HTTP 200 with the response envelope containing `data.entry` and `data.references`.

## Extensions

**1a. `id` is absent from the request.**
The system returns HTTP 400 with a validation error envelope.

**2a. The trip ID is not recognised in the transit graph.**
The system returns HTTP 404 (`resource not found`).

**2b. `serviceDate` is provided but no block instance exists for that service date.**
The system returns HTTP 404 (`resource not found`).

**2c. `vehicleId` is provided but no block location exists for that vehicle.**
The system returns HTTP 404 (`resource not found`).

**3a. The agency that owns this trip has schedule suppression enabled in the server configuration, and no vehicle is currently assigned to the trip.**
The system treats the trip as unavailable and returns HTTP 404. This behaviour is a per-agency server-side configuration; callers cannot influence it from the request.

**4e. `includeStatus` is false, or the block has no current tracking record (the trip has not yet started, has already completed, or no location data has ever been received).**
The `status` key is absent from the entry entirely.

**4d. `includeSchedule` is false.**
The `schedule` key is absent from the entry entirely. Schedule-derived references are not added from that path: the schedule's stop-times, previous trip, and next trip are omitted along with the schedule block. Status-derived references still apply when `includeStatus` is true and a status block is present, so `status.activeTripId`, `status.closestStop`, and `status.nextStop` may still have corresponding entries in `references.trips` and `references.stops` even when `includeSchedule=false`; see step 4e.

**4c. `includeTrip` is false.**
The requested trip's own record is not added to references on that account; only the preceding and following block trips (if any, and only when `includeSchedule` is true) appear in `references.trips` from that path. This does not affect the active trip: if `includeStatus` is true and a tracking record exists, `status.activeTripId`'s full trip record is still added to `references.trips`, independent of `includeTrip` — see steps 4.c and 4.e in the Main Success Scenario. [Source: [`TripStatusBeanServiceImpl.java#L294–L295`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L294-L295)]

**4f. No service alerts apply to this trip.**
`situationIds` is an empty array `[]`.

**API key missing or invalid.**
The system returns HTTP 401 (`permission denied`) before the action logic runs.

---

## Suspected Defects

### Defects that affect the use case

**1. `TripDetailsAction` — NullPointerException when schedule suppression is active and `includeStatus=false`**

At [`TripDetailsAction.java:124–129`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripDetailsAction.java#L124-L129), the schedule-suppression check calls `trip.getStatus().getVehicleId()`. When the caller passes `includeStatus=false`, `trip.getStatus()` returns `null` (the status section was never populated), causing a NullPointerException and an HTTP 500 response. The intended behaviour is to return HTTP 404 when no vehicle is assigned. This only manifests on agencies with the schedule-suppression flag active.

**2. `BeanFactoryV2` — application-scoped service alert filtering always bypassed**

At [`BeanFactoryV2.java:1381`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1381), the filter reads `return !_applicationKey.contains(_applicationKey)`. Because a string always contains itself, this expression is always `false` — meaning no service alert is ever excluded on the basis of application key. The intended code is `return !applicationIds.contains(_applicationKey)`. As a result, service alerts targeted at specific client applications are shown to all callers regardless of their API key, and their full details always appear in `references.situations`.

### Implementation defects only

**3. `TripStatusBeanServiceImpl` — `lastKnownLocation` assigned twice**

At [`TripStatusBeanServiceImpl.java:270–276`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L270-L276), `bean.setLastKnownLocation(blockLocation.getLastKnownLocation())` is called on line 270 and again on line 276, each time with the same value. The second call is a no-op. A clean reimplementation should set the field once.

**4. `TripStopTimesBeanServiceImpl` — historical occupancy computed and written twice per stop**

At [`TripStopTimesBeanServiceImpl.java:109–113`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStopTimesBeanServiceImpl.java#L109-L113), `getStopTimesForBlockTrip` iterates over the stop times returned by `getStopTimesForTrip` and sets historical occupancy again. `getStopTimesForTrip` already computed and set the same value at lines 155–156. Both lookups use identical parameters, so the second write overwrites the first with the same result. The work is duplicated unnecessarily.

---

## Implementation Decisions

**`lastUpdateTime` and `lastLocationUpdateTime` emit `0` rather than being absent when no real-time update has been received.**

The legacy Java implementation omitted `status.lastUpdateTime` and `status.lastLocationUpdateTime` from the response when no real-time data had been received for the vehicle. Maglev emits `0` for both fields in that case.

The fields are carried on the shared `TripStatus` struct, which is reused across multiple endpoints. Adding per-endpoint omission logic to a shared struct would require either endpoint-specific wrapper types or a custom marshaler — complexity that is not warranted given that `0` is an unambiguous sentinel and no known client treats absence and `0` differently for these fields.

---

**Situation IDs use idempotent agency-prefixing, not Java's unconditional prefixing.**

In Java, `GtfsRealtimeSource.handleAlerts` prefixes every alert ID unconditionally by calling `createId(entity.getId())`, which always returns `new AgencyAndId(_agencyIds.get(0), id)` without inspecting the raw ID first. [Source: [`GtfsRealtimeSource.java#L1064-L1065`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/realtime/gtfs_realtime/GtfsRealtimeSource.java#L1064-L1065), [`#L1176-L1177`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/realtime/gtfs_realtime/GtfsRealtimeSource.java#L1176-L1177)]

Maglev's `situationID()` (`internal/restapi/reference_utils.go`) instead skips prefixing when the raw alert ID already starts with `"{agencyID}_"`. This is deliberate (PR #1313): some GTFS-RT feeds — Puget Sound among them — already export alert IDs in prefixed form (`1_92239`); prefixing unconditionally would double-prefix them to `1_1_92239`, which resolves against nothing any client has seen from upstream OBA for that alert.

Known edge case: a raw alert ID that happens to be shaped like `{agencyID}_{x}` for reasons unrelated to Maglev's prefixing is left unprefixed when it should have been prefixed.

If this is reconsidered in the future, the fallback is Java's unconditional prefixing — accepting the double-prefix regression on feeds that already export prefixed IDs.

---

**Situation ID's agency prefix is chosen from the alert's own informed entity, not the feed's first configured agency.**

In Java, `createId` always prefixes an alert ID using the feed's first configured agency — `_agencyIds.get(0)` — regardless of which agency the alert's own `informed_entity` list names. [Source: [`GtfsRealtimeSource.java#L1176-L1177`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/realtime/gtfs_realtime/GtfsRealtimeSource.java#L1176-L1177)]

Maglev's `agencyIDForAlert()` (`internal/restapi/reference_utils.go`) instead prefers the alert's own `InformedEntities[].AgencyID` over whichever agency the caller's lookup path resolved (PR #1313). This avoids a problem Java's approach doesn't guard against: the same alert can be reached two different ways within a single response — e.g. via a route belonging to agency A and a stop belonging to agency B — and scoping the ID to whichever lookup path found it first would publish that alert under two different `situationId` values in the same response.

This is invisible on any GTFS-RT feed configured with a single `agency-ids` entry, since the alert's own informed-entity agency and the feed's first configured agency are then the same value — the only configuration verified live for PR #1313. On a `gtfs-rt-feeds` entry configured with multiple `agency-ids`, an alert whose `informed_entity.agency_id` differs from that feed's first configured agency gets a different ID prefix than Java produces for the same alert.

If this is reconsidered in the future, the fallback is Java's behaviour — always prefix using the feed's first configured agency — which reintroduces the two-IDs-for-one-alert problem on cross-agency lookups that this deviation exists to avoid.

**Malformed `includeTrip`, `includeSchedule` or `includeStatus` returns HTTP 400, checked before the trip lookup (deviates from legacy).**

Legacy binds these parameters to primitive `boolean` setters ([`TripDetailsAction.java#L88–L98`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripDetailsAction.java#L88)). Struts converts them with `Boolean.valueOf`, which never fails: `true` in any letter case means true, and every other value (`abc`, `1`, `yes`) means false. So legacy never records a field error for these parameters. Legacy does check field errors before looking up the trip ([`TripDetailsAction.java#L105–L106`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripDetailsAction.java#L105)), but only a malformed `serviceDate` or `time` can fail that check. For those two, Maglev and legacy agree.

Maglev parses the three booleans with Go's `strconv.ParseBool` (`utils.ParseBoolParam`, `internal/utils/api.go`). It accepts `1`, `t`, `T`, `TRUE`, `true`, `True`, `0`, `f`, `F`, `FALSE`, `false` and `False`, and rejects anything else with HTTP 400 and the field error `must be a boolean value (true/false)`. Since [PR #1489](https://github.com/OneBusAway/maglev/pull/1489), every query parameter is checked before the trip lookup, so a malformed value returns 400 even for an unknown trip ID, where legacy returns 404.

This follows Maglev's policy of rejecting malformed input rather than reproducing legacy's lenient parsing: silently reading a typo such as `includeStatus=ture` as `false` hides the caller's mistake. One consequence goes beyond rejection: Maglev reads `1`, `t` and the other non-`true` spellings Go accepts as true, where legacy reads them as false. That difference, and applying the policy consistently across endpoints, is tracked in [#1499](https://github.com/OneBusAway/maglev/issues/1499).

---

## Open Questions

None. All observable behaviours were resolved through static analysis and live server testing.

---

## Request Parameters

```json
{
  "type": "object",
  "properties": {
    "id": {
      "type": "string"
    },
    "serviceDate": {
      "type": "string"
    },
    "time": {
      "type": "string"
    },
    "vehicleId": {
      "type": "string"
    },
    "includeTrip": {
      "type": "boolean",
      "default": true
    },
    "includeSchedule": {
      "type": "boolean",
      "default": true
    },
    "includeStatus": {
      "type": "boolean",
      "default": true
    },
    "includeReferences": {
      "type": "boolean",
      "default": true
    },
    "key": {
      "type": "string"
    }
  },
  "required": ["id", "key"]
}
```

**`id`** — The trip to retrieve, as a combined `{agencyId}_{entityId}` string. Encoded directly in the URL path: `/api/where/trip-details/{id}.json`. The agency ID is everything before the first underscore; the entity ID is everything after it (and may itself contain underscores).

**`serviceDate`** — Restricts the lookup to a specific operating service date. Accepts either a Unix millisecond timestamp (an integer string, e.g., `1778310000000`) or a calendar date in `yyyy-MM-dd` form. When omitted, all service dates on which the trip is currently active are considered, and the first matching instance is returned. Providing this parameter is necessary to disambiguate trips that run on multiple service dates simultaneously (e.g., late-night trips straddling midnight).

**`time`** — The reference time used to compute the real-time status section and to evaluate which service alerts are currently active. Accepts a Unix millisecond timestamp or a datetime string in `yyyy-MM-dd_HH-mm-ss` format. Defaults to the server's current time. Has no effect on whether the trip is found; it only influences the content of `status` and `situationIds`.

**`vehicleId`** — Filters the result to a specific vehicle, in combined `{agencyId}_{entityId}` form. When omitted, any vehicle serving the trip's block is considered.

**`includeTrip`** — When `true` (the default), the full trip record is included in `references.trips`. Set to `false` to suppress it and reduce response size when only the schedule or status is needed.

**`includeSchedule`** — When `true` (the default), the `schedule` block is included in the entry. Set to `false` to omit all stop time data.

**`includeStatus`** — When `true` (the default), the `status` block is included in the entry when a current tracking record exists. Set to `false` to omit real-time position data.

**`includeReferences`** — When `true` (the default), the response includes a fully populated `references` object. When `false`, the object is present but all its collections are empty, reducing response size for callers that already have the referenced entities cached.

**`key`** — API authentication key. Required on every request.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "code": { "type": "integer" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "text": { "type": "string" },
    "version": { "type": "integer" },
    "data": {
      "type": "object",
      "properties": {
        "entry": { "type": "object" },
        "references": { "type": "object" }
      }
    }
  }
}
```

**`code`** — HTTP status code mirrored in the body (`200`, `400`, `401`, `404`).

**`currentTime`** — Server wall-clock time at the moment the response was generated, as Unix milliseconds.

**`text`** — Human-readable status message (e.g., `"OK"`, `"resource not found"`).

**`version`** — API version number. Always `2` for responses to requests handled by this endpoint.

**`data.entry`** — The trip details object; see below.

**`data.references`** — Shared entity objects (agencies, routes, trips, stops, situations) keyed by the IDs that appear in the entry. Situation references are included only for explicit `situationIds` values; see [[Situation-References]]. Absent when `includeReferences=false`.

---

### data.entry

```json
{
  "type": "object",
  "properties": {
    "tripId": { "type": "string" },
    "serviceDate": { "type": "integer", "description": "Unix ms" },
    "frequency": { "type": "object" },
    "schedule": { "type": "object" },
    "status": { "type": "object" },
    "situationIds": {
      "type": "array",
      "items": { "type": "string" }
    }
  }
}
```

**`data.entry.tripId`** — The combined `{agencyId}_{entityId}` identifier of the requested trip.

**`data.entry.serviceDate`** — The Unix millisecond timestamp of midnight on the service date for which this trip instance was resolved. Use this value when constructing subsequent requests that require a service date for disambiguation.

**`data.entry.frequency`** — Present only for frequency-based (headway-operated) trips. See the `frequency` schema below.

**`data.entry.schedule`** — The trip's full stop-time sequence. Absent when `includeSchedule=false`.

**`data.entry.status`** — The vehicle's current real-time or schedule-derived status. Absent when `includeStatus=false` or when no tracking record exists for the block.

**`data.entry.situationIds`** — Combined IDs of service alerts currently applicable to this trip. Full alert records are in `references.situations`. May be an empty array. See [[Situation-References]] for the shared dereferencing and ID-formation rules. The ID prefixing used here deviates from Java in two ways — see [Implementation Decisions](#implementation-decisions).

---

### data.entry.schedule

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

**`data.entry.schedule.timeZone`** — The IANA timezone identifier for the trip's agency (e.g., `"America/Los_Angeles"`). Use this to convert the stop-time offsets into wall-clock times.

**`data.entry.schedule.stopTimes`** — Ordered list of stop visits; see `stopTimes[]` below.

**`data.entry.schedule.previousTripId`** — Combined trip ID of the trip that immediately precedes this one in the block — i.e., the trip the vehicle was operating before pulling into the first stop of this trip. Absent if this trip is the first in its block. The referenced trip record is included in `references.trips`.

**`data.entry.schedule.nextTripId`** — Combined trip ID of the trip that immediately follows this one in the block. Absent if this trip is the last in its block. The referenced trip record is included in `references.trips`.

**`data.entry.schedule.frequency`** — Present for frequency-based trips where the schedule section inherits a frequency window. See the `frequency` schema below.

---

### data.entry.schedule.stopTimes[]

```json
{
  "type": "object",
  "properties": {
    "stopId": { "type": "string" },
    "arrivalTime": { "type": "integer" },
    "departureTime": { "type": "integer" },
    "distanceAlongTrip": { "type": "number" },
    "stopHeadsign": { "type": "string" },
    "historicalOccupancy": { "type": "string" }
  }
}
```

**`data.entry.schedule.stopTimes[].stopId`** — Combined `{agencyId}_{entityId}` identifier for the stop. The full stop record is in `references.stops`.

**`data.entry.schedule.stopTimes[].arrivalTime`** — Scheduled arrival time at this stop, expressed as seconds elapsed since midnight of the service date (i.e., `data.entry.serviceDate / 1000`). Values greater than 86 400 are valid for trips that run past midnight.

**`data.entry.schedule.stopTimes[].departureTime`** — Scheduled departure time, same semantics as `arrivalTime`.

**`data.entry.schedule.stopTimes[].distanceAlongTrip`** — Cumulative distance in metres from the first stop of the trip to this stop, as derived from the shape geometry.

**`data.entry.schedule.stopTimes[].stopHeadsign`** — A stop-level destination override for the vehicle's front sign at this specific stop, when the operator has defined one. Empty string if no override is set.

**`data.entry.schedule.stopTimes[].historicalOccupancy`** — The typical occupancy level at this stop on this trip based on historical ridership data, expressed as an occupancy category string (e.g., `"MANY_SEATS_AVAILABLE"`, `"FULL"`). Empty string when no historical data is available.

---

### data.entry.status

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
    "distanceAlongTrip": { "type": "number" },
    "position": {
      "type": "object",
      "properties": {
        "lat": { "type": "number" },
        "lon": { "type": "number" }
      }
    },
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
    "lastKnownLocation": {
      "type": "object",
      "properties": {
        "lat": { "type": "number" },
        "lon": { "type": "number" }
      }
    },
    "lastKnownDistanceAlongTrip": { "type": "number" },
    "lastKnownOrientation": { "type": "number" },
    "scheduleDeviation": { "type": "integer" },
    "vehicleId": { "type": "string" },
    "occupancyStatus": { "type": "string" },
    "occupancyCount": { "type": "integer" },
    "occupancyCapacity": { "type": "integer" },
    "vehicleFeatures": {
      "type": "array",
      "items": { "type": "string" }
    },
    "situationIds": {
      "type": "array",
      "items": { "type": "string" }
    }
  }
}
```

**`data.entry.status.activeTripId`** — Combined trip ID of the trip the vehicle is currently executing. This matches the queried trip ID in normal operation, but will differ when the vehicle has advanced to the next trip in its block (interlining). The trip record is in `references.trips`.

**`data.entry.status.blockTripSequence`** — Zero-based position of the active trip within the block's ordered trip list.

**`data.entry.status.serviceDate`** — Unix milliseconds of midnight on the operating service date, matching `data.entry.serviceDate`.

**`data.entry.status.frequency`** — Frequency window if the active trip is headway-operated. See the `frequency` schema below.

**`data.entry.status.scheduledDistanceAlongTrip`** — How far the vehicle should have travelled along the active trip at the query time, in metres, based on the static schedule.

**`data.entry.status.totalDistanceAlongTrip`** — The total length of the active trip from first stop to last stop, in metres.

**`data.entry.status.distanceAlongTrip`** — How far the vehicle has actually travelled along the active trip, in metres. Present when the underlying real-time feed supplies it; may be extrapolated from the last known position.

**`data.entry.status.position`** — Current estimated lat/lon of the vehicle. Derived from real-time GPS when `predicted` is `true`; otherwise extrapolated from the schedule.

**`data.entry.status.orientation`** — Vehicle heading in degrees, where 0° is east, 90° is north, 180° is west, and 270° is south.

**`data.entry.status.closestStop`** — Combined stop ID of the stop nearest the vehicle's current position among the active trip's stops. The stop record is in `references.stops`.

**`data.entry.status.closestStopTimeOffset`** — Seconds between the vehicle's current position and the scheduled time at `closestStop`. Positive means the stop has not yet been reached; negative means it has already been passed.

**`data.entry.status.nextStop`** — Combined stop ID of the next stop the vehicle will serve. Absent after the vehicle has passed the last stop. The stop record is in `references.stops`.

**`data.entry.status.nextStopTimeOffset`** — Seconds until the vehicle reaches `nextStop` based on the schedule.

**`data.entry.status.phase`** — The vehicle's current phase in its block execution. Values include `in_progress` (serving a trip), `layover_before` / `layover_during` / `layover_after` (waiting at a terminal), `deadhead_before` / `deadhead_during` / `deadhead_after` (non-revenue travel), and `at_base` (at a depot).

**`data.entry.status.status`** — A status modifier. `"default"` under normal operation; `"CANCELED"` when the trip has been cancelled in real time.

**`data.entry.status.predicted`** — `true` if the position and schedule deviation are derived from real-time GPS data; `false` if they are computed from the static timetable.

**`data.entry.status.lastUpdateTime`** — Unix millisecond timestamp of the most recent real-time message received from the vehicle (position, status, or any other update). Absent if no updates have ever been received (Maglev emits `0` — see [Implementation Decisions](#implementation-decisions)).

**`data.entry.status.lastLocationUpdateTime`** — Unix millisecond timestamp of the most recent real-time message that contained a location fix. Absent if no location updates have been received (Maglev emits `0` — see [Implementation Decisions](#implementation-decisions)).

**`data.entry.status.lastKnownLocation`** — The last GPS-confirmed position of the vehicle, without extrapolation. May lag behind `position` when the system has extrapolated forward.

**`data.entry.status.lastKnownDistanceAlongTrip`** — The distance along the active trip reported in the most recent real-time location update, in metres.

**`data.entry.status.lastKnownOrientation`** — The heading reported in the most recent real-time location update, in degrees.

**`data.entry.status.scheduleDeviation`** — How many seconds behind (positive) or ahead (negative) of schedule the vehicle is at the query time. Zero when `predicted` is `false`.

**`data.entry.status.vehicleId`** — Combined `{agencyId}_{entityId}` identifier of the vehicle currently assigned to this block. Absent when no vehicle is assigned.

**`data.entry.status.occupancyStatus`** — Current passenger load as a GTFS-RT occupancy category string (e.g., `"EMPTY"`, `"MANY_SEATS_AVAILABLE"`, `"STANDING_ROOM_ONLY"`, `"FULL"`). Empty string when no real-time occupancy data is available.

**`data.entry.status.occupancyCount`** — Raw passenger count reported by the vehicle's automatic passenger counter. `-1` when unavailable.

**`data.entry.status.occupancyCapacity`** — Reported vehicle capacity in passengers. `-1` when unavailable.

**`data.entry.status.vehicleFeatures`** — List of feature strings describing the vehicle (agency-defined; may be empty).

**`data.entry.status.situationIds`** — Combined IDs of service alerts applicable to the vehicle's current journey. Full records are in `references.situations`; see [[Situation-References]] for the shared dereferencing and ID-formation rules. The ID prefixing used here deviates from Java in two ways — see [Implementation Decisions](#implementation-decisions).

---

### data.entry.frequency (also data.entry.schedule.frequency, data.entry.status.frequency)

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

**`frequency.startTime`** — Unix millisecond timestamp of the start of the frequency window on this service date.

**`frequency.endTime`** — Unix millisecond timestamp of the end of the frequency window on this service date.

**`frequency.headway`** — Target interval between successive departures, in seconds.

**`frequency.exactTimes`** — `0` for headway-based service (departures are approximately every `headway` seconds, not at fixed times); `1` for schedule-based service (exact departure times are derived by repeating the trip at multiples of `headway` from `startTime`).

---

### references.trips[]

```json
{
  "type": "object",
  "properties": {
    "id": { "type": "string" },
    "routeId": { "type": "string" },
    "routeShortName": { "type": "string" },
    "tripShortName": { "type": "string" },
    "tripHeadsign": { "type": "string" },
    "serviceId": { "type": "string" },
    "shapeId": { "type": "string" },
    "directionId": { "type": "string" },
    "blockId": { "type": "string" },
    "timeZone": { "type": "string" },
    "peakOffpeak": { "type": "integer" }
  }
}
```

**`references.trips[].id`** — Combined `{agencyId}_{entityId}` trip identifier.

**`references.trips[].routeId`** — Combined ID of the route this trip belongs to.

**`references.trips[].routeShortName`** — Short route name for this specific trip, when it differs from the route-level default (e.g., a branch variation). Empty string if not set.

**`references.trips[].tripShortName`** — A short name for the trip (e.g., a train number). Empty string if not set.

**`references.trips[].tripHeadsign`** — The destination text shown on the vehicle's front display for this trip.

**`references.trips[].serviceId`** — The service calendar ID that determines on which dates this trip runs.

**`references.trips[].shapeId`** — Combined ID of the geographic shape the vehicle follows. Use the `shape` endpoint to retrieve the encoded polyline.

**`references.trips[].directionId`** — `"0"` for outbound, `"1"` for inbound; the agency defines which direction each value represents.

**`references.trips[].blockId`** — Combined ID of the block this trip belongs to. All trips in the same block are operated by the same vehicle on the same service day.

**`references.trips[].timeZone`** — A per-trip timezone override. Empty string in most feeds, which inherit the agency timezone.

**`references.trips[].peakOffpeak`** — Indicates whether the trip operates during peak hours. `1` for peak, `0` for off-peak or unspecified.
