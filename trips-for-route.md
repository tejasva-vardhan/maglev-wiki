# trips-for-route

## Goal in Context

A rider's client application wants to show all currently active vehicle journeys on a given route — with real-time position and status for each vehicle and, optionally, the full stop-time schedule for each trip — so the rider can see when and where buses are running right now.

## Scope

OneBusAway REST API, `/api/where/trips-for-route/{id}`.

## Level

User goal.

## Primary Actor

Rider (accessing the system via a client application).

## Stakeholders and Interests

- **Rider** — wants up-to-date, accurate information about which vehicles are active on the route and where they are.

## Preconditions

- The route identified by `id` is served by this OBA instance.
- The caller supplies a valid combined route ID in the URL path.

## Minimal Guarantees

- The response is always `200 OK` with a JSON envelope (even for unknown route IDs and out-of-service-area conditions).
- The `data.limitExceeded` field is always `false`.

## Success Guarantees

- The response contains one entry per active block whose scheduled trips overlap the query window, with real-time status and/or schedule as requested.
- Trip beans for all returned entries appear in `data.references.trips`. **Maglev does not always uphold this** — a reference-lookup failure can omit a bean while the referencing ID remains in `data.list`; see Implementation Decisions.

## Trigger

A GET request to `/api/where/trips-for-route/{id}.json`.

## Main Success Scenario

1. The caller issues `GET /api/where/trips-for-route/{id}.json?key=…` with a combined route ID encoded in the path.
2. The server resolves the route ID and looks up all vehicle blocks that have at least one trip on that route active within the window spanning 30 minutes before and 10 minutes after the query time. ([`BlockStatusServiceImpl.java#L182-L188`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/blocks/BlockStatusServiceImpl.java#L182-L188))
3. For each matched block, the server identifies the trip the vehicle is currently executing (the active trip), and populates `data.list[].tripId`, `data.list[].schedule`, `data.list[].situationIds`, and `data.list[].status.activeTripId` all from that same active trip instance — the block was selected because *some* trip in it is on the queried route, but which trip is reported is not filtered to that route. Due to interlining, the active trip may therefore belong to a different route than the one queried, and `tripId` reflects that other-route trip just like `status.activeTripId` does; the two fields are always equal. ([`TripStatusBeanServiceImpl.java#L231-L236`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L231-L236), [`#L417-L429`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L417-L429), [`#L461-L491`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L461-L491)) **Maglev deviates from this** — see Implementation Decisions.
4. For each entry, the server optionally includes:
   - A trip bean in `data.references.trips` (controlled by `includeTrip`; default on).
   - A `schedule` object with the full stop-time sequence (controlled by `includeSchedule`; default on).
   - A `status` object with real-time vehicle position, phase, schedule deviation, and occupancy (controlled by `includeStatus`; default on).
5. The server responds with `200 OK`. The envelope contains the list, a `limitExceeded: false` flag, and a `references` block.

## Extensions

**2a. Route ID is unknown or has no active blocks in the window:**
- The server returns `200 OK` with `data.list: []` and `data.limitExceeded: false`. No error or 404 is returned.

**2b. Request is out of service area:**
- `OutOfServiceAreaServiceException` is caught; the server returns `200 OK` with an empty list (identical to 2a). ([`TripsForRouteAction.java#L121-L123`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripsForRouteAction.java#L121-L123))

**1c. `id` parameter missing from path:**
- The Struts2 required-field validator fires; the server returns `400 Bad Request` with a validation error envelope.

**5a. `includeTrip=false`:**
- Trip beans are not added to `data.references.trips`. The `tripId` field in each list entry still identifies the trip, but no trip detail is available in references.

**5b. `includeSchedule=false`:**
- Each list entry omits the `schedule` object entirely.

**5c. `includeStatus=false`:**
- Each list entry omits the `status` object entirely. Vehicle position, phase, deviation, and occupancy data are absent.

**5d. `includeReferences=false`:**
- The entire `data.references` block is returned empty (no trips, stops, routes, or agencies).

**3a. Canceled trip with hide-canceled-trips configured:**
- If the server is configured to suppress canceled trips and a block's active trip is canceled, the status sub-object is omitted from that entry even when `includeStatus=true`. The entry still appears in the list (the trip is not removed).

**Ordering note:** Results are collected into a hash set during block resolution, so the order of entries in `data.list` is non-deterministic across requests. ([`BlockCalendarServiceImpl.java#L209-L221`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/blocks/BlockCalendarServiceImpl.java#L209-L221))

---

## Suspected Defects

### Defects that affect the use case

**`maxCount` parameter is accepted but silently ignored.**
`TripsForRouteAction` accepts a `maxCount` query parameter, validates it, and writes it into the query bean ([`TripsForRouteAction.java#L78-L79, 109`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripsForRouteAction.java#L78-L79)). However, `TripStatusBeanServiceImpl.getTripsForRoute` never reads `query.getMaxCount()` ([`TripStatusBeanServiceImpl.java#L231-L237`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L231-L237)). All matching trips are always returned regardless of `maxCount`. This describes Java's behaviour only — Maglev does not replicate even the validation half of it; see Implementation Decisions.

**`data.limitExceeded` is always `false`.**
The `ListBean` returned by `getBlockLocationsAsTripDetails` is always constructed with `limitExceeded = false` ([`TripStatusBeanServiceImpl.java#L428`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L428)). Because `maxCount` is also never applied, there is no scenario in which this field can be `true`. The field is present in the response but conveys no information.

### Implementation defects only

**Wrong variable in null guard causes null entries in the intermediate list.**
In `TripStatusBeanServiceImpl.getBlockLocationsAsTripDetails`, the null check reads `if (tripDetails != null)` where `tripDetails` is the accumulator list (never null). The intent is `if (details != null)` ([`TripStatusBeanServiceImpl.java#L424`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L424)). As a result, null beans can accumulate in the intermediate list. `BeanFactoryV2.getTripDetailsResponse` compensates with its own null check ([`BeanFactoryV2.java#L327-L329`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L327-L329)), so no null entries reach the serialized response.

**`lastKnownLocation` is set twice in `getBlockLocationAsStatusBean`.**
Lines 270 and 276 of `TripStatusBeanServiceImpl` call `bean.setLastKnownLocation(blockLocation.getLastKnownLocation())` with the same value ([`TripStatusBeanServiceImpl.java#L270-L276`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L270-L276)). The second call is redundant and has no observable effect.

---

## Implementation Decisions

**`maxCount` is not implemented for `trips-for-route` — no validation, no enforcement (see DC-5b).**
Unlike the endpoints that do support it (`stops-for-location`, `routes-for-location`, `route-search`, `search-stops`), Maglev does not process `maxCount` here at all: any value, including non-numeric input, is silently accepted and has no effect. This is a deliberate decision, not an oversight, and it goes further than replicating the Suspected Defect above — Java at least validates the parameter's numeric format (rejecting non-numeric values with `400`); Maglev does not even do that.

The reasoning, from #1210 / PR #1223:

- **The result set is not open-ended.** Unlike `stops-for-location` or `route-search`, `data.list` here is bounded by how many vehicles a route can have concurrently in service within the ±30/+10 minute query window — never large enough to need capping.
- **There is nothing meaningful to keep when truncating.** Per the Ordering note above, entries are collected in non-deterministic order, so a `maxCount`-bounded response would return an arbitrary subset rather than a useful "top N."
- **Truncation would actively mislead real clients.** Client code (e.g. Wayfinder's vehicle-marker sweep, Android's trip-extrapolation layer) treats an entry's presence in `data.list` as "this vehicle is currently active" and removes or ages out anything missing. Unlike ordinary pagination, a truncated response here would read as vehicles going offline that are still running.

If this is reconsidered in the future, matching Java's validate-only behaviour (reject non-numeric `maxCount`, silently accept and ignore everything else) is the minimum bar for parity — full enforcement (truncating results) is not recommended, for the reasons above.

---

**`tripId` is decoupled from `status.activeTripId` under interlining — `schedule` and `situationIds` follow `tripId`.**
In Java, `data.list[].tripId`, `data.list[].status.activeTripId`, `data.list[].schedule`, and `data.list[].situationIds` are all populated from the exact same active-trip instance, with no filtering by the queried route ([`TripStatusBeanServiceImpl.java#L231-L236`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L231-L236), [`#L417-L429`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L417-L429), [`#L461-L491`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L461-L491), [`#L283-L295`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L283-L295)). This conflation is not a documented defect — it's simply how Java is designed — so it does not appear in Suspected Defects above.

Maglev deliberately diverges (#1254 / PR #1256): when a block's active trip is interlined onto a different route than the one queried, `tripId` (and the `schedule`/`situationIds` built from it) resolve to the queried-route trip that caused the block to be selected — the trip on the requested route whose scheduled time window is nearest the active trip's — while `status.activeTripId` continues to reflect whatever trip the vehicle is actually running, exactly as Java does.

The reasoning:

- **The queried-route trip is genuinely useful information that Java discards.** A client that asked for route A's active vehicles has to already know route A's trip to make sense of an entry; reporting the other route's trip in `tripId` under interlining tells it nothing about the route it asked about.
- **No surveyed client relies on the two fields being equal.** iOS, the JS SDK, and Wayfinder don't read `tripId` from this endpoint at all. Android's own trip-extrapolation layer already treats `tripId`, `status.activeTripId`, and `schedule` as independent, not-safe-to-assume-consistent values — it keys off `status.activeTripId` and fetches schedule/route data separately for interlined continuations, citing prior interlining bugs in its own code. Maglev's decoupling is consistent with how the most sophisticated existing client already defends against exactly this ambiguity.

Trade-offs to be aware of:

- **`schedule` and `status` can now describe two different trips in the same entry.** `schedule.stopTimes` belongs to the resolved queried-route trip (potentially scheduled hours from the query time), while `status.position`/`distanceAlongTrip`/etc. describe the actively-running trip. No response field signals this split to a client rendering both together.
- **If no queried-route trip can be found anywhere in the block** (e.g. the block's only queried-route trip runs under a service_id that isn't active today or yesterday), Maglev falls back to Java's conflated behaviour for that one entry — `tripId` equals `status.activeTripId` — rather than omitting the entry, preserving the Success Guarantee of one entry per active block.

If this is reconsidered in the future, the fallback is Java's original behaviour: drop the interlining resolution and always set `tripId` equal to `status.activeTripId`.

---

**Reference-trip/route lookup failures inside `buildTripReferences` degrade to a silently incomplete response — Maglev does not fail loud like Java does.**
In Java, `TripsForRouteAction.show()` wraps `_service.getTripsForRoute(query)` and `factory.getTripDetailsResponse(trips)` in a `try` block that only catches `OutOfServiceAreaServiceException`; any other exception raised while building the response — including one from bean/reference construction — propagates uncaught, which Struts2 turns into a `500` ([`TripsForRouteAction.java#L118-123`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripsForRouteAction.java#L118-L123)). Java also never fetches reference beans independently of the entities it's already resolved into `data.list` (see Main Success Scenario and the `data.references` field above), so it has no code path where reference-building can fail without failing the whole request.

Maglev's `buildTripReferences` does a separate batched DB round-trip (`GetTripsByIDs`, then `GetRoutesByIDs`, then per-route agency lookups) to hydrate trip/route/agency beans for IDs referenced via `tripId`, `schedule.previousTripId`/`nextTripId`, and `status.activeTripId` that weren't already fetched earlier in the request. If any of these queries fails, the handler logs the error and continues, returning `200 OK` with the affected bean(s) simply missing from `data.references` — even when the missing bean belongs to an entry's own `tripId`, which the Success Guarantees above otherwise claim is always resolvable.

This is a deliberate, existing posture, not something introduced by any single change: commit `3dd2995e` removed a dead-code path that had `404`'d the entire response on a single interlined block's unresolved trip, on the grounds that "the trips-for-route spec guarantees 200 OK even for unknown routes and out-of-service-area conditions, with no error or 404 case defined," and that a hard failure would discard every other correctly-resolved entry in the same response over one bad lookup. This reasoning was reaffirmed during review of PR #1288, which made the reference-ID tracking in `buildTripReferences` more explicit but deliberately left this error posture untouched.

Trade-offs:
- A reference-fetch failure can produce a response where `data.list[]` cites a trip ID (via `tripId`, `schedule.previousTripId`/`nextTripId`, or `status.activeTripId`) with no corresponding bean in `data.references` — something Java's design cannot produce, since Java only ever references a bean it already holds in memory.
- Clients that assume every referenced ID always resolves in `data.references` (rather than treating the lookup as optional) can hit a missing key. Known clients already treat this defensively: Wayfinder's `activeTripId` lookup simply skips rendering that vehicle marker on a miss, and Android's own `?:`/`mapNotNull` fallbacks plus its independent re-fetch of interlined trips over a separate network call insulate it from this exact failure mode. No client is currently known to fail hard on it.

If this is reconsidered in the future, the fallback is to propagate reference-fetch errors as `500` (matching Java's fail-loud posture for unexpected errors), following the same `errors.Is(err, sql.ErrNoRows)` → `404` / other → `500` split CONTRIBUTING.md prescribes for handlers generally — which would mean reversing this handler's specific, deliberate carve-out from that rule.

---

## Open Questions

None. All behaviours were resolved by static analysis of the call chain and confirmed against the running server.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["id"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Combined route ID encoded in the URL path, e.g. /api/where/trips-for-route/40_100232.json"
    },
    "time": {
      "type": "integer",
      "description": "Unix ms — query time. Defaults to current server time if omitted or 0."
    },
    "maxCount": {
      "type": "integer",
      "description": "In Java: accepted and validated (non-numeric input returns 400) but never applied to result count — a Java defect; see Suspected Defects. Maglev's behaviour deviates further; see Implementation Decisions."
    },
    "includeTrip": {
      "type": "boolean",
      "default": true,
      "description": "Whether to populate trip beans in data.references.trips."
    },
    "includeStatus": {
      "type": "boolean",
      "default": true,
      "description": "Whether to include the status sub-object in each list entry."
    },
    "includeSchedule": {
      "type": "boolean",
      "default": true,
      "description": "Whether to include the schedule sub-object in each list entry."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "Whether to populate data.references. Pass false to omit the entire references block."
    },
    "key": {
      "type": "string",
      "description": "API key."
    }
  }
}
```

**`id`** — Combined route ID of the form `{agencyId}_{routeId}`, e.g. `40_100232`. Encoded directly in the URL path before the format suffix.

**`time`** — Query time in Unix milliseconds. The server selects blocks active in the 40-minute window centred 10 minutes after this time (30 minutes before to 10 minutes after). Defaults to the current server time.

**`maxCount`** — Integer limit on the number of results. In Java, accepted and validated (non-numeric input returns `400`) but never applied; all matching trips are always returned regardless of value — see Suspected Defects. Maglev does not implement this parameter at all, including the validation; see Implementation Decisions.

**`includeTrip`** — When `true` (default), each matched trip's bean is added to `data.references.trips`. When `false`, the trip ID is still present in the list entry but no detail is available in references.

**`includeStatus`** — When `true` (default), each list entry includes a `status` object with real-time vehicle position, phase, schedule deviation, and occupancy.

**`includeSchedule`** — When `true` (default), each list entry includes a `schedule` object with the full ordered stop-time sequence for the trip.

**`includeReferences`** — When `false`, the `data.references` block is returned empty. Default `true`.

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

**`code`** — HTTP status code mirrored in the body; `200` on success.

**`text`** — Human-readable status string, e.g. `"OK"`.

**`version`** — API version; always `2`.

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`data`** — Container for the list and references.

---

### data

```json
{
  "type": "object",
  "properties": {
    "limitExceeded": { "type": "boolean" },
    "list":          { "type": "array" },
    "references":    { "type": "object" }
  }
}
```

**`data.limitExceeded`** — Always `false`; see Suspected Defects.

**`data.list`** — Array of `tripDetails` entries, one per active block. Order is non-deterministic.

**`data.references`** — Keyed collections of referenced entities: `trips`, `routes`, `agencies`, `stops`, `situations`. Populated from all entities referenced within the list entries. In Java, an entity only ever appears in `data.list` if a bean for it already exists in memory, so every referenced ID always resolves in `data.references`. **Maglev deviates from this**: reference beans not already on hand are fetched from the database separately, and a failed fetch is logged and skipped rather than failing the response, which can leave a referenced ID (`tripId`, `schedule.previousTripId`/`nextTripId`, `status.activeTripId`) with no matching entry here — see Implementation Decisions.

---

### data.list[]

```json
{
  "type": "object",
  "properties": {
    "tripId":       { "type": "string" },
    "serviceDate":  { "type": "integer", "description": "Unix ms" },
    "frequency":    { "type": "object" },
    "situationIds": { "type": "array",  "items": { "type": "string" } },
    "schedule":     { "type": "object" },
    "status":       { "type": "object" }
  }
}
```

**`data.list[].tripId`** — Combined ID of the trip the vehicle is currently executing. In Java this is always equal to `status.activeTripId`, regardless of whether that trip is on the queried route — see Main Success Scenario step 3. **Maglev deviates from this**: `tripId` resolves to the queried-route trip that caused the block to be selected, which may differ from `status.activeTripId`; see Implementation Decisions.

**`data.list[].serviceDate`** — Unix millisecond timestamp of midnight at the start of the service day for this trip. For trips that extend past midnight, the service date is the previous calendar day: the block search resolves candidate service dates against the block's scheduled time range (which can exceed 24:00:00), not against the calendar date of the query time. [Source: [`BlockStatusServiceImpl.java#L180-L189`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/blocks/BlockStatusServiceImpl.java#L180-L189), [`BlockCalendarServiceImpl.java#L247-L265`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/blocks/BlockCalendarServiceImpl.java#L247-L265)]

**`data.list[].frequency`** — Frequency descriptor for headway-based service; `null` for timetable-based trips.

**`data.list[].situationIds`** — IDs of active service alerts applicable to this trip. Empty array if none. Full alert details appear in `data.references.situations`. In Java "this trip" is always the active trip (same as `status.activeTripId`); under Maglev's interlining deviation, it follows `tripId` instead — see Implementation Decisions.

**`data.list[].schedule`** — Present only when `includeSchedule=true`. See `data.list[].schedule` below.

**`data.list[].status`** — Present only when `includeStatus=true`. See `data.list[].status` below.

---

### data.list[].schedule

```json
{
  "type": "object",
  "properties": {
    "timeZone":      { "type": "string" },
    "stopTimes":     { "type": "array" },
    "previousTripId":{ "type": "string" },
    "nextTripId":    { "type": "string" },
    "frequency":     { "type": "object" }
  }
}
```

**`data.list[].schedule.timeZone`** — IANA timezone identifier for the agency operating this trip, e.g. `"America/Los_Angeles"`.

**`data.list[].schedule.stopTimes`** — Ordered array of scheduled stop times for this trip. See `data.list[].schedule.stopTimes[]` below. In Java "this trip" is always the active trip (same as `status.activeTripId`), so `schedule` and `status` never describe two different trips. Under Maglev's interlining deviation, `schedule` (and `previousTripId`/`nextTripId` below) follow `tripId` instead of the active trip — see Implementation Decisions.

**`data.list[].schedule.previousTripId`** — Combined ID of the preceding trip in this vehicle's block (i.e. the trip the same vehicle ran immediately before), if any. Present when the vehicle has a prior trip in its block for this service day. The corresponding trip bean may be missing from `data.references.trips` if Maglev's reference lookup for it failed; see Implementation Decisions.

**`data.list[].schedule.nextTripId`** — Combined ID of the following trip in this vehicle's block (i.e. the trip the same vehicle will run immediately after), if any. Same reference-lookup-failure caveat as `previousTripId` above; see Implementation Decisions.

**`data.list[].schedule.frequency`** — Frequency descriptor for headway-based service; `null` for timetable trips.

---

### data.list[].schedule.stopTimes[]

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

**`data.list[].schedule.stopTimes[].arrivalTime`** — Scheduled arrival time in seconds elapsed since midnight of the service date. Values may exceed 86 400 for trips running past midnight.

**`data.list[].schedule.stopTimes[].departureTime`** — Scheduled departure time, same units as `arrivalTime`.

**`data.list[].schedule.stopTimes[].stopId`** — Combined stop ID. Full stop detail appears in `data.references.stops`.

**`data.list[].schedule.stopTimes[].stopHeadsign`** — Destination text displayed at this specific stop, if it differs from the trip headsign. Empty string when not overridden.

**`data.list[].schedule.stopTimes[].distanceAlongTrip`** — Distance in metres from the first stop of the trip to this stop, measured along the vehicle's path.

**`data.list[].schedule.stopTimes[].historicalOccupancy`** — Historical average occupancy at this stop, expressed as a GTFS-RT `OccupancyStatus` name (e.g. `"MANY_SEATS_AVAILABLE"`). Empty string when no historical data is available.

---

### data.list[].status

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
    "position": {
      "type": "object",
      "properties": {
        "lat": { "type": "number" },
        "lon": { "type": "number" }
      }
    },
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
    "lastKnownLocation": {
      "type": "object",
      "properties": {
        "lat": { "type": "number" },
        "lon": { "type": "number" }
      }
    },
    "lastKnownDistanceAlongTrip": { "type": "number" },
    "lastKnownOrientation":       { "type": "number" },
    "scheduleDeviation":          { "type": "integer" },
    "distanceAlongTrip":          { "type": "number" },
    "vehicleId":                  { "type": "string" },
    "occupancyStatus":            { "type": "string" },
    "occupancyCount":             { "type": "integer" },
    "occupancyCapacity":          { "type": "integer" },
    "vehicleFeatures":            { "type": "array", "items": { "type": "string" } },
    "situationIds":               { "type": "array", "items": { "type": "string" } }
  }
}
```

**`data.list[].status.activeTripId`** — Combined ID of the trip the vehicle is currently executing. In Java this is always equal to `tripId` — both are populated from the same active trip instance. ([`TripStatusBeanServiceImpl.java#L283-L295`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripStatusBeanServiceImpl.java#L283-L295)) **Maglev deviates from this**: `activeTripId` always reflects the vehicle's actual running trip, and may differ from `tripId` when the vehicle is interlining on an adjacent block trip that belongs to a different route — see Implementation Decisions. Its trip bean may also be missing from `data.references.trips` if Maglev's reference lookup for it failed — see Implementation Decisions.

**`data.list[].status.blockTripSequence`** — Zero-based index of the active trip within the vehicle's block configuration. Useful for comparing against the same field in arrival-and-departure responses.

**`data.list[].status.serviceDate`** — Unix millisecond timestamp of midnight for the active trip's service date. Duplicates `data.list[].serviceDate`, including the previous-calendar-day behaviour for trips extending past midnight.

**`data.list[].status.frequency`** — Frequency descriptor for headway-based service; `null` for timetable trips.

**`data.list[].status.scheduledDistanceAlongTrip`** — Distance in metres the vehicle is scheduled to have travelled along the active trip at the query time.

**`data.list[].status.totalDistanceAlongTrip`** — Total length of the active trip in metres.

**`data.list[].status.position`** — Current geographic position of the vehicle. Uses real-time GPS data when available (`predicted=true`); otherwise extrapolated from the schedule. Absent if not determinable.

**`data.list[].status.orientation`** — Vehicle heading in degrees, where 0° is east, 90° is north, 180° is west, 270° is south. Absent if not available.

**`data.list[].status.closestStop`** — Combined ID of the stop on the active trip's stop sequence that is nearest to the vehicle's current position.

**`data.list[].status.closestStopTimeOffset`** — Seconds between the vehicle's current position and the closest stop's scheduled time. Positive means the stop is ahead; negative means it has been passed.

**`data.list[].status.nextStop`** — Combined ID of the next stop the vehicle has not yet passed. Absent once the vehicle has passed the last stop in the trip.

**`data.list[].status.nextStopTimeOffset`** — Seconds until the vehicle reaches `nextStop` according to the schedule.

**`data.list[].status.phase`** — Current phase of the vehicle's journey. Possible values: `at_base`, `deadhead_before`, `layover_before`, `in_progress`, `deadhead_during`, `layover_during`, `deadhead_after`, `layover_after`. Empty string when not available.

**`data.list[].status.status`** — Status modifier for the trip. `"default"` under normal conditions; `"canceled"` for a canceled trip (when the server is not configured to hide canceled trips).

**`data.list[].status.predicted`** — `true` if real-time data was used to compute position and deviation; `false` if the position is derived from the static schedule.

**`data.list[].status.lastUpdateTime`** — Unix millisecond timestamp of the most recent real-time update received from the vehicle. `0` if no update has been received.

**`data.list[].status.lastLocationUpdateTime`** — Unix millisecond timestamp of the most recent real-time update that contained a location. `0` if no location update has been received.

**`data.list[].status.lastKnownLocation`** — Last confirmed GPS location of the vehicle. Unlike `position`, this is not extrapolated forward. Absent if no real-time location has ever been received.

**`data.list[].status.lastKnownDistanceAlongTrip`** — Last confirmed distance along the active trip in metres, as reported by the vehicle. Absent if not available.

**`data.list[].status.lastKnownOrientation`** — Last confirmed heading in degrees. Absent if not available.

**`data.list[].status.scheduleDeviation`** — Schedule deviation in seconds. Positive means the vehicle is running late; negative means early. `0` when `predicted=false`.

**`data.list[].status.distanceAlongTrip`** — Distance in metres the vehicle has actually travelled along the active trip. May be extrapolated from the last known reading. Absent if not available.

**`data.list[].status.vehicleId`** — Combined vehicle ID. Empty string when no real-time data is available.

**`data.list[].status.occupancyStatus`** — Passenger load level, expressed as a GTFS-RT `OccupancyStatus` name: `EMPTY`, `MANY_SEATS_AVAILABLE`, `FEW_SEATS_AVAILABLE`, `STANDING_ROOM_ONLY`, `CRUSHED_STANDING_ROOM_ONLY`, `FULL`, or `NOT_ACCEPTING_PASSENGERS`. Empty string when no occupancy data is available.

**`data.list[].status.occupancyCount`** — Raw passenger count from the vehicle, if reported. `-1` when not available.

**`data.list[].status.occupancyCapacity`** — Vehicle capacity, if reported. `-1` when not available.

**`data.list[].status.vehicleFeatures`** — Array of feature strings reported by the vehicle (e.g. accessibility equipment). Empty array when none reported.

**`data.list[].status.situationIds`** — IDs of active service alerts applying to this vehicle journey. Empty array if none.

---

### data.list[].frequency (when present)

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

**`data.list[].frequency.startTime`** — Start of the frequency window in Unix milliseconds (absolute, not relative to service date).

**`data.list[].frequency.endTime`** — End of the frequency window in Unix milliseconds.

**`data.list[].frequency.headway`** — Seconds between successive departures during this window.

**`data.list[].frequency.exactTimes`** — `1` if departures occur at exact multiples of the headway from `startTime`; `0` for approximate headway-based service.
