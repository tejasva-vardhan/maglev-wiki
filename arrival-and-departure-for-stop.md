# arrival-and-departure-for-stop

## Goal in Context

A rider wants to see real-time status for one specific vehicle arrival at a specific stop — the predicted time, how far away the vehicle is, and whether it is running on schedule — so they know exactly when their bus will arrive at that stop.

## Scope

The OBA REST API, specifically the single-arrival lookup endpoint exposed at `/api/where/arrival-and-departure-for-stop/{stopId}`.

## Level

User goal.

## Primary Actor

Rider (via a client application).

## Stakeholders and Interests

- **Rider**: Wants accurate, real-time arrival information for a specific trip at a specific stop so they can make a precise boarding decision.

## Preconditions

- The server has loaded a valid GTFS feed and the transit graph is ready.
- The caller knows the stop ID, trip ID, and service date of the arrival they wish to inspect. These are typically obtained from a prior call to `arrivals-and-departures-for-stop`.

## Minimal Guarantees

- If required parameters are missing, the server responds with HTTP 400 and a field-error body identifying which parameters are absent.
- If the block instance for the given trip and service date cannot be found, the server responds with HTTP 404.

## Success Guarantees

- The server returns HTTP 200 with a single arrival-and-departure record describing the scheduled times, real-time predicted times (if available), vehicle distance from the stop, occupancy, and trip status for the specified trip at the specified stop.

## Trigger

An HTTP GET request to `/api/where/arrival-and-departure-for-stop/{stopId}.json` with at minimum `tripId` and `serviceDate` query parameters.

## Main Success Scenario

1. The caller supplies the stop ID in the URL path, the trip ID and service date as query parameters, and optionally the vehicle ID, stop sequence, and query time.

2. The server validates that `id` (stop), `tripId`, and `serviceDate` are all present. If any are missing it returns HTTP 400 immediately.

3. The server resolves the stop and trip from their combined IDs (`{agencyId}_{entityId}` form). Both must exist in the loaded transit graph.

4. The server looks up block instances for the trip's block on the given service date, optionally filtered to the supplied vehicle ID. The query time (defaulting to the current server time if not supplied) is used to determine vehicle position and real-time status. ([`ArrivalAndDepartureServiceImpl.java#L253`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/ArrivalAndDepartureServiceImpl.java#L253))

5. The server locates the specific stop-time entry within the trip:
   - If `stopSequence` was supplied, the search starts at that position in the trip's stop time list and expands outward one step at a time (checking both before and after the given index) until the stop is found. This tolerates minor schedule edits that shift the stop's position by a few places. ([`ArrivalAndDepartureServiceImpl.java#L1124`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/ArrivalAndDepartureServiceImpl.java#L1124))
   - If `stopSequence` was not supplied, the server selects the stop visit whose scheduled time is closest to the query time, resolving ambiguity in the uncommon case where the same stop appears more than once in the trip. ([`ArrivalAndDepartureServiceImpl.java#L1151`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/ArrivalAndDepartureServiceImpl.java#L1151))

6. The server applies real-time block location data to the arrival record if available, setting predicted arrival and departure times, vehicle distance from the stop, the number of intermediate stops between the vehicle's current position and this stop, and the vehicle ID. ([`ArrivalsAndDeparturesBeanServiceImpl.java#L420`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/ArrivalsAndDeparturesBeanServiceImpl.java#L420))

7. The server attaches any active service alerts that apply to this stop call.

8. The server returns HTTP 200 with a response entry containing the arrival-and-departure record and a populated references block.

## Extensions

**2a. Missing required parameters:**
The server responds with HTTP 400. The body is a JSON object with a `fieldErrors` key listing the missing parameters (e.g. `{"fieldErrors": {"tripId": ["missingRequiredField"]}}`).

**3a. Stop ID not found in the transit graph:**
The server should return HTTP 404 but due to a defect (see Suspected Defects) returns HTTP 200 with a null body instead.

**3b. Trip ID not found in the transit graph:**
The server should return HTTP 404 but due to a defect (see Suspected Defects) returns HTTP 200 with a null body instead.

**4a. No block instance found for the given service date or vehicle:**
The server returns HTTP 404 with a standard error envelope (`{"code": 404, "text": "resource not found", ...}`).

**4b. Trip's block exists but the stop is not in the trip:**
The server returns HTTP 404 (block stop time cannot be found, returns null internally).

**4c. Agency has schedule display suppressed and no real-time prediction is available:**
Agencies may be configured to hide schedule information. If the trip's agency has this configuration and no real-time prediction exists for this arrival, the server returns HTTP 404. ([`ArrivalAndDepartureServiceImpl.java#L280`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/ArrivalAndDepartureServiceImpl.java#L280))

**5a. Multiple block locations match the query:**
When more than one block location is found (which can occur when multiple vehicles are associated with the same block), only the first location is used. ([`ArrivalAndDepartureServiceImpl.java#L276`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/ArrivalAndDepartureServiceImpl.java#L276))

**6a. Trip is cancelled and the server is configured to hide cancelled trips:**
The server returns HTTP 404. If the server is not configured to hide cancelled trips (a deployment-time setting), the record is returned with `status: "CANCELED"`, `predicted: false`, and no vehicle ID. The trip status object is also omitted. ([`ArrivalsAndDeparturesBeanServiceImpl.java#L267`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/ArrivalsAndDeparturesBeanServiceImpl.java#L267))

**6b. No real-time data available:**
`predicted` is `false`. `predictedArrivalTime` and `predictedDepartureTime` are `0` (not absent). `distanceFromStop` is computed from the vehicle's schedule-based position. `vehicleId` in the entry is an empty string.

**6c. Trip is frequency-based:**
For trips scheduled using a headway (frequency) window rather than fixed times, the `scheduledArrivalTime` and `scheduledDepartureTime` fields are overwritten with the current predicted arrival/departure time. A `frequency` object is present in the entry describing the service window. ([`ArrivalsAndDeparturesBeanServiceImpl.java#L429`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/ArrivalsAndDeparturesBeanServiceImpl.java#L429))

**8a. `includeReferences=false` supplied:**
The `references` object is returned with all lists empty. The structure is present but unpopulated.

## Suspected Defects

### Defects that affect the use case

**Invalid stop or trip ID returns HTTP 200 with null body instead of HTTP 404.**
Class: `ExceptionInterceptor`, [`ExceptionInterceptor.java#L69`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ExceptionInterceptor.java#L69).

When the stop ID or trip ID cannot be resolved from the transit graph, a `NoSuchStopServiceException` or `NoSuchTripServiceException` is thrown. The `ExceptionInterceptor` catches these and constructs a 404 response bean, but the JSON serializer writes the action model (which was never set, so it is null) rather than the error bean. The result is HTTP 200 with a four-byte body `null`. The intended behaviour is HTTP 404 with a JSON error envelope matching the format returned for other not-found conditions. *Maglev intentionally corrects this — see [Implementation Decisions](#implementation-decisions).*

### Implementation defects only

**`hasPredictedDepartureTime()` in `ArrivalAndDepartureV2Bean` tests the wrong field.**
Class: `ArrivalAndDepartureV2Bean`, [`ArrivalAndDepartureV2Bean.java#L311`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/ArrivalAndDepartureV2Bean.java#L311).

The method checks `this.predictedArrivalTime > 0` instead of `this.predictedDepartureTime > 0`. This does not affect the JSON API response because the field is serialised directly from `predictedDepartureTime`, not through this method. It would affect any Java consumer that calls `computeBestDepartureTime()` on the V2 bean.

**`setStatus` called twice in succession in `BeanFactoryV2.getArrivalAndDeparture`.**
Class: `BeanFactoryV2`, [`BeanFactoryV2.java#L1318`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1318) and [`BeanFactoryV2.java#L1322`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1322).

`bean.setStatus(ad.getStatus())` is called twice with the same value. Harmless; one call is redundant.

## Implementation Decisions

**Unknown stop or trip ID returns HTTP 404 (deviates from legacy).**

The legacy implementation returns HTTP 200 with a null body when the stop ID or trip ID cannot be resolved (see [Suspected Defects](#suspected-defects)). Maglev intentionally corrects this: an unknown stop or trip ID returns HTTP 404 with a standard error envelope, consistent with the treatment of other unknown entity IDs across the API.

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

## Open Questions

None.

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "tripId", "serviceDate"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Stop ID in combined {agencyId}_{stopId} form. Supplied as the path segment."
    },
    "tripId": {
      "type": "string",
      "description": "Trip ID in combined {agencyId}_{tripId} form."
    },
    "serviceDate": {
      "type": "integer",
      "description": "Unix ms. The service date of the trip. Accepts a Unix millisecond timestamp or a YYYY-MM-DD date string."
    },
    "vehicleId": {
      "type": "string",
      "description": "Vehicle ID in combined {agencyId}_{vehicleId} form. Optional. Narrows the block location lookup to a specific vehicle when multiple vehicles are associated with the same block."
    },
    "stopSequence": {
      "type": "integer",
      "default": -1,
      "description": "0-indexed position of the target stop in the trip's stop time sequence. Optional. When supplied, the server anchors its search at this position and expands outward, which disambiguates stops visited more than once and improves performance. When omitted (default -1), the stop visit closest in scheduled time to the query time is selected."
    },
    "time": {
      "type": "integer",
      "description": "Unix ms. The time at which to evaluate real-time status. Accepts a Unix millisecond timestamp or a datetime string in yyyy-MM-dd_HH-mm-ss format. Defaults to the current server time. Affects vehicle position, distance-from-stop, and trip status computation."
    },
    "key": {
      "type": "string",
      "description": "API key."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "When false, the references block is returned empty. The structure is still present."
    }
  }
}
```

**`id`** — Stop ID in combined `{agencyId}_{entityId}` form. Supplied in the URL path as the segment after `arrival-and-departure-for-stop/`.

**`tripId`** — Trip ID in combined `{agencyId}_{entityId}` form. Identifies which scheduled trip the caller is asking about.

**`serviceDate`** — The service date of the trip as a Unix millisecond timestamp (all-digit string) or a `YYYY-MM-DD` date. A timestamp is read as the service date that contains that instant. Trips running past midnight carry the previous calendar day's service date.

**`vehicleId`** — Optional. Combined `{agencyId}_{vehicleId}` form. Restricts the block location lookup to a specific vehicle, helping when two vehicles are both associated with the same block at the same time.

**`stopSequence`** — Optional. The 0-indexed position of the stop in the trip's stop time list as maintained internally by OBA. This is **not** the GTFS `stop_sequence` value. When provided, the server starts its search at this position and expands outward one step at a time; a mismatch of a few positions (caused by stops being added or removed from a trip) is tolerated.

**`time`** — Optional. The point-in-time at which real-time status is evaluated. Defaults to the current server clock. Useful for replaying historical data or testing.

**`key`** — API authentication key. Required by the framework.

**`includeReferences`** — Optional boolean (default `true`). When `false`, the server skips populating the `references` block, reducing response size when the caller does not need the denormalised entity details.

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
    "data": {
      "type": "object",
      "properties": {
        "entry": { "$ref": "#/definitions/arrivalAndDeparture" },
        "references": { "$ref": "#/definitions/references" }
      }
    }
  }
}
```

**`version`** — API version (always `2`).

**`code`** — HTTP status code mirrored into the response body (`200` on success, `400` for validation errors, `404` when the arrival is not found).

**`currentTime`** — Server clock at response time, Unix milliseconds.

**`text`** — Human-readable status string (e.g. `"OK"`, `"resource not found"`).

**`data.entry`** — The single arrival-and-departure record; see schema below.

**`data.references`** — Denormalised entity details for all IDs referenced in the entry.

---

### data.entry

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
    "blockTripSequence":          { "type": "integer" },
    "totalStopsInTrip":           { "type": "integer" },
    "routeShortName":             { "type": "string" },
    "routeLongName":              { "type": "string" },
    "tripHeadsign":               { "type": "string" },
    "arrivalEnabled":             { "type": "boolean" },
    "departureEnabled":           { "type": "boolean" },
    "scheduledArrivalTime":       { "type": "integer", "description": "Unix ms" },
    "scheduledDepartureTime":     { "type": "integer", "description": "Unix ms" },
    "predictedArrivalTime":       { "type": "integer", "description": "Unix ms; 0 when no real-time data" },
    "predictedDepartureTime":     { "type": "integer", "description": "Unix ms; 0 when no real-time data" },
    "scheduledArrivalInterval":   { "$ref": "#/definitions/timeInterval" },
    "scheduledDepartureInterval": { "$ref": "#/definitions/timeInterval" },
    "predictedArrivalInterval":   { "$ref": "#/definitions/timeInterval" },
    "predictedDepartureInterval": { "$ref": "#/definitions/timeInterval" },
    "frequency":                  { "$ref": "#/definitions/frequency" },
    "predicted":                  { "type": "boolean" },
    "lastUpdateTime":             { "type": "integer", "description": "Unix ms; 0 when no real-time data" },
    "distanceFromStop":           { "type": "number" },
    "numberOfStopsAway":          { "type": "integer" },
    "status":                     { "type": "string" },
    "occupancyStatus":            { "type": "string" },
    "historicalOccupancy":        { "type": "string" },
    "predictedOccupancy":         { "type": "string" },
    "scheduledTrack":             { "type": "string" },
    "actualTrack":                { "type": "string" },
    "tripStatus":                 { "$ref": "#/definitions/tripStatus" },
    "situationIds":               { "type": "array", "items": { "type": "string" } }
  }
}
```

**`data.entry.routeId`** — Combined `{agencyId}_{routeId}` of the route this trip belongs to.

**`data.entry.tripId`** — Combined `{agencyId}_{tripId}` of the requested trip.

**`data.entry.serviceDate`** — Unix milliseconds of the service-day start in the agency's timezone (local noon minus 12 hours, Java `ServiceDate.getAsDate`). That instant is local midnight except on a DST transition day. For trips running past midnight this is the previous calendar day's service date.

**`data.entry.vehicleId`** — Combined `{agencyId}_{vehicleId}` of the vehicle currently serving this trip. Empty string when no real-time vehicle data is available or when the trip is cancelled.

**`data.entry.stopId`** — Combined `{agencyId}_{stopId}` of the stop at which this arrival-and-departure occurs.

**`data.entry.stopSequence`** — The 0-indexed position of this stop in the trip's stop time list as maintained internally by OBA (0 = first stop, `totalStopsInTrip - 1` = last stop). This is **not** the GTFS `stop_sequence` value.

**`data.entry.blockTripSequence`** — The position of this trip within the vehicle's full block (day's sequence of trips). Clients can compare this against `tripStatus.blockTripSequence` to determine whether the vehicle is currently on this trip or an earlier/later trip in the same block.

**`data.entry.totalStopsInTrip`** — Total number of stop-time entries in this trip. If a stop is visited more than once, each visit counts separately.

**`data.entry.routeShortName`** — Short display name of the route (e.g. `"44"`). Resolved through a three-step fallback: stop-time-level narrative override → trip-level narrative → route's own short name. ([`BeanFactoryV2.java#L1288`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1288))

**`data.entry.routeLongName`** — Long display name of the route, taken directly from the route record.

**`data.entry.tripHeadsign`** — The destination text to display on the vehicle. Uses the stop-specific headsign from the stop-time narrative if set; falls back to the trip headsign. ([`BeanFactoryV2.java#L1297`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1297))

**`data.entry.arrivalEnabled`** — `true` if passengers can alight from this vehicle at this stop. `false` for the first stop in the trip (where no arrival is possible because the trip originates there). ([`ArrivalsAndDeparturesBeanServiceImpl.java#L327`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/ArrivalsAndDeparturesBeanServiceImpl.java#L327))

**`data.entry.departureEnabled`** — `true` if passengers can board this vehicle at this stop. `false` for the last stop in the trip. ([`ArrivalsAndDeparturesBeanServiceImpl.java#L328`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/ArrivalsAndDeparturesBeanServiceImpl.java#L328))

**`data.entry.scheduledArrivalTime`** — Scheduled arrival time in Unix milliseconds. For frequency-based trips, this is overwritten with the current predicted arrival time when real-time data is available.

**`data.entry.scheduledDepartureTime`** — Scheduled departure time in Unix milliseconds. For frequency-based trips, overwritten with the predicted departure time when real-time data is available.

**`data.entry.predictedArrivalTime`** — Predicted arrival time in Unix milliseconds, or `0` if no real-time prediction is available.

**`data.entry.predictedDepartureTime`** — Predicted departure time in Unix milliseconds, or `0` if no real-time prediction is available.

**`data.entry.scheduledArrivalInterval`** — Confidence interval on the scheduled arrival, expressed as a `{from, to}` time range. Typically `null` for point-scheduled trips.

**`data.entry.scheduledDepartureInterval`** — Confidence interval on the scheduled departure. Typically `null`.

**`data.entry.predictedArrivalInterval`** — Confidence interval on the predicted arrival. `null` when no real-time data is present or when no uncertainty model is available.

**`data.entry.predictedDepartureInterval`** — Confidence interval on the predicted departure. `null` under the same conditions.

**`data.entry.frequency`** — Present only for frequency-based (headway) trips. Describes the service window (`startTime`, `endTime`) and the headway in seconds. `null` for point-scheduled trips.

**`data.entry.predicted`** — `true` if real-time GPS data was used to compute the vehicle's position and predicted times; `false` if the position is extrapolated from the schedule. Always `false` for cancelled trips.

**`data.entry.lastUpdateTime`** — Unix milliseconds of the most recent real-time position update for this vehicle. `0` when no real-time data is available.

**`data.entry.distanceFromStop`** — The vehicle's distance from this stop in metres, along the block path. Positive means the vehicle has not yet reached the stop; negative means it has already passed. Uses real-time distance if a GPS fix is available; falls back to schedule-based distance otherwise. ([`ArrivalsAndDeparturesBeanServiceImpl.java#L456`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/ArrivalsAndDeparturesBeanServiceImpl.java#L456))

**`data.entry.numberOfStopsAway`** — The number of intermediate stops between the vehicle's current next-stop and this stop (not counting the current stop itself). Computed as the difference in block stop-time sequence indices. ([`ArrivalsAndDeparturesBeanServiceImpl.java#L467`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/ArrivalsAndDeparturesBeanServiceImpl.java#L467))

**`data.entry.status`** — A string describing the arrival status. `"default"` in normal operation; `"CANCELED"` for a cancelled trip (only present when the server is configured to expose cancelled trips rather than suppress them).

**`data.entry.occupancyStatus`** — Real-time vehicle occupancy category (e.g. `"MANY_SEATS_AVAILABLE"`, `"STANDING_ROOM_ONLY"`). Empty string when no occupancy data is available.

**`data.entry.historicalOccupancy`** — Historical average occupancy for this trip/stop combination. Empty string when unavailable.

**`data.entry.predictedOccupancy`** — Predicted occupancy. Empty string when unavailable.

**`data.entry.scheduledTrack`** — For rail services: the scheduled platform or track identifier. Empty string for bus services or when not provided.

**`data.entry.actualTrack`** — For rail services: the actual (real-time) platform or track identifier. Empty string when unavailable.

**`data.entry.tripStatus`** — Full real-time status of the vehicle operating this trip, including its current position, heading, schedule deviation, and closest stop. Present whenever a block location exists (even if the position is schedule-based rather than GPS-derived). Absent when no block location is available. The `activeTripId` within `tripStatus` may differ from `entry.tripId` in cases of interlining — for example, if the vehicle is currently running the previous trip in the block that connects to the queried trip.

**`data.entry.situationIds`** — IDs of active service alerts that apply to this stop call. Full alert objects are in `references.situations`; see [[Situation-References]] for the shared dereferencing and ID-formation rules. The ID prefixing used here deviates from Java in two ways — see [Implementation Decisions](#implementation-decisions).

---

### data.entry.tripStatus

```json
{
  "type": "object",
  "properties": {
    "activeTripId":                  { "type": "string" },
    "blockTripSequence":             { "type": "integer" },
    "serviceDate":                   { "type": "integer", "description": "Unix ms" },
    "frequency":                     { "$ref": "#/definitions/frequency" },
    "scheduledDistanceAlongTrip":    { "type": "number" },
    "totalDistanceAlongTrip":        { "type": "number" },
    "position": {
      "type": "object",
      "properties": {
        "lat": { "type": "number" },
        "lon": { "type": "number" }
      }
    },
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
    "lastKnownDistanceAlongTrip":    { "type": "number" },
    "lastKnownLocation": {
      "type": "object",
      "properties": {
        "lat": { "type": "number" },
        "lon": { "type": "number" }
      }
    },
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

**`data.entry.tripStatus.activeTripId`** — Combined `{agencyId}_{tripId}` of the trip the vehicle is currently executing. May differ from `entry.tripId` when the vehicle is on a different trip in the same block (interlining).

**`data.entry.tripStatus.blockTripSequence`** — The position of the *active* trip within the block. Compare against `entry.blockTripSequence` to tell whether the vehicle is still on a prior trip (lower value) or has already reached this trip (equal value).

**`data.entry.tripStatus.serviceDate`** — Unix milliseconds of the service-day start for the active trip's service date, same as `data.entry.serviceDate`.

**`data.entry.tripStatus.frequency`** — Frequency window if the active trip is headway-based. `null` for point-scheduled trips.

**`data.entry.tripStatus.scheduledDistanceAlongTrip`** — The vehicle's expected distance along the active trip at the query time, in metres.

**`data.entry.tripStatus.totalDistanceAlongTrip`** — The total length of the active trip, in metres.

**`data.entry.tripStatus.position`** — Current geographic position of the vehicle (lat/lon). Derived from GPS if real-time data is available; extrapolated from the schedule otherwise.

**`data.entry.tripStatus.orientation`** — Vehicle heading in degrees clockwise from north, derived from the trip shape when real-time orientation is unavailable.

**`data.entry.tripStatus.closestStop`** — Combined ID of the stop nearest to the vehicle's current position.

**`data.entry.tripStatus.closestStopTimeOffset`** — Seconds between the vehicle's current position and the scheduled time at `closestStop`. Negative means the vehicle has passed that stop.

**`data.entry.tripStatus.nextStop`** — Combined ID of the next stop the vehicle will serve.

**`data.entry.tripStatus.nextStopTimeOffset`** — Seconds until the vehicle is scheduled to reach `nextStop`.

**`data.entry.tripStatus.phase`** — Operating phase of the vehicle (e.g. in-progress, layover). Empty string when unavailable.

**`data.entry.tripStatus.status`** — Status string for the trip (e.g. `"default"`, `"CANCELED"`).

**`data.entry.tripStatus.predicted`** — `true` if the position was computed from real-time GPS data.

**`data.entry.tripStatus.lastUpdateTime`** — Unix milliseconds of the most recent real-time update for this vehicle.

**`data.entry.tripStatus.lastLocationUpdateTime`** — Unix milliseconds of the most recent GPS position update.

**`data.entry.tripStatus.lastKnownDistanceAlongTrip`** — Distance along the trip at the time of the last known real-time position, in metres.

**`data.entry.tripStatus.lastKnownLocation`** — Geographic coordinates at the time of the last known real-time position.

**`data.entry.tripStatus.lastKnownOrientation`** — Heading at the time of the last known real-time position.

**`data.entry.tripStatus.scheduleDeviation`** — How many seconds late (positive) or early (negative) the vehicle is running. Absent when not determinable.

**`data.entry.tripStatus.distanceAlongTrip`** — Current distance of the vehicle along the active trip, in metres. Absent when not determinable.

**`data.entry.tripStatus.vehicleId`** — Combined `{agencyId}_{vehicleId}` of the vehicle.

**`data.entry.tripStatus.occupancyStatus`** — Vehicle-level occupancy category. Empty string when unavailable.

**`data.entry.tripStatus.occupancyCount`** — Number of passengers on board. `-1` when unavailable.

**`data.entry.tripStatus.occupancyCapacity`** — Rated passenger capacity of the vehicle. `-1` when unavailable.

**`data.entry.tripStatus.vehicleFeatures`** — List of feature tags for the vehicle (e.g. accessibility features).

**`data.entry.tripStatus.situationIds`** — IDs of service alerts applying to this trip's current status. Full alert objects are in `references.situations`; see [[Situation-References]] for the shared dereferencing and ID-formation rules. The ID prefixing used here deviates from Java in two ways — see [Implementation Decisions](#implementation-decisions).

---

### data.entry.frequency

```json
{
  "type": "object",
  "properties": {
    "startTime":  { "type": "integer", "description": "Unix ms" },
    "endTime":    { "type": "integer", "description": "Unix ms" },
    "headway":    { "type": "integer" },
    "exactTimes": { "type": "integer" }
  }
}
```

**`data.entry.frequency.startTime`** — Start of the frequency-based service window, Unix milliseconds.

**`data.entry.frequency.endTime`** — End of the frequency-based service window, Unix milliseconds.

**`data.entry.frequency.headway`** — Approximate service interval in seconds.

**`data.entry.frequency.exactTimes`** — `1` if departures occur at exact multiples of `headway` from `startTime`; `0` for approximate headway operation.

---

### data.references

```json
{
  "type": "object",
  "properties": {
    "agencies":   { "type": "array", "items": { "type": "object" } },
    "routes":     { "type": "array", "items": { "type": "object" } },
    "stops":      { "type": "array", "items": { "type": "object" } },
    "trips":      { "type": "array", "items": { "type": "object" } },
    "situations": { "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.references.agencies`** — Full agency records for all agency IDs referenced in the entry.

**`data.references.routes`** — Full route records for all route IDs referenced in the entry.

**`data.references.stops`** — Full stop records for the queried stop and any stops referenced from `tripStatus`. Each record's `direction` field is taken from the feed when defined, otherwise inferred from shape geometry — see [[Stop-Direction-Calculation]].

**`data.references.trips`** — Full trip records for the queried trip and the active trip referenced in `tripStatus`.

**`data.references.situations`** — Full service alert records for all situation IDs listed in `entry.situationIds` and `entry.tripStatus.situationIds`. No other ambient alert objects are included; see [[Situation-References]].
