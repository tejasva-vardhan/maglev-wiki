# arrivals-and-departures-for-stop

## Goal in Context

A rider's client application retrieves the live departure board for a specific stop: the set of vehicle arrivals and departures — with predicted times where real-time data is available — within a caller-specified window around a reference time, so the rider can see how long to wait and plan accordingly.

## Scope

OBA REST API — `GET /api/where/arrivals-and-departures-for-stop/{id}.json`

## Level

User goal

## Primary Actor

Rider (via a client application)

## Stakeholders and Interests

- **Rider** — wants accurate, up-to-date arrival/departure times at their chosen stop to decide when to leave and which vehicle to board.

## Preconditions

- A valid API key is supplied.
- The caller knows the combined `{agencyId}_{stopId}` identifier of the stop.

## Minimal Guarantees

- All responses use the standard envelope with `code`, `version`, `currentTime`, `text`, and `data`.
- If the stop is not found or a service error occurs, the system returns an appropriate error code with no `data` body.

## Success Guarantees

- The response contains every arrival and departure at the stop whose scheduled or predicted time falls within the requested window, sorted by best arrival time ascending.
- Full route, trip, stop, and agency objects for all IDs in the response are present in the references block.
- Service alerts that affect the stop or any route currently or historically serving the stop are included.

## Trigger

A GET request to `/api/where/arrivals-and-departures-for-stop/{id}.json` with a valid API key.

## Main Success Scenario

1. The caller sends a GET request with a stop ID encoded in the URL path and optional time-window query parameters.

2. The system validates that the `id` path parameter is present. If absent, it returns a 400 validation error.

3. The system resolves the stop by parsing the combined `{agencyId}_{stopId}` identifier. If the stop does not exist, it returns 404.

4. The system determines the **query time**: the value of the `time` parameter if supplied, otherwise the current server time.

5. The system computes two time windows from the query time ([`ArrivalsAndDeparturesQueryBean` defaults](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data/src/main/java/org/onebusaway/transit_data/model/ArrivalsAndDeparturesQueryBean.java#L35-L41)):
   - **Standard window**: `[time − minutesBefore, time + minutesAfter]` (defaults: 5 min before, 35 min after).
   - **Frequency window**: `[time − frequencyMinutesBefore, time + frequencyMinutesAfter]` (defaults: 2 min before, 30 min after), applied only to frequency-based trips.

   The transit data layer is queried over the union of both windows to obtain all candidate instances.

6. For each candidate arrival/departure instance ([`getArrivalsAndDeparturesByStopId`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/ArrivalsAndDeparturesBeanServiceImpl.java#L212-L243)):
   - **Agency schedule filter**: If the instance belongs to an agency configured server-side to hide scheduled information, and the instance has neither a predicted arrival nor a predicted departure time, it is excluded.
   - **Time-range filter**: The instance is included only if at least one of its scheduled arrival, scheduled departure, predicted arrival, or predicted departure time falls within its applicable window (frequency or standard, depending on whether the trip is frequency-based).
   - **Negative-arrival filter**: If the agency is configured to suppress arrivals where the vehicle has already passed the stop, and the vehicle is past this stop with no real-time prediction, the instance is excluded.
   - **Filter chain**: Any server-side configured arrival filters are applied.

7. The surviving instances are sorted by best arrival time — predicted arrival time if real-time data is available, scheduled arrival time otherwise — in ascending order ([`ArrivalAndDepartureComparator`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/ArrivalsAndDeparturesBeanServiceImpl.java#L548-L559)).

8. The system looks up all stops within 100 metres of the queried stop, excluding the queried stop itself ([`getNearbyStops`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L74-L75)).

9. The system collects service alerts in two passes ([`getArrivalsAndDeparturesByStopId`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L80-L91)):
   - Alerts currently affecting the stop entity.
   - Alerts affecting any route that serves this stop (including routes with no currently-active trips), gathered by querying all route IDs associated with the stop.

10. The system builds the response entry containing `stopId`, the sorted arrivals list, nearby stop IDs, and the stop-level alert IDs.

11. Each arrival/departure entry is built with ([`getArrivalAndDeparture`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1265-L1348)):
    - IDs for the route, trip, and stop (all in combined `{agencyId}_{entityId}` form); full objects for these are added to the references block.
    - The route short name, resolved in priority order: arrival-specific short name → trip-specific short name → route short name. Similarly, the trip headsign is resolved from the arrival-specific value first, then the trip.
    - Scheduled and predicted arrival/departure times in Unix milliseconds (predicted times are 0 when unavailable).
    - `arrivalEnabled` is `true` for every stop except the first stop of the trip; `departureEnabled` is `true` for every stop except the last stop of the trip.
    - Vehicle distance from stop (metres, positive = approaching, negative = already passed) and number of stops away, when a block location is known.
    - A `tripStatus` sub-object when block location data is available. The `tripStatus.activeTripId` may differ from the entry's `tripId` when the vehicle is currently executing an earlier trip in the same block and has not yet started the trip that will call at this stop.
    - For frequency-based trips with a predicted time, both `scheduledArrivalTime` and `scheduledDepartureTime` are set equal to the predicted time, so legacy clients that show the scheduled time receive the best available information.
    - Per-arrival service alert IDs.

12. The routes in the references block are sorted by a deployment-configurable comparator (typically by route short name) ([`getResponse`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L207-L218)).

13. The system returns HTTP 200 with the assembled entry and references block.

## Extensions

**2a. `id` parameter absent:**
The system returns HTTP 400 with a validation error body.

**3a. Stop not found or service error during lookup:**
The system returns HTTP 404 with `"text": "resource not found"` and no data body. A `ServiceException` during the stop lookup is also mapped to 404. An unexpected runtime exception is mapped to 500.

**3b. Result is `null` after a successful call:**
The system returns HTTP 404. ([`show()` null guard](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForStopAction.java#L139-L142))

**6a. A cancelled trip (status `CANCELED`):**
The arrival/departure entry is included in the list with `status` set to `"CANCELED"` and no predicted times. The `tripStatus` sub-object is omitted when the server is configured to hide cancelled trips; otherwise it is included.

**11a. No block location available for a trip:**
`distanceFromStop`, `numberOfStopsAway`, `vehicleId`, `predicted`, `lastUpdateTime`, and `tripStatus` are absent or zero. `predicted` is `false`. The scheduled times are still present.

## Suspected Defects

### Defects that affect the use case

**`BeanFactoryV2.isSituationExcludedForApplication()` always returns `false` when an API key is present.**

The method is intended to suppress service alerts that are targeted at specific application IDs when the requesting application is not among them. The final `return` statement reads:

```java
return !_applicationKey.contains(_applicationKey);
```

[`BeanFactoryV2.java#L1381`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1381)

A string always contains itself, so this expression is always `false`, meaning the method never signals exclusion. The intended expression is `!applicationIds.contains(_applicationKey)`. As a result, service alerts with application-specific `affects` entries are included in every response regardless of which application key the caller supplied.

### Implementation defects only

**`ArrivalAndDepartureV2Bean.hasPredictedDepartureTime()` checks the wrong field.**

The method reads `return this.predictedArrivalTime > 0` instead of `return this.predictedDepartureTime > 0`.

[`ArrivalAndDepartureV2Bean.java#L311`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/ArrivalAndDepartureV2Bean.java#L311)

This does not affect REST API serialisation (the `predictedDepartureTime` field is written directly to JSON and not gated on this method). It would affect Java clients that call `computeBestDepartureTime()` on the response bean: when a vehicle has a predicted departure but no predicted arrival, `computeBestDepartureTime()` would incorrectly fall back to the scheduled departure time.

## Implementation Decisions

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

---

**`data.entry.situationIds` is always emitted as `[]`, never omitted, though the field description above calls for omitting it when there are no alerts.**

`models.ArrivalsAndDeparturesEntry.SituationIDs` (`internal/models/arrival_and_departure.go`) lacks `omitempty` and is always assigned a non-nil (possibly empty) slice, so Maglev always emits `situationIds: []` at the top level rather than omitting the key when no alerts apply. This predates PR #1313 — it is not something that PR introduced — but PR #1313's changes sit directly in this code path.

This is not simply an oversight to correct casually: iOS's Swift models decode `situationIds` as a **required** (non-optional) field — `try container.decode([String].self, ...)` — on this endpoint as well as `trip-for-vehicle`, `trip-details`, and `arrival-and-departure-for-stop`. If Maglev started omitting the key to match the spec, decoding would throw on iOS for the ordinary no-alerts case, breaking decoding for those clients. Maglev's current spec-deviating behaviour is load-bearing, not incidental.

If this is reconsidered in the future, it needs to be coordinated with an iOS-side change to decode `situationIds` as optional (`decodeIfPresent`) first — changing Maglev's output before that lands would break the iOS app.

## Open Questions

None. All behaviours were determinable from the source code and confirmed against the live server.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Combined {agencyId}_{stopId} identifier, encoded in the URL path"
    },
    "key": {
      "type": "string",
      "description": "API key"
    },
    "time": {
      "type": "number",
      "description": "Unix ms"
    },
    "minutesBefore": {
      "type": "integer",
      "default": 5
    },
    "minutesAfter": {
      "type": "integer",
      "default": 35
    },
    "frequencyMinutesBefore": {
      "type": "integer",
      "default": 2
    },
    "frequencyMinutesAfter": {
      "type": "integer",
      "default": 30
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

**`id`** — The stop to query, in combined `{agencyId}_{stopId}` form (e.g. `1_75403`). Required; encoded directly in the URL path before the format extension.

**`key`** — API key. Required on every request.

**`time`** — Reference time for the query as a Unix millisecond timestamp. Defaults to the current server time. Passing a past or future value is useful for testing or for building historical departure boards.

**`minutesBefore`** — How far before the reference time (in minutes) to include arrivals and departures whose scheduled or predicted time falls in that window. Default 5. Applies to regular scheduled trips only; see `frequencyMinutesBefore` for frequency-based trips.

**`minutesAfter`** — How far after the reference time (in minutes) to include arrivals and departures. Default 35.

**`frequencyMinutesBefore`** — Equivalent of `minutesBefore` for frequency-based (headway-based) trips. Default 2.

**`frequencyMinutesAfter`** — Equivalent of `minutesAfter` for frequency-based trips. Default 30.

**`version`** — API version selector. Default 2. Only version 2 is described in this spec and relevant for the Go rewrite.

**`includeReferences`** — When `false`, the references block in the response is omitted. Default `true`.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "code":        { "type": "integer" },
    "version":     { "type": "integer" },
    "currentTime": { "type": "number", "description": "Unix ms" },
    "text":        { "type": "string" },
    "data": {
      "type": "object",
      "properties": {
        "entry":      { "$ref": "#/definitions/entry" },
        "references": { "$ref": "#/definitions/references" }
      }
    }
  }
}
```

**`code`** — HTTP-equivalent status code mirrored inside the body (200 on success).

**`version`** — API version in effect for this response (typically 2).

**`currentTime`** — Unix millisecond timestamp at the moment the server generated the response.

**`text`** — Human-readable status string (e.g. `"OK"`).

**`data.entry`** — The stop departure board; see below.

**`data.references`** — Full objects for every route, trip, stop, agency, and service alert ID referenced in the entry. Each stop record's `direction` field is taken from the feed when defined, otherwise inferred from shape geometry — see [[Stop-Direction-Calculation]].

---

### data.entry

```json
{
  "type": "object",
  "properties": {
    "stopId":                 { "type": "string" },
    "arrivalsAndDepartures":  { "type": "array", "items": { "$ref": "#/definitions/arrivalAndDeparture" } },
    "nearbyStopIds":          { "type": "array", "items": { "type": "string" } },
    "situationIds":           { "type": "array", "items": { "type": "string" } }
  }
}
```

**`data.entry.stopId`** — Combined `{agencyId}_{stopId}` of the queried stop.

**`data.entry.arrivalsAndDepartures[]`** — List of vehicle arrivals/departures at the stop within the requested time window, sorted by best arrival time ascending. May be empty when no trips are scheduled within the window.

**`data.entry.nearbyStopIds[]`** — Combined IDs of other stops within 100 metres of the queried stop (excluding the queried stop itself). Full stop objects for each ID appear in `references.stops`.

**`data.entry.situationIds[]`** — IDs of service alerts that affect this stop or any route serving it. Full alert objects appear in `references.situations`. Omitted when there are no applicable alerts — **Maglev deviates from this and always emits `[]`**, and its ID prefixing also deviates from Java in two ways — see [Implementation Decisions](#implementation-decisions).

---

### data.entry.arrivalsAndDepartures[]

```json
{
  "type": "object",
  "properties": {
    "routeId":                    { "type": "string" },
    "tripId":                     { "type": "string" },
    "serviceDate":                { "type": "number", "description": "Unix ms" },
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
    "scheduledArrivalTime":       { "type": "number", "description": "Unix ms" },
    "scheduledDepartureTime":     { "type": "number", "description": "Unix ms" },
    "predictedArrivalTime":       { "type": "number", "description": "Unix ms" },
    "predictedDepartureTime":     { "type": "number", "description": "Unix ms" },
    "scheduledArrivalInterval":   { "$ref": "#/definitions/timeInterval" },
    "scheduledDepartureInterval": { "$ref": "#/definitions/timeInterval" },
    "predictedArrivalInterval":   { "$ref": "#/definitions/timeInterval" },
    "predictedDepartureInterval": { "$ref": "#/definitions/timeInterval" },
    "frequency":                  { "$ref": "#/definitions/frequency" },
    "status":                     { "type": "string" },
    "predicted":                  { "type": "boolean" },
    "lastUpdateTime":             { "type": "number", "description": "Unix ms" },
    "distanceFromStop":           { "type": "number" },
    "numberOfStopsAway":          { "type": "integer" },
    "occupancyStatus":            { "type": "string" },
    "historicalOccupancy":        { "type": "string" },
    "scheduledTrack":             { "type": "string" },
    "actualTrack":                { "type": "string" },
    "situationIds":               { "type": "array", "items": { "type": "string" } },
    "tripStatus":                 { "$ref": "#/definitions/tripStatus" }
  }
}
```

**`routeId`** — Combined `{agencyId}_{routeId}` of the route this vehicle is serving. A full route object appears in `references.routes`.

**`tripId`** — Combined `{agencyId}_{tripId}` of the scheduled trip. A full trip object appears in `references.trips`. Note: in interlining scenarios, the vehicle may currently be executing a different trip in the same block; `tripStatus.activeTripId` identifies the vehicle's current trip.

**`serviceDate`** — Unix milliseconds at midnight of the service date for this trip. Times past midnight belong to the service date of the previous calendar day. For example, a 01:00 Tuesday departure has a service date of Monday midnight.

**`vehicleId`** — Combined `{agencyId}_{vehicleId}` of the vehicle currently assigned, or empty when no real-time data is available.

**`stopId`** — Combined `{agencyId}_{stopId}` of the stop this entry describes. Full stop object in `references.stops`.

**`stopSequence`** — 0-indexed position of this stop within the trip's stop sequence, as generated internally (not the GTFS `stop_sequence` value). The first stop of the trip is always 0; the last is always `totalStopsInTrip − 1`.

**`blockTripSequence`** — 0-indexed position of this trip within its block. Compare against `tripStatus.blockTripSequence` to determine whether the vehicle has already started the block trip that will serve this stop.

**`totalStopsInTrip`** — Total number of stop visits in the trip (each visit counted separately if the same stop appears more than once).

**`routeShortName`** — Short name for the route as it applies to this particular arrival/departure, resolved in order: stop-specific headsign override → trip-level short name → route short name. May differ from the `shortName` in the referenced route object.

**`routeLongName`** — Long name of the route. Taken directly from the route record.

**`tripHeadsign`** — Destination text for this trip at this stop, resolved from a stop-time-level headsign override first, then the trip headsign.

**`arrivalEnabled`** — `true` when passengers can board this vehicle at this stop (i.e., this is not the first stop of the trip).

**`departureEnabled`** — `true` when this vehicle will continue beyond this stop (i.e., this is not the last stop of the trip).

**`scheduledArrivalTime`** — Scheduled arrival time in Unix milliseconds. For frequency-based trips with a real-time prediction, this is set to the predicted arrival time so legacy clients that only read the scheduled field still receive the best available time.

**`scheduledDepartureTime`** — Scheduled departure time in Unix milliseconds. Same frequency-based override applies as for `scheduledArrivalTime`.

**`predictedArrivalTime`** — Predicted arrival time in Unix milliseconds when real-time data is available; `0` otherwise.

**`predictedDepartureTime`** — Predicted departure time in Unix milliseconds when real-time data is available; `0` otherwise.

**`scheduledArrivalInterval`** — Optional uncertainty window `{from, to}` in Unix milliseconds around the scheduled arrival time. `null` when not provided.

**`scheduledDepartureInterval`** — Optional uncertainty window around the scheduled departure time. `null` when not provided.

**`predictedArrivalInterval`** — Optional uncertainty window around the predicted arrival time. `null` when not provided.

**`predictedDepartureInterval`** — Optional uncertainty window around the predicted departure time. `null` when not provided.

**`frequency`** — Present only for frequency-based (headway-based) trips. Describes the repeating schedule: the time window over which the service runs and the headway in seconds. `null` for regular timetabled trips.

**`status`** — Textual status of this arrival. `"default"` for a normal scheduled or real-time trip; `"CANCELED"` when the trip has been cancelled via real-time data. Other values may appear for agency-specific status extensions.

**`predicted`** — `true` when real-time GPS data was used to compute the predicted times; `false` when times are derived from the static schedule.

**`lastUpdateTime`** — Unix millisecond timestamp of the most recent real-time update for this vehicle. `0` when no real-time data is available.

**`distanceFromStop`** — Distance in metres between the vehicle's current (or schedule-projected) position and this stop, along the route. Positive values indicate the vehicle is approaching; negative values indicate it has already passed the stop. Present whenever block location information is available (which includes schedule-derived positions).

**`numberOfStopsAway`** — Number of stops between the vehicle's current position and this stop, not counting the current stop. Negative when the vehicle has already passed.

**`occupancyStatus`** — Current vehicle occupancy category reported via real-time data (e.g. `"MANY_SEATS_AVAILABLE"`, `"STANDING_ROOM_ONLY"`). Empty string when not available.

**`historicalOccupancy`** — Typical occupancy category derived from historical ridership data. Empty string when not available.

**`scheduledTrack`** — Track or platform number from the published schedule, when applicable to the service type. Empty string otherwise.

**`actualTrack`** — Track or platform number from real-time data, when applicable. Empty string otherwise.

**`situationIds[]`** — IDs of service alerts that specifically affect this arrival/departure (e.g. an alert scoped to this route/stop/trip combination). Full alert objects in `references.situations`. The ID prefixing used here deviates from Java in two ways — see [Implementation Decisions](#implementation-decisions).

**`tripStatus`** — Real-time or schedule-projected status of the vehicle executing this trip. Present whenever block location information is available; absent otherwise. See below for structure. When the vehicle is in the middle of a block and has not yet started the trip that will call at this stop, `tripStatus.activeTripId` will name the vehicle's current trip while the parent entry's `tripId` names the future trip.

---

### data.entry.arrivalsAndDepartures[].tripStatus

```json
{
  "type": "object",
  "properties": {
    "activeTripId":                { "type": "string" },
    "blockTripSequence":           { "type": "integer" },
    "serviceDate":                 { "type": "number", "description": "Unix ms" },
    "frequency":                   { "$ref": "#/definitions/frequency" },
    "scheduledDistanceAlongTrip":  { "type": "number" },
    "totalDistanceAlongTrip":      { "type": "number" },
    "position":                    { "type": "object", "properties": { "lat": { "type": "number" }, "lon": { "type": "number" } } },
    "orientation":                 { "type": "number" },
    "closestStop":                 { "type": "string" },
    "closestStopTimeOffset":       { "type": "integer" },
    "nextStop":                    { "type": "string" },
    "nextStopTimeOffset":          { "type": "integer" },
    "phase":                       { "type": "string" },
    "status":                      { "type": "string" },
    "predicted":                   { "type": "boolean" },
    "lastUpdateTime":              { "type": "number", "description": "Unix ms" },
    "lastLocationUpdateTime":      { "type": "number", "description": "Unix ms" },
    "distanceAlongTrip":           { "type": "number" },
    "scheduleDeviation":           { "type": "integer" },
    "vehicleId":                   { "type": "string" },
    "occupancyStatus":             { "type": "string" },
    "occupancyCount":              { "type": "integer" },
    "occupancyCapacity":           { "type": "integer" },
    "vehicleFeatures":             { "type": "array", "items": { "type": "string" } },
    "situationIds":                { "type": "array", "items": { "type": "string" } }
  }
}
```

**`tripStatus.activeTripId`** — Combined `{agencyId}_{tripId}` of the trip the vehicle is currently executing. May differ from the parent entry's `tripId` when the vehicle has not yet started the trip that will call at the queried stop.

**`tripStatus.blockTripSequence`** — Index of the currently-active trip within the vehicle's block. Compare against the parent entry's `blockTripSequence` to determine whether the vehicle is on a preceding or the same trip.

**`tripStatus.serviceDate`** — Unix milliseconds at midnight of the service date for the currently-active trip.

**`tripStatus.frequency`** — Frequency descriptor, present only for frequency-based trips.

**`tripStatus.scheduledDistanceAlongTrip`** — Distance in metres from the start of the active trip to where the vehicle is expected to be at the query time, based on the schedule.

**`tripStatus.totalDistanceAlongTrip`** — Total route length of the active trip in metres.

**`tripStatus.position`** — Current latitude/longitude of the vehicle. Schedule-derived when no real-time GPS is available.

**`tripStatus.orientation`** — Vehicle bearing in degrees (0 = north, clockwise).

**`tripStatus.closestStop`** — Combined stop ID of the stop closest to the vehicle's current position.

**`tripStatus.closestStopTimeOffset`** — Seconds between the scheduled time at `closestStop` and the query time (positive = stop is in the future, negative = stop has been passed).

**`tripStatus.nextStop`** — Combined stop ID of the next stop the vehicle will serve.

**`tripStatus.nextStopTimeOffset`** — Seconds until the vehicle is scheduled (or predicted) to depart from `nextStop`.

**`tripStatus.phase`** — Operational phase of the vehicle (e.g. `"in_progress"`, `"layover_before"`). Empty string when not available.

**`tripStatus.status`** — High-level status string for the trip (e.g. `"default"`, `"CANCELED"`).

**`tripStatus.predicted`** — `true` when real-time GPS data was used; `false` for schedule-derived position.

**`tripStatus.lastUpdateTime`** — Unix ms of the most recent real-time update.

**`tripStatus.lastLocationUpdateTime`** — Unix ms of the most recent GPS position update.

**`tripStatus.distanceAlongTrip`** — Metres from the start of the active trip to the vehicle's real-time position. Present only when real-time data is available.

**`tripStatus.scheduleDeviation`** — Seconds the vehicle is running late (positive) or early (negative) relative to its schedule. `0` when schedule-derived.

**`tripStatus.vehicleId`** — Combined `{agencyId}_{vehicleId}`.

**`tripStatus.occupancyStatus`** — Real-time occupancy category for the vehicle. Empty string when not available.

**`tripStatus.occupancyCount`** — Current passenger count when reported. `-1` when not available.

**`tripStatus.occupancyCapacity`** — Vehicle capacity when reported. `-1` when not available.

**`tripStatus.vehicleFeatures[]`** — List of feature strings describing vehicle capabilities (e.g. accessibility features).

**`tripStatus.situationIds[]`** — IDs of service alerts affecting this vehicle/trip. Full alert objects in `references.situations`. The ID prefixing used here deviates from Java in two ways — see [Implementation Decisions](#implementation-decisions).

---

### data.entry.arrivalsAndDepartures[].frequency

Present only for frequency-based (headway-based) trips.

```json
{
  "type": "object",
  "properties": {
    "startTime": { "type": "number", "description": "Unix ms" },
    "endTime":   { "type": "number", "description": "Unix ms" },
    "headway":   { "type": "integer" },
    "exactTimes":{ "type": "integer" }
  }
}
```

**`frequency.startTime`** — Unix ms at the start of the frequency-based service window.

**`frequency.endTime`** — Unix ms at the end of the frequency-based service window.

**`frequency.headway`** — Target interval between departures, in seconds.

**`frequency.exactTimes`** — `1` when departures occur at exact multiples of `headway` from `startTime`; `0` for approximate headway-based service.

---

### Time interval (scheduledArrivalInterval, predictedDepartureInterval, etc.)

Used for `scheduledArrivalInterval`, `scheduledDepartureInterval`, `predictedArrivalInterval`, `predictedDepartureInterval` when present. `null` when the server has no uncertainty data for that time.

```json
{
  "type": "object",
  "properties": {
    "from": { "type": "number", "description": "Unix ms" },
    "to":   { "type": "number", "description": "Unix ms" }
  }
}
```

**`from`** — Earliest bound of the uncertainty window in Unix milliseconds.

**`to`** — Latest bound of the uncertainty window in Unix milliseconds.
