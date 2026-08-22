# schedule-for-stop

## Goal in Context

A rider wants to see the complete timetable for a stop on a specific service date — which routes call there, in which directions, and at what times — so they can plan a trip without needing real-time data.

## Scope

OneBusAway REST API — schedule-for-stop endpoint.

## Level

User goal.

## Primary Actor

Rider.

## Stakeholders and Interests

- **Rider** — wants an accurate, complete list of all scheduled arrivals and departures at a stop on a chosen day, organised by route and direction, so they can read it like a printed timetable.

## Preconditions

- The API server is running with GTFS data loaded.
- The caller supplies a valid API key.
- The stop ID is supplied in the combined `{agencyId}_{stopId}` form.

## Minimal Guarantees

- The response is well-formed JSON with a top-level `code` and `version` field.
- When the date parameter is unrecognisable, a field-error response is returned before any stop lookup is attempted.
- When `includeReferences` is not set to `false`, every entity ID appearing in `data.entry` has a corresponding full record in `data.references`.

## Success Guarantees

- `data.entry.stopRouteSchedules` contains one group per route that serves this stop on the queried service date.
- Within each group, stop times are partitioned by direction, sorted by departure time, and accompanied by the most representative headsign for that direction.
- Frequency-based service, where present in the feed, is represented separately from fixed-schedule stop times.
- The stop record for the queried stop appears in `data.references.stops`.
- When references are included, `data.references.routes` is populated from the queried stop's serving-route list, not only from the routes with active trips on the queried service date. `data.references.agencies` is derived from those route references, so a stop with no serving routes has an empty `routes` and `agencies` reference list while still including the queried stop.

## Trigger

An HTTP GET request to `/api/where/schedule-for-stop/{id}.json`.

## Main Success Scenario

1. The client sends GET `/api/where/schedule-for-stop/{id}.json?key=…&date=2026-05-08`.
2. The server validates the `key` parameter (401 if absent or invalid).
3. The server parses the optional `date` parameter. If supplied as `YYYY-MM-DD`, it is interpreted as midnight of that calendar date in the server's local timezone. If supplied as a Unix millisecond integer, the timestamp is used as-is. If omitted, the current server time is used. In all cases the local calendar date is extracted from the timestamp to identify the service date.
4. The server looks up the stop by its combined ID. If the stop does not exist, see extension 3a.
5. The server collects all trips active on the resolved service date that include a stop time at this stop. A trip is active on a service date if its service calendar or calendar date overrides include that date. [(service date resolution)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopScheduleBeanServiceImpl.java#L372-L380)
6. Trips are partitioned first by route (using the route collection ID), then by direction within each route. The direction key is the trip's GTFS `direction_id`, defaulting to `"0"` when absent. [(direction defaulting)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopScheduleBeanServiceImpl.java#L239-L241)
7. Within each direction group, the server selects the most commonly occurring headsign (by trip count; frequency-based entries are weighted by estimated trip count in the interval) as the group's representative `tripHeadsign`. [(headsign selection)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopScheduleBeanServiceImpl.java#L614)
8. Fixed-schedule stop times within each direction group are sorted by departure time. [(stop time sort)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopScheduleBeanServiceImpl.java#L302) Frequency-based entries are sorted by start time. [(frequency sort)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopScheduleBeanServiceImpl.java#L303) Java applies this grouping uniformly regardless of the trip's `frequencies.txt` `exact_times` value — Maglev deviates here for `exact_times=1` trips; see [Implementation Decisions](#implementation-decisions).
9. Direction groups within each route are sorted alphabetically by headsign. [(direction sort)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopScheduleBeanServiceImpl.java#L315)
10. Route groups are sorted by the route's short name (falling back to long name, then combined ID) using a natural-string ordering that sorts embedded numbers numerically (so route "10" follows route "9", not "1"). [(route sort)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopScheduleBeanServiceImpl.java#L535-L551)
11. Each stop time carries two boolean flags. `arrivalEnabled` is `false` only when the stop time is the very first stop in the vehicle's entire block for the service day — meaning the vehicle starts its work here and no inbound arrival exists. `departureEnabled` is `false` only when the stop time is the very last stop in the vehicle's block — meaning the vehicle ends its work here and no onward departure exists. Both are `true` for the vast majority of stop times. [(arrivalEnabled)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopScheduleBeanServiceImpl.java#L236) [(departureEnabled)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopScheduleBeanServiceImpl.java#L237)
12. The server returns HTTP 200 with `code: 200`. The response entry contains the echoed date timestamp, the stop's combined ID, and the organised stop route schedules. The queried stop appears in the references block. Route references are generated from the stop's full serving-route list, and agency references are generated from those routes; this reference membership is independent of which routes have active trips on the queried date. [(stop reference)](https://github.com/OneBusAway/onebusaway-application-modules/blob/fed9494e4b3ff345efabcd17b9b21fbf2a9dab7e/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1068-L1086) [(stop route references)](https://github.com/OneBusAway/onebusaway-application-modules/blob/fed9494e4b3ff345efabcd17b9b21fbf2a9dab7e/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L472-L478) [(route agency references)](https://github.com/OneBusAway/onebusaway-application-modules/blob/fed9494e4b3ff345efabcd17b9b21fbf2a9dab7e/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L432-L442)

## Extensions

**1a. API key missing or invalid:**
At step 2, the server returns HTTP 401 with `code: 401` and text "permission denied".

**2a. Date parameter is not a valid `YYYY-MM-DD` string or integer:**
At step 3, the server returns HTTP 400 with `code: 400` and a `fieldErrors.date` array describing the parsing failure.

**3a. Stop ID is not found (unknown or unrecognisable ID):**
At step 4, the server should return a resource-not-found response. Due to a defect (see Suspected Defects), it instead returns HTTP 200 with a bare `null` body rather than a structured error or a 404 response.

**4a. Queried service date falls outside the feed's validity period:**
At step 5, no trips are active, and `stopRouteSchedules` is an empty array. The response still has HTTP 200 and a valid entry structure. Unless `includeReferences=false`, the queried stop remains in `references.stops`, and `references.routes` / `references.agencies` remain populated from the routes that serve the stop generally rather than from the empty schedule rows. If the stop has no serving routes, route and agency references are empty.

**5a. Stop has frequency-based service:**
At step 8, such trips contribute entries to `scheduleFrequencies` instead of `scheduleStopTimes` within their direction group, regardless of the trip's `frequencies.txt` `exact_times` value — Java does not distinguish `exact_times=0` from `exact_times=1` at this endpoint, and `FrequencyInstanceBean` has no `exactTimes` field. [(frequency grouping)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopScheduleBeanServiceImpl.java#L257-L282) [(FrequencyInstanceBean fields)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data/src/main/java/org/onebusaway/transit_data/model/schedule/FrequencyInstanceBean.java#L20-L40) A direction group may contain both fixed-schedule and frequency-based entries simultaneously. *Maglev intentionally diverges for `exact_times=1` trips — see [Implementation Decisions](#implementation-decisions).*

## Suspected Defects

### Defects that affect the use case

**Null body returned for unknown or unrecognisable stop ID** — `ScheduleForStopAction`

When `getScheduleForStop` cannot find the stop it returns `null`, but the action class passes this directly to the response factory with no null check at [ScheduleForStopAction.java lines 68–71](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ScheduleForStopAction.java#L68-L71). The factory then throws a NullPointerException ([BeanFactoryV2.java line 1070](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1070)), which propagates unhandled through the Struts2 REST layer and results in HTTP 200 with a bare `null` body. The intended behaviour is an HTTP 404 structured response (as produced by `setResourceNotFoundResponse()`). The same applies when the stop ID lacks an underscore separator, which causes `AgencyAndIdLibrary.convertFromString` to throw `IllegalStateException`.

*Maglev intentionally corrects this — see [Implementation Decisions](#implementation-decisions).*

### Implementation defects only

**`stopHeadsign` on fixed-schedule stop times always reads from the output bean** — `BeanFactoryV2`

At [BeanFactoryV2.java line 1130](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1130) the assignment is `stiBean.setStopHeadsign(stiBean.getStopHeadsign())` — it reads from the newly created output bean rather than from the source `sti.getStopHeadsign()`. Because the service layer also never populates `stopHeadsign` on the source `StopTimeInstanceBean`, the observable result is the same either way: the field is always null (serialised as `""` in JSON). A clean reimplementation should populate `stopHeadsign` from the source — but as the service layer never sets it, the field will remain empty until the service layer is fixed too.

## Implementation Decisions

**Unknown stop ID returns HTTP 404 (deviates from legacy).**

The legacy implementation returns HTTP 200 with a null body when the stop ID cannot be resolved (see [Suspected Defects](#suspected-defects)). Maglev intentionally corrects this: an unknown or unresolvable stop ID returns HTTP 404 with a standard error envelope, consistent with the treatment of other unknown entity IDs across the API.

**`exact_times=1` frequency trips are expanded into discrete `scheduleStopTimes` (deviates from legacy).**

Legacy Java does not distinguish `frequencies.txt`'s `exact_times` value anywhere in this endpoint: every frequency-based trip — whether `exact_times=0` (approximate headway) or `exact_times=1` (exact, schedule-equivalent headway) — is grouped into `scheduleFrequencies` as an undifferentiated `FrequencyInstanceBean`, which has no `exactTimes` field at all. [(frequency grouping)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopScheduleBeanServiceImpl.java#L257-L282) [(FrequencyInstanceBean fields)](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data/src/main/java/org/onebusaway/transit_data/model/schedule/FrequencyInstanceBean.java#L20-L40)

Per the GTFS spec, `exact_times=1` means the trip's individual departures are precisely timed and `frequencies.txt` is only a compact encoding of what is functionally a normal timetable, unlike `exact_times=0` where the headway is approximate. Maglev honors that distinction instead of replicating Java's undifferentiated handling: `exact_times=1` trips are expanded in-memory into individual `scheduleStopTimes` entries — one per headway interval within the frequency window, computed from the template trip's arrival/departure offsets — identical in shape to fixed-schedule stop times. `exact_times=0` trips continue to populate `scheduleFrequencies` as an undifferentiated headway window, matching Java.

This special-casing is confined to this endpoint because it is the only place where the underlying legacy bean (`FrequencyInstanceBean`) has no `exactTimes` field to fall back on. Elsewhere — `trip-details`, `trip-for-vehicle`, `arrival-and-departure-for-stop`, `trips-for-route`, `trips-for-location` — the response's `frequency` object matches legacy `FrequencyV2Bean`, which does carry `exactTimes`, so Maglev exposes it as-is there and leaves any client-side expansion to the caller rather than performing it server-side.

Trade-off: this is new behavior with no Java precedent to verify against. A client built against legacy Java's uniform treatment will see more, individually-timed entries in `scheduleStopTimes` for `exact_times=1` routes than Java ever produced, and correspondingly fewer entries in `scheduleFrequencies`.

Fallback: if this proves disruptive to clients expecting Java's undifferentiated behavior, revert to routing `exact_times=1` trips into `scheduleFrequencies` alongside `exact_times=0` trips, matching Java exactly.

## Open Questions

None. All behaviour was determined by static analysis of the full call chain and confirmed by runtime queries.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Combined stop ID encoded in the URL path: /api/where/schedule-for-stop/{id}.json"
    },
    "key": {
      "type": "string",
      "description": "API access key"
    },
    "date": {
      "type": "string",
      "description": "Service date to query. Accepts YYYY-MM-DD or a Unix millisecond integer. Defaults to the current server time."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "When false, the references block is still present but every list within it is empty."
    },
    "version": {
      "type": "integer",
      "default": 2
    }
  }
}
```

**`id`** — Combined stop ID in `{agencyId}_{stopId}` form (e.g., `1_75403`). Encoded as a path segment immediately before the `.json` extension.

**`key`** — API access key. Required; a missing or unrecognised key produces a 401 response.

**`date`** — The service date for which to retrieve the schedule. When given as `YYYY-MM-DD`, it is parsed as midnight of that date in the server's local timezone. When given as a Unix millisecond integer, the local calendar date is derived from that timestamp. When omitted, the current server time is used (meaning the schedule for today is returned). The `date` field in the response entry echoes back the raw timestamp that was used — which is midnight only when `YYYY-MM-DD` was supplied; otherwise it is the exact timestamp provided or the current wall-clock time.

**`includeReferences`** — Controls whether the `data.references` block is populated. Defaults to `true`. The block itself is always present in the response; when `false`, each of its lists (`agencies`, `routes`, `stops`, `trips`, `situations`, `stopTimes`) is returned empty rather than populated, and the batch lookups that would otherwise fill them are skipped.

**`version`** — API version selector. Defaults to `2`. Only version 2 behaviour is in scope for the Go reimplementation.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "code":        { "type": "integer" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "text":        { "type": "string" },
    "version":     { "type": "integer" },
    "data": {
      "type": "object",
      "properties": {
        "entry":      { "$ref": "#/$defs/entry" },
        "references": { "$ref": "#/$defs/references" }
      }
    }
  }
}
```

**`code`** — HTTP-mirrored status code. Always `200` on a successful response.

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`text`** — Human-readable status string, e.g. `"OK"`.

**`version`** — Effective API version (always `2`).

**`data.entry`** — The stop schedule payload; see below.

**`data.references`** — Referenced entities. Always present as an object; when `includeReferences=false`, every list within it is empty rather than the key being omitted. When references are included, `stops` contains the queried stop. `routes` contains the stop's serving routes using each route's combined ID, even if some or all of those routes have no active trips on the queried date; `agencies` contains the agencies for those route references. A stop that exists but has no serving routes therefore returns the queried stop reference with empty route and agency reference lists. [(service-date-independent stop routes)](https://github.com/OneBusAway/onebusaway-application-modules/blob/fed9494e4b3ff345efabcd17b9b21fbf2a9dab7e/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopBeanServiceImpl.java#L166-L189)

---

### data.entry

```json
{
  "type": "object",
  "properties": {
    "date":                { "type": "integer", "description": "Unix ms" },
    "stopId":              { "type": "string" },
    "stopRouteSchedules":  {
      "type": "array",
      "items": { "$ref": "#/$defs/stopRouteSchedule" }
    }
  }
}
```

**`data.entry.date`** — The timestamp used to determine the service date, in Unix milliseconds. When `date=YYYY-MM-DD` was supplied, this is midnight of that date in the server's local timezone. When a Unix ms integer was supplied, this is that integer. When the parameter was omitted, this is the current server time. See the note in Request Parameters for the inconsistency.

**`data.entry.stopId`** — The combined stop ID (`{agencyId}_{stopId}`) of the queried stop. A full stop record for this ID always appears in `data.references.stops`.

**`data.entry.stopRouteSchedules`** — One entry per route that has scheduled service at this stop on the queried date. Empty array when no trips are active (e.g., the date is outside the feed's validity period, or it is a day with no service). Ordered by route short name, with a natural-string sort (e.g., route `"9"` before `"10"`); falls back to long name, then combined route ID if the short name is absent.

---

### data.entry.stopRouteSchedules[]

```json
{
  "type": "object",
  "properties": {
    "routeId": { "type": "string" },
    "stopRouteDirectionSchedules": {
      "type": "array",
      "items": { "$ref": "#/$defs/stopRouteDirectionSchedule" }
    }
  }
}
```

**`stopRouteSchedules[].routeId`** — Combined route ID. A full route record for this ID always appears in `data.references.routes`.

**`stopRouteSchedules[].stopRouteDirectionSchedules`** — One entry per direction of travel on this route at this stop. Most stops are served in only one direction, but stops near a terminal may appear in both. Ordered alphabetically by the group's `tripHeadsign`.

---

### data.entry.stopRouteSchedules[].stopRouteDirectionSchedules[]

```json
{
  "type": "object",
  "properties": {
    "tripHeadsign": { "type": "string" },
    "scheduleStopTimes": {
      "type": "array",
      "items": { "$ref": "#/$defs/scheduleStopTime" }
    },
    "scheduleFrequencies": {
      "type": "array",
      "items": { "$ref": "#/$defs/scheduleFrequency" }
    }
  }
}
```

**`stopRouteDirectionSchedules[].tripHeadsign`** — The destination text displayed on vehicles in this direction group. Chosen as the plurality headsign across all trips in the group (for fixed-schedule trips, each trip contributes one vote; for frequency-based trips, each contributes votes proportional to the estimated number of runs in its interval). May be `null` when no headsign data is present in the feed.

**`stopRouteDirectionSchedules[].scheduleStopTimes`** — Fixed-schedule stop times sorted by departure time. Empty array when the direction has no fixed-schedule service on this date. In Maglev, this also includes entries synthesized from `exact_times=1` frequency trips — a deviation from Java; see [Implementation Decisions](#implementation-decisions).

**`stopRouteDirectionSchedules[].scheduleFrequencies`** — Frequency-based service windows sorted by start time. Empty array when the direction has no frequency-based service on this date. Frequency and fixed-schedule entries may coexist in the same direction group. In Java this includes both `exact_times=0` and `exact_times=1` trips undifferentiated; Maglev only places `exact_times=0` trips here — see [Implementation Decisions](#implementation-decisions).

---

### data.entry.stopRouteSchedules[].stopRouteDirectionSchedules[].scheduleStopTimes[]

```json
{
  "type": "object",
  "properties": {
    "arrivalTime":       { "type": "integer", "description": "Unix ms" },
    "departureTime":     { "type": "integer", "description": "Unix ms" },
    "arrivalEnabled":    { "type": "boolean" },
    "departureEnabled":  { "type": "boolean" },
    "serviceId":         { "type": "string" },
    "tripId":            { "type": "string" },
    "stopHeadsign":      { "type": "string" }
  }
}
```

**`scheduleStopTimes[].arrivalTime`** — Scheduled arrival time at this stop, in Unix milliseconds. Equal to `departureTime` when the GTFS feed does not separately record an arrival time.

**`scheduleStopTimes[].departureTime`** — Scheduled departure time at this stop, in Unix milliseconds. Stop times are sorted by this value within the direction group.

**`scheduleStopTimes[].arrivalEnabled`** — `true` for the overwhelming majority of stop times. `false` only when this stop is the first stop in the vehicle's entire block for the service day — meaning the vehicle starts its work here and there is no inbound arrival from a preceding trip.

**`scheduleStopTimes[].departureEnabled`** — `true` for the overwhelming majority of stop times. `false` only when this stop is the last stop in the vehicle's entire block for the service day — meaning the vehicle ends its work here and there is no onward departure.

**`scheduleStopTimes[].serviceId`** — Combined service calendar ID (`{agencyId}_{serviceId}`) identifying the GTFS service pattern that activates this trip on the queried date.

**`scheduleStopTimes[].tripId`** — Combined trip ID (`{agencyId}_{tripId}`). No full trip record appears in `data.references`; clients must use `/api/where/trip/{id}` for trip details.

**`scheduleStopTimes[].stopHeadsign`** — Per-stop headsign override from the GTFS `stop_times.stop_headsign` field. Always `""` in the current Java implementation due to the defect described in Suspected Defects.

---

### data.entry.stopRouteSchedules[].stopRouteDirectionSchedules[].scheduleFrequencies[]

```json
{
  "type": "object",
  "properties": {
    "serviceDate":       { "type": "integer", "description": "Unix ms" },
    "startTime":         { "type": "integer", "description": "Unix ms" },
    "endTime":           { "type": "integer", "description": "Unix ms" },
    "headway":           { "type": "integer" },
    "serviceId":         { "type": "string" },
    "tripId":            { "type": "string" },
    "stopHeadsign":      { "type": "string" },
    "arrivalEnabled":    { "type": "boolean" },
    "departureEnabled":  { "type": "boolean" }
  }
}
```

**`scheduleFrequencies[].serviceDate`** — The start of the service day in Unix milliseconds (midnight of the service date in the agency's timezone).

**`scheduleFrequencies[].startTime`** — The beginning of the frequency window in Unix milliseconds (absolute wall-clock time, not a relative offset). Vehicles depart approximately every `headway` seconds from this time until `endTime`.

**`scheduleFrequencies[].endTime`** — The end of the frequency window in Unix milliseconds.

**`scheduleFrequencies[].headway`** — Approximate seconds between successive departures within the frequency window.

**`scheduleFrequencies[].serviceId`** — Combined service calendar ID (`{agencyId}_{serviceId}`).

**`scheduleFrequencies[].tripId`** — Combined trip ID (`{agencyId}_{tripId}`) for the template trip that defines the stop sequence and relative timing.

**`scheduleFrequencies[].stopHeadsign`** — Per-stop headsign override, if present in the feed; otherwise `null` or absent.

**`scheduleFrequencies[].arrivalEnabled`** — `false` only when this stop is the first stop in the vehicle's block. Same semantics as in fixed-schedule stop times.

**`scheduleFrequencies[].departureEnabled`** — `false` only when this stop is the last stop in the vehicle's block. Same semantics as in fixed-schedule stop times.
