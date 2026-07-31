# OBA API Legacy Defects

## Purpose & Scope

The OBA API has a number of documented defects in its legacy Java implementation. Maglev is the new Go implementation of the same API. For each defect class we must decide: does Maglev **fix** the defect (returning correct behaviour), or does it **preserve** it (returning the same broken behaviour as the legacy)?

This is not a straightforward "fix it" question. Some clients may have adapted their code to work around broken behaviour. Fixing a defect can break those clients in ways that are disruptive to end users. Conversely, preserving a defect in a new implementation perpetuates technical debt and may prevent future improvements.

This document covers only defects that are **observable by API callers** — defects that affect HTTP status codes, response bodies, or response fields. Pure implementation defects with no observable effect on callers are excluded.

## How to Run This Session

Work through the defect classes in order, top to bottom. For each class:

1. Read the **observed behaviour** and **intended behaviour** aloud.
2. Review the **client impact** analysis.
3. Where Maglev has already recorded a decision, note it and move on — these are not re-opened unless the group has a strong reason.
4. Where no decision exists, discuss and record the outcome in the **Decision** and **Notes** fields before moving on.

## Decision Criteria

When no prior decision exists, apply this rubric:

**Fix** when well-written clients (those following the API contract, not relying on broken behaviour) are unaffected or benefit; or when the broken behaviour actively harms clients (silent data loss, security leak, success indistinguishable from error).

**Preserve** when fixing would break clients that have adapted to the broken response in a way that is disruptive to end users — triggering error dialogs, breaking navigation flows, or where the broken behaviour has become the de-facto contract.

**Conditional fix** when a compatibility flag or graceful migration path can serve both client populations.

**Dual-implementation note:** The legacy implementation will remain in service alongside Maglev after release. Clients targeting both must handle both behaviours simultaneously. A fix in Maglev therefore adds complexity for client authors who cannot simply adopt the corrected behaviour — they must continue supporting the legacy behaviour as well. Where noted in individual client impact sections, this can make a fix that appears minor in isolation more burdensome in practice.

---

## DC-1: Unknown entity or malformed ID returns HTTP 200 with null body

The most pervasive defect class in the OBA API. When an endpoint cannot fulfil a request — because the requested entity does not exist, or because the ID supplied is syntactically malformed — it should return an appropriate HTTP error code (404 or 400). Instead, a defect in the exception handling infrastructure causes many endpoints to return HTTP 200 with a literal `null` body. Several endpoints have already been addressed and are recorded as Implementation Decistions in the relavent specs.

**Root cause:** Two related mechanisms produce the same symptom. For *unknown entities*: `ExceptionInterceptor` either does not handle `NoSuchAgencyServiceException` at all (leaving the response bean unset), or it handles the exception but sets the HTTP status only on a `DefaultHttpHeaders` instance that is never applied to the actual HTTP response — in both cases the framework serialises `null` with HTTP 200. For *malformed IDs* (no underscore): `AgencyAndIdLibrary.convertFromString` throws `IllegalStateException`, which escapes the action method uncaught and produces the same HTTP 200 null result.

**Observed behaviour:** HTTP 200 with a literal 4-byte `null` body, regardless of whether the resource was not found or the request was syntactically invalid.

**Intended behaviour:** HTTP 404 with a structured OBA error envelope (`code: 404`, `text`, no `data`) for unknown entities; HTTP 400 with `code: 400` for malformed IDs.

**Client impact if fixed:** Clients that detect "not found" by checking for a null body will no longer get that signal — the body will be a 404 error envelope. Their null check won't fire, and if they then try to process the response as a success payload they will fail. Additionally, if their HTTP library throws on non-200 status codes by default, that exception may now surface as a user-visible error where previously the null body was handled silently.

Clients targeting both implementations face a particular burden here: they cannot simply switch to status-code checking, since legacy returns HTTP 200 with null body while Maglev returns 4xx. They must handle both signals — checking for null body (legacy) and for a 4xx status code (Maglev) — for the same logical error condition.

**Prior Maglev decisions (already resolved — for reference only):**

The following endpoints already have an Implementation Decisions entry stating Maglev corrects this defect:

- [[stop]] — unknown stop ID → HTTP 404; malformed stop ID → HTTP 400
- [[stops-for-agency]] — unknown agency ID → HTTP 404
- [[stop-ids-for-agency]] — unknown agency ID → HTTP 404
- [[routes-for-agency]] — unknown agency ID → HTTP 404
- [[route-ids-for-agency]] — unknown agency ID → HTTP 404
- [[schedule-for-stop]] — unknown stop ID → HTTP 404
- [[arrival-and-departure-for-stop]] — unknown stop ID or trip ID → HTTP 404

**Endpoints with no Maglev decision yet:**

The three malformed-ID cases below are the same defect pattern as the already-decided endpoints above. The fix in each case would be HTTP 400, consistent with the treatment of malformed IDs elsewhere.

- [[stops-for-route]] — a malformed route ID (no underscore) returns HTTP 200 null. Note that a *valid but unknown* route ID on this endpoint already correctly returns HTTP 404, making the malformed-ID case an inconsistency within the same endpoint.
- [[trip]] — a malformed trip ID (no underscore) returns HTTP 200 null.
- [[shape]] — a malformed shape ID (no underscore) returns HTTP 200 null.

The following case is different in nature — it is not an ID parsing issue but a parameter interaction that triggers an NPE:

- [[search-stop]] — passing `includeReferences=false` triggers a NullPointerException before the response bean is set, returning HTTP 200 null. Any client passing this parameter is already receiving a broken null response and cannot be relying on it working correctly. Fixing it makes the parameter work for the first time rather than changing existing working behaviour.

**Decision:** [ ] Preserve  [x] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

A client would have to be sending requests with unknown entity ids to trigger this.

---

## DC-2: Wrong status code for "no service" conditions

[[schedule-for-route]] can encounter two distinct conditions where no schedule data can be returned: the requested date falls outside the route's service period entirely (`ServiceDateOutOfRange`), or the route exists and the date is valid but no trips happen to run that day (`NoServiceThatDay`). Both conditions are currently signalled with `code: 510` in the response body — HTTP 510 ("Not Extended") is a protocol-level negotiation code with no relation to transit scheduling, and is not propagated to the transport layer in any case. This defect is about choosing and correctly returning an appropriate status code for these conditions.

**Root cause:** The legacy implementation uses `code: 510` in the response body for two "no service" conditions on [[schedule-for-route]]. HTTP 510 means "Not Extended" — a protocol-level negotiation status with no relation to transit scheduling. Additionally, even this wrong code is not propagated to the transport layer: `setNoServiceResponse` and `setNoServiceThatDayResponse` return `new DefaultHttpHeaders()` without calling `.withStatus(510)`, so the HTTP response is always 200 regardless.

**Affected endpoint:** [[schedule-for-route]] — two distinct conditions:
- `ServiceDateOutOfRange`: the requested date is outside the valid service period for the route
- `NoServiceThatDay`: the route exists and the date is in range, but no trips run on that day

**Observed behaviour:** HTTP 200 at the transport layer, with `code: 510` and `text: "ServiceDateOutOfRange"` or `"NoServiceThatDay"` in the body.

**Intended behaviour:** A semantically appropriate status code, correctly propagated to the transport layer. Candidates: HTTP 404 (nothing found for this date) or HTTP 200 with an empty/annotated response body (the schedule exists but has no service). The decision here is what the right code should be, not just whether to propagate the existing one.

**Note on possible intent:** 510 is such an unusual choice that it may have been deliberate — perhaps as a distinctive application-level signal that clients could detect without ambiguity, even if the choice is semantically wrong by HTTP standards. If any clients have been written to specifically check for `code: 510`, fixing this would break them.

**Client impact if fixed:** Clients that inspect `code: 510` or `text` in the body to detect these conditions would need updating if the code changes. Clients that treat HTTP 200 as success and branch on `code` in the body would be affected by any transport-layer status change. Clients written against a corrected status code would need no changes once migrated.

Clients targeting both implementations must detect "no service" conditions via two different signals simultaneously: `code: 510` in the body (legacy) and whatever status code Maglev adopts.

**Decision:** [ ] Preserve  [x] Fix  [ ] Conditional fix

**Notes:** Both cases return HTTP 200 with `code: 200` in the body. The `text` field carries `"ServiceDateOutOfRange"` or `"NoServiceThatDay"` so clients that care about the reason can still distinguish them. These are not error conditions — the route exists, the request succeeded, the schedule for that date is simply empty. HTTP 404 was considered and rejected: a 404 means the resource doesn't exist, but the route is real in both cases — the caller just picked a date with no service. Returning 200 avoids conflating "no service today" with "route not found," which require different user-facing messages. Static analysis of all three clients (Wayfinder, iOS, Android) confirms none inspect `code: 510` or the `text` field for this endpoint; the change is invisible to existing clients.

---

## DC-3: `NoServiceThatDay` incorrectly includes a partial data body

**Note:** DC-3 is tightly coupled to DC-2 and should be decided alongside it. The status code question is addressed there.

As covered in DC-2, both `ServiceDateOutOfRange` and `NoServiceThatDay` are error conditions on [[schedule-for-route]]. Whatever status code is chosen in DC-2, this defect concerns what should be in the response body. The two conditions currently behave differently: `ServiceDateOutOfRange` returns no data body, while `NoServiceThatDay` returns a partial route schedule. Since both are error conditions, neither should probably include data.

**Root cause:** `NoServiceThatDay` is an error condition — the route exists but no trips run on the requested day. As an error response it probably should carry no data body, consistent with `ServiceDateOutOfRange` which returns `data: null`. Instead, `ScheduleForRouteAction` populates a partial route schedule and includes it in the response.

**Affected endpoint:** [[schedule-for-route]]

**Observed behaviour:** `ServiceDateOutOfRange` returns `data: null`; `NoServiceThatDay` returns a partial route schedule in `data`.

**Intended behaviour:** Both are error conditions and probably should return no data body. `ServiceDateOutOfRange` already does this correctly.

**Client impact if fixed:** Clients that read the partial data body from `NoServiceThatDay` responses — for example, to display route information even when no service runs — would stop receiving that data. The asymmetry between the two conditions also means clients may currently distinguish them by checking whether `data` is null; fixing this would remove that signal.

**Decision:** [ ] Preserve  [x] Fix  [ ] Conditional fix

**Notes:** Resolved by the DC-2 decision. Since both cases now return HTTP 200 with `code: 200`, they are no longer error responses — they are successful responses with an empty schedule. Both cases include the same partial body: route ID, schedule date, empty `serviceIds` and `stopTripGroupings` arrays, and agency+route references. This makes the two responses structurally consistent (the only difference is the `text` label) and gives clients useful context (route metadata) even when no trips run. `ServiceDateOutOfRange` gains the partial body it previously lacked; `NoServiceThatDay` keeps the one it already had.

---

## DC-4: Required parameter absence returns wrong status or body format

Two endpoints produce incorrect error responses when a required parameter is absent.

### DC-4a: Missing `input` in search endpoints returns HTTP 404 with non-OBA body

**Root cause:** [[search-route]] and [[search-stop]] both extend `ApiSearchAction`, which applies Struts2's `@RequiredFieldValidator` on the `input` field. This interceptor fires before the action method runs and returns Struts2's own validation error format.

**Observed behaviour:** HTTP 404 with body `{"fieldErrors":{"input":["missing input"]}}` — no `code`, `version`, `currentTime`, or `text` fields.

**Intended behaviour:** HTTP 400 in the standard OBA error envelope.

**Client impact if fixed:** Clients whose error handling branches on HTTP status code would be affected — code currently reaching the 404 handler would now reach the 400 handler instead. Additionally, clients parsing the Struts2 `fieldErrors` body would break entirely as the body format changes to the standard OBA envelope.

Clients targeting both implementations must parse two completely different body formats for the same error — Struts2 `fieldErrors` (legacy) and the standard OBA envelope (Maglev) — making any error handling for this case significantly more complex.

**Decision (DC-4a):** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

## DC-5: `maxCount` parameter accepted but silently ignored; `limitExceeded` always false

Several OBA endpoints accept a `maxCount` parameter intended to cap the number of results returned, paired with a `limitExceeded` boolean in the response to signal when the full result set was larger than the cap. On the endpoints below, the service layer never reads `maxCount` from the query, so the full result set is always returned and `limitExceeded` is always `false`.

**Root cause:** The service methods underlying these endpoints accept a query object containing `maxCount` but never read it. The response list is always the full result set; `limitExceeded` is always `false`.

**Affected endpoints:**
- [[agencies-with-coverage]] — `AgenciesWithCoverageAction` passes `maxCount` into the query but the response factory ignores it
- [[trips-for-route]] — `TripStatusBeanServiceImpl.getTripsForRoute` never reads `query.getMaxCount()`
- [[trips-for-location]] — same service method, same issue

**Observed behaviour:** All matching results are returned regardless of `maxCount`; `data.limitExceeded` is always `false`.

**Intended behaviour:** Results capped at `maxCount`; `limitExceeded: true` when the cap is hit.

**Client impact if fixed:** Clients that pass `maxCount` expecting a truncated result and currently receive the full set would start getting fewer results. For agencies-with-coverage most deployments have few agencies so `maxCount` rarely matters in practice; for the trips endpoints, response sizes can be large.

Clients targeting both implementations that use `maxCount` for pagination or display capping may receive the full dataset from legacy and a truncated set from Maglev for the same query. Result sizes cannot be assumed consistent between servers.

**Decision:** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

## DC-6: Non-deterministic or incorrect result truncation

Related to DC-5, but distinct: on these endpoints the truncation mechanism or the `limitExceeded` flag itself behaves incorrectly rather than being ignored entirely. Three endpoints are affected, each with a different bug in how results are capped and signalled.

**Root cause (routes-for-location):** `BeanServiceSupport.checkLimitExceeded` randomly shuffles the list before discarding the tail when results exceed `maxCount`. The same codebase omits this shuffle on other endpoints, so routes-for-location is uniquely non-deterministic.

**Root cause (stops-for-location, routeType path):** When `routeType` is applied in the geospatial (non-query) path, `limitExceeded` is set to `true` when the post-filter count exceeds `maxCount`, but the list is not actually truncated before it is returned.

**Root cause (search/stop):** The `maxCount` cap is applied to the stop list *before* the route-type filter runs, so `limitExceeded` can be `true` even when the returned list is shorter than `maxCount`.

**Affected endpoints:**
- [[routes-for-location]] — non-deterministic truncation
- [[stops-for-location]] — `limitExceeded: true` but list not truncated (when using `routeType` without `query`)
- [[search-stop]] — `limitExceeded` reflects pre-filter count, not the size of the returned list

**Observed behaviour:** Repeated identical requests to `routes-for-location` return different subsets when truncation occurs. `stops-for-location` returns more results than `maxCount` permits. `search/stop` signals truncation even when the returned list is within the limit.

**Intended behaviour:** Truncation should be deterministic, accurate, and consistent: the list capped at `maxCount` and `limitExceeded` indicating whether additional results were discarded.

**Client impact if fixed:** Clients that issue repeated requests to `routes-for-location` to enumerate the full result set (exploiting non-determinism) would no longer be able to do so, but this is an anti-pattern. Clients paginating on `limitExceeded` from `stops-for-location` or `search/stop` would receive more accurate signals.

### DC-6a: routes-for-location — non-deterministic truncation

**Decision (DC-6a):** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

### DC-6b: stops-for-location — `limitExceeded` true but list not truncated

**Decision (DC-6b):** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

### DC-6c: search/stop — `limitExceeded` reflects pre-filter count

**Decision (DC-6c):** [x] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** Decided alongside [PR #1220](https://github.com/OneBusAway/maglev/pull/1220) ("Fix: Implement route-type exclusion for search-stops endpoint"), which implemented the DC-15 route-type/zero-route filter and, in doing so, kept the existing order of operations: the FTS5 prefix match is truncated to `maxCount` first, and the route-type filter runs on that already-truncated page, with `limitExceeded` set from the truncation step before filtering.

Fixing it would mean evaluating the filter against the *entire* set of raw prefix matches before truncating to `maxCount`, not just the current page — otherwise a filtered-out stop near the front of the match order still wrongly shrinks the returned page below `maxCount`. That in turn requires either removing the `LIMIT` at the FTS5 query layer (fetching and route/agency-joining every raw match for the prefix instead of just `maxCount + 1` rows) or replacing the single fetch with a windowed fetch-filter-repeat loop against the index. Both add real query cost or real code complexity to fix a discrepancy that's already fully documented in the `search-stop` spec (Extension 2b, `data.limitExceeded` description) as something callers must treat as a raw-match indicator rather than a strict page-completeness guarantee. No client currently calls `search/stop.json` (confirmed across Wayfinder, js-sdk, iOS, Android), so there is also no near-term client harm from preserving it.

Practical effect: unchanged from legacy — `data.limitExceeded: true` does not guarantee `data.list` has `maxCount` entries, and a response can be `data.list: []` with `data.limitExceeded: true` if every stop in the truncated page was subsequently filtered out.

---

## DC-7: `nullSafeShortName` exposed as a public response field

Route objects returned by several endpoints include an unexpected field, `nullSafeShortName`, which was never intended to be part of the public API. It is an internal Java helper method that was accidentally exposed because the JSON serialiser reflects all public getters on the response bean — any method named `getX()` becomes a field `x` in the response.

**Root cause:** `RouteV2Bean.getNullSafeShortName()` is a Java helper method following the `getX()` naming convention. The JSON serialiser reflects all public getters, causing `nullSafeShortName` to appear as a first-class field in every route object. It was intended as an internal display helper for route sorting, not a public API surface.

**Value of the field:** Equals `shortName` when non-null; otherwise equals the full combined route `id`.

**Affected endpoints:** [[route]], [[routes-for-agency]], [[routes-for-location]], [[search-route]]

**Observed behaviour:** Every route object contains `nullSafeShortName` alongside `shortName`.

**Intended behaviour:** The field should not appear in responses. Callers already receive `shortName` and `id` and can implement the same fallback themselves.

**Client impact if fixed:** Clients that use `nullSafeShortName` to avoid null-checking `shortName` would break — they would need to implement `shortName ?? id` themselves.

**Decision:** [x] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

Wayfinder uses nullSafeShortName in its search results. When a rider searches for a route, the result titles would show internal database identifiers, opaque numbers like `"100479"`, instead of the human-readable route names they recognise from bus stops and schedules, such as `"E Line"`. In the case of King County Metro for example, every search result would be affected, since all KCM routes have short names.


---

## DC-8: Service alert application-key filter always no-ops

**Root cause:** `BeanFactoryV2.isSituationExcludedForApplication()` contains a tautological boolean: `return !_applicationKey.contains(_applicationKey)`. A string always contains itself, so this always returns `false` — the method never signals exclusion. The intended code is `return !applicationIds.contains(_applicationKey)`.

**Affected endpoints:** [[arrivals-and-departures-for-stop]], [[trip-details]] (and any other endpoint using `BeanFactoryV2`'s situation filtering)

**Observed behaviour:** The OBA API supports service alerts (referred to as "situations" in responses) that can be targeted at specific client applications — an alert can include a list of API keys in its `affects` field, meaning it should only be shown to those clients. The filter that enforces this targeting is broken: every alert is included in every response regardless of the caller's API key.

**Intended behaviour:** An alert with application-specific targeting should only appear in responses to callers whose API key is in the alert's `affects` list.

**Client impact if fixed:** Clients that happen to receive alerts not intended for them will stop seeing those alerts in responses. Any client that has built logic around those alerts — for example displaying them to users — would silently lose that data.

**Decision:** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

## DC-9: Occupancy fields use -1 sentinel instead of field absence

Two endpoints return real-time vehicle occupancy data in their responses. When no occupancy data is available, the convention used elsewhere in the API is to omit the field entirely — clients check for field presence to determine whether data exists. The occupancy count and capacity fields instead use `-1` as a sentinel value for "no data", leaking an implementation detail into the API surface and creating an inconsistency with the related `occupancyStatus` field on the same response, which is already handled correctly by omission.

**Root cause:** `BeanFactoryV2` emits `occupancyCount: -1` and `occupancyCapacity: -1` when real-time occupancy data is unavailable. These sentinel values are serialised directly to JSON. The related `occupancyStatus` field is already handled correctly (absent when unavailable), creating an inconsistency within the same response block.

**Affected endpoints:** [[trip-for-vehicle]], [[vehicles-for-agency]]

**Observed behaviour:** `occupancyCount: -1` and `occupancyCapacity: -1` appear in responses when no occupancy data is available.

**Intended behaviour:** The fields should be absent (omitted from the JSON object) when no data is available, consistent with how `occupancyStatus` is handled.

**Client impact if fixed:** Clients that use `-1` as their "no data" signal for these fields must switch to checking for field absence.

Clients targeting both implementations must handle both conventions simultaneously: `-1` (legacy) and field absence (Maglev). A defensive check along the lines of `occupancyCount != null && occupancyCount != -1` covers both, but is more complex than either convention alone.

**Decision:** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

## DC-10: Error response `version` hardcoded to 1

The OBA response envelope includes a `version` field that echoes the API version the caller requested (always version 2 in current implementations). When an error occurs, two separate code sites that construct error response envelopes both hardcode `version: 1` regardless of what the caller supplied, creating an inconsistency between success and error responses on the same endpoint.

**Root cause:** Two code sites hardcode `version: 1` in error responses regardless of the `version` parameter the caller supplied:
1. `ExceptionInterceptor.getExceptionAsResponseBean` — affects all endpoints where an unhandled exception produces an error response
2. `ApiKeyInterceptor` — affects 401 (invalid key) and 429 (rate limit) responses across all authenticated endpoints

**Affected endpoints:** All endpoints (cross-cutting)

**Observed behaviour:** Error responses always contain `"version": 1`; successful responses for the same endpoint carry `"version": 2`.

**Intended behaviour:** Error responses should echo the caller's requested version, consistent with success responses.

**Client impact if fixed:** Clients that branch on the `version` field in error responses would receive `2` instead of `1`. This is an unlikely pattern but possible in defensive code.

**Note:** This defect is largely masked on most endpoints by DC-1 — the null-body response has no version field at all. As DC-1 is fixed, this defect becomes more visible.

**Decision:** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

## DC-11: stops-for-location — query-mode parameter interactions

[[stops-for-location]] supports two distinct search modes. The geospatial mode (default) finds stops within a geographic area specified by coordinates and radius or span. The query mode is activated by supplying the `query` parameter, which searches for stops by stop code (the short public-facing number shown on signage) rather than by location. These two modes follow different code paths, and the query-mode path has two defects where parameters are silently ignored:

**Root cause (DC-11a — `time` ignored):** In the query-mode path, `null` is passed as the service interval when loading each stop's routes. The `time` parameter has no effect; `routeIds` on each returned stop reflects all routes across all service dates, rather than routes with service in the time window containing `time`.

**Root cause (DC-11b — `routeType` ignored):** When both `query` and `routeType` are supplied, `getStopsByBoundsAndQuery` never evaluates the route-type filter chain. Stops are returned regardless of route type.

**Affected endpoint:** [[stops-for-location]]

**Observed behaviour:** When `query` is present, `time` and `routeType` are silently ignored. A stop's `routeIds` reflects all service dates regardless of `time`. Stops of the wrong route type appear when `routeType` is combined with `query`.

**Intended behaviour:** All supplied parameters should be respected.

**Client impact if fixed:** Clients combining `query`+`routeType` currently receive unfiltered results; fixing this would reduce the result set. Clients combining `query`+`time` currently see routes from all service dates on each stop; fixing this could remove routes from displayed stop records.

Clients targeting both implementations may see different route sets on the same stop depending on which server responded — legacy returns all-dates routes regardless of `time`, Maglev returns time-filtered routes. Display of "which routes serve this stop" could differ between servers for the same query.

**Decision:** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

## DC-12: trips-for-location — search area clamping bugs

[[trips-for-location]] accepts a search area either as a `radius` (in
metres, centred on a lat/lon point) or as `latSpan`/`lonSpan` offsets from
a centre point. The API imposes a maximum search area of 20 km to
limit response sizes. Two defects in the same
`SearchBoundsFactory.createBounds()` clamping block mean this limit is
not correctly enforced in either mode — the code computes clamped
values into local variables but then reads the original unclamped
fields:

**Root cause (DC-12a — radius mode):** When radius > 20,000 m is supplied, the clamping branch rebuilds the bounding box using `_latSpan / 2` and `_lonSpan / 2` — the raw query parameters, which are zero for radius-based requests (the caller used `radius`, not `latSpan`/`lonSpan`). The result is a zero-area point, returning no trips.

**Root cause (DC-12b — latSpan/lonSpan mode):** When `latSpan`/`lonSpan` exceed the maximum, clamped values are computed into local variables but `boundsFromLatLonOffset` is called with the original unclamped field values.

**Affected endpoint:** [[trips-for-location]]

**Observed behaviour:** Radius > 20 km → HTTP 200 with empty list. Oversized `latSpan`/`lonSpan` → unclamped search area, potentially returning more results than intended.

**Intended behaviour (stated inline in spec):** Radius should be clamped to 20 km; oversized spans should use the clamped values.

**Client impact if fixed:** Clients passing large radii currently receive zero results; fixing this would return results up to the 20 km cap. Clients relying on large spans to search very wide areas would see their results capped.

**Decision:** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

## DC-13: trip-details — NPE when schedule suppression active and `includeStatus=false`

[[trip-details]] supports a `includeStatus=false` parameter that omits real-time vehicle status from the response. Some agencies also configure schedule suppression, which hides schedule data unless a vehicle is actively assigned to the trip. When both are in play together, a null-dereference bug produces an HTTP 500.

**Root cause:** `TripDetailsAction` calls `trip.getStatus().getVehicleId()` unconditionally in its schedule-suppression check. When `includeStatus=false`, `trip.getStatus()` returns null, causing a NullPointerException and an HTTP 500 response.

**Affected endpoint:** [[trip-details]]

**Observed behaviour:** HTTP 500 when an agency has schedule suppression enabled, `includeStatus=false` is supplied, and no vehicle is assigned to the trip.

**Intended behaviour:** HTTP 404 when no vehicle is assigned (per the spec).

**Client impact if fixed:** Clients receiving 500 in this scenario and treating it as "try later" would now receive 404 (trip unavailable). Only affects deployments with schedule suppression enabled.

**Decision:** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

## DC-14: search/route — empty-string `route_long_name` makes route unsearchable

[[search-route]] builds its search index from route names drawn from the GTFS feed. When a route has neither a `route_short_name` nor a meaningful `route_long_name`, the indexer is supposed to fall back to the entity ID so the route remains findable. A null check that fails to account for empty strings means routes with `route_long_name = ""` are silently dropped from the index entirely.

**Root cause:** `BundleSearchServiceImpl` checks `if (hint == null)` to fall back to the entity ID when no long name exists. An empty string is not null, so routes with `route_long_name = ""` are indexed under the search term `""`, generating no search keys and making the route invisible to any query.

**Affected endpoint:** [[search-route]]

**Observed behaviour:** Routes with an empty `route_long_name` (and no `route_short_name`) do not appear in any search result.

**Intended behaviour:** The empty-string case should fall back to the entity ID, matching the null case.

**Client impact if fixed:** Routes that were invisible to search would now appear. Clients that worked around the issue with direct ID lookups would see no breakage but might encounter the routes in search results for the first time.

**Decision:** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

## DC-15: search/stop — multi-route stops bypass route-type filter

[[search-stop]] applies a filter to exclude stops that serve only special vehicle types — specifically school bus route types (GTFS types 711–714) — on the basis that these are not generally useful to transit riders. The filter logic contains an off-by-one style mistake: it only checks the route type for stops served by a single route, letting stops served by two or more school-bus routes through unchecked.

**Root cause:** `StopAction` passes any stop with two or more routes unconditionally, checking the excluded-type list (school bus types 711–714) only for single-route stops.

**Affected endpoint:** [[search-stop]]

**Observed behaviour:** Stops served by two or more school-bus-type routes appear in search results even when the route-type filter should exclude them.

**Intended behaviour:** The filter should suppress stops serving only excluded vehicle types regardless of how many routes they have.

**Client impact if fixed:** Some stops currently visible in search results would disappear.

**Decision:** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

## DC-16: stop-ids-for-agency returns IDs in undefined order

[[stop-ids-for-agency]] and [[stops-for-agency]] both return the complete set of stops for an agency — the former as a compact list of IDs, the latter as full stop records. The full-records endpoint sorts its results alphabetically by combined stop ID. The IDs-only endpoint uses a different code path that performs no sorting, so the two endpoints return the same set of stops in inconsistent and unpredictable order.

**Root cause:** `StopsBeanServiceImpl.getStopsIdsForAgencyId` iterates the transit graph without sorting, whereas its counterpart `getStopsForAgencyId` (used by `stops-for-agency`) applies an alphabetical sort by combined stop ID. Both endpoints return the same set of stops for an agency, but in different orders.

**Affected endpoints:** [[stop-ids-for-agency]] (see also [[stops-for-agency]])

**Observed behaviour:** IDs returned by `stop-ids-for-agency` are in transit-graph iteration order, which may differ from alphabetical order and may vary across server restarts.

**Intended behaviour:** The same alphabetical sort by combined ID that `stops-for-agency` applies.

**Client impact if fixed:** Clients that have inferred a particular order from repeated calls and relied on positional access or diff-based change detection would see re-ordered output.

**Decision:** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

## DC-17: current-time — `readableTime` formatted in JVM timezone, not feed timezone

[[current-time]] returns the server's current time in two forms: `time` as a Unix millisecond timestamp, and `readableTime` as a human-readable ISO 8601 string. The stated purpose of `readableTime` is to help clients determine the active service day — which requires the timestamp to be expressed in the transit feed's local timezone. Instead it uses the JVM process timezone, which may differ from the feed's timezone depending on how the server is deployed.

**Root cause:** `CurrentTimeAction` calls `DateLibrary.getTimeAsIso8601String(date)`, which uses `TimeZone.getDefault()` — the JVM process timezone — rather than the transit feed's configured local timezone.

**Affected endpoint:** [[current-time]]

**Observed behaviour:** `readableTime` carries a UTC offset corresponding to the JVM's timezone, which may differ from the feed's local timezone. A feed configured for `America/Los_Angeles` running on a UTC-configured JVM produces timestamps with `+00:00` instead of `-07:00`/`-08:00`.

**Intended behaviour:** `readableTime` should use the feed's configured timezone, since its stated purpose is to help clients determine the active service day.

**Client impact if fixed:** Deployments where the JVM timezone matches the feed timezone see no change. In mismatched deployments, fixing this would change the UTC offset suffix in `readableTime` — for example, a string that previously read `2026-05-29T14:32:00+00:00` would now read `2026-05-29T07:32:00-07:00`. Clients that parse the full ISO 8601 string correctly (treating the offset as part of the timestamp) are unaffected. Clients that strip or hardcode the offset, or compare the string lexicographically against a fixed pattern, could produce incorrect results.

**Decision:** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

---

## DC-19: vehicles-for-agency — `ageInSeconds=0` silently disables age filter

[[vehicles-for-agency]] supports an `ageInSeconds` parameter that filters out vehicle tracking records older than the specified number of seconds, allowing clients to request only fresh real-time data. Supplying `ageInSeconds=0` is treated identically to omitting the parameter, returning all vehicles regardless of how stale their data is.

**Root cause:** `VehiclesForAgencyAction` checks `_ageInSeconds != null && _ageInSeconds > 0` before applying the age filter, treating zero identically to absent.

**Affected endpoint:** [[vehicles-for-agency]]

**Observed behaviour:** Supplying `ageInSeconds=0` returns all tracked vehicles regardless of staleness, identical to omitting the parameter.

**Intended behaviour:** Ambiguous. `ageInSeconds=0` literally means "only vehicles updated at this exact instant", which is rarely useful. The current behaviour (treat as absent) may be the pragmatic intent.

**Client impact if fixed:** Clients using `ageInSeconds=0` as a "no filter" shorthand would need to switch to omitting the parameter. Given the ambiguity of the intended semantics, this defect may be best left as-is with documentation.

**Decision:** [ ] Preserve  [ ] Fix  [ ] Conditional fix

**Notes:** _(to be filled during workshop)_

---

## Next Steps

After the workshop, decisions recorded here will be fed back into the individual spec files. For each defect class where the decision is **Fix** or **Conditional fix**, the affected spec files should receive an **Implementation Decisions** section (or have their existing one updated) stating the Maglev behaviour and how it deviates from the legacy.
