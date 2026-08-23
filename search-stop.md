# Search: Stop

## Goal in Context

A rider's client app wants to help the user find stops by name as they type, using a prefix-based autocomplete interface.

## Scope

OBA REST API — stop name search endpoint.

## Level

User goal.

## Primary Actor

Rider (accessing the system via a client app).

## Stakeholders and Interests

- **Rider** — wants to find boarding locations by entering part of a stop name; expects matches to appear after a few keystrokes and to see enough context (location, served routes) to confirm they have the right stop.

## Preconditions

- The search index has been built from the feed data (this happens asynchronously at application startup; if the application has just started, the index may be empty).
- The caller supplies a valid API key.

## Minimal Guarantees

- If the input is missing, the server returns HTTP 404 with a validation error body (outside the standard envelope).
- If no stops match the prefix query, the server returns a 404 response in the standard envelope. *Maglev deviates from this — see [Implementation Decisions](#implementation-decisions).*
- The response never contains stops that lack revenue service (at least one scheduled stop time with unrestricted pick-up or drop-off).

## Success Guarantees

- The response contains up to `maxCount` stops whose names match the given prefix.
- Each stop entry includes its combined ID, name, location, direction, stop code, location type, wheelchair accessibility, associated route IDs, and static route IDs.
- All routes referenced in the stop entries are fully described in the references block; all agencies owning those routes are also included.
- The `limitExceeded` flag correctly indicates whether the number of raw prefix matches (before route-type filtering) exceeded `maxCount`.

## Trigger

An HTTP GET request to `/api/where/search/stop.json` (or `.xml`) with a non-empty `input` parameter.

## Main Success Scenario

1. The server receives a GET request. It converts `input` to lower case before using it as the lookup key.

2. The server looks up the key in the stop-name search index. This index is keyed by all prefixes of each whitespace/hyphen/slash/parenthesis/ampersand-delimited token in every stop name. For multi-word names, the index also stores keys that span the first token boundary and extend character by character up to 32 characters, enabling queries like `pine st & 9th` to match `Pine St & 9th Ave`. The key is matched exactly — there is no fuzzy matching. ([`BundleSearchServiceImpl.java:169-201`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/bundle/BundleSearchServiceImpl.java#L169-L201))

3. Stops within the matching bucket are ordered by their combined stop ID (ascending lexicographic order). ([`BundleSearchServiceImpl.java:337-345`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/bundle/BundleSearchServiceImpl.java#L337-L345))

4. If the bucket contains more than `maxCount` stops, the list is truncated to `maxCount` and `limitExceeded` is set to `true`. ([`BundleSearchServiceImpl.java:213-223`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/bundle/BundleSearchServiceImpl.java#L213-L223))

5. The truncated list is filtered to remove stops that are served exclusively by special-vehicle route types. Specifically: a stop with exactly one route is excluded if that route's type is 711, 712, 713, or 714; a stop with two or more routes is always included regardless of their types; a stop with zero routes is excluded. ([`StopAction.java:61-65`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/search/StopAction.java#L61-L65))

6. The server converts each remaining stop to its API representation and collects all routes and agencies referenced by those stops into the references block. Routes in the references block are sorted by short name (plain lexicographic compare, falling back directly to the combined route ID when short name is unset); within each agency, the primary agency's routes appear first when a custom sort is configured. ([`BeanFactoryV2.java:147-161`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L147-L161)) *Maglev deviates from this — see [Implementation Decisions](#implementation-decisions).*

7. The server returns HTTP 200 with the standard envelope containing the list, `limitExceeded`, `outOfRange` (always `false`), and the references block.

## Extensions

**2a. No stops match the prefix:**
The server returns HTTP 404 with code 404 and text "resource not found" in the standard envelope. *Maglev deviates from this — see [Implementation Decisions](#implementation-decisions).*

**2b. Stops match the prefix but all are excluded by the route-type filter:**
The server returns HTTP 200 with an empty list. `limitExceeded` may be `true` if the pre-filter count exceeded `maxCount`. (Note: the 404 check occurs before filtering, so an all-filtered result does not produce a 404.)

**Input validation failure:**
If `input` is absent, Struts2 request validation fires before the action runs. The response is HTTP 404 with the body `{"fieldErrors":{"input":["missing input"]}}` — a raw validation error object, not the standard OBA response envelope.

**Missing or invalid API key:**
The server returns HTTP 401.

## Suspected Defects

### Defects that affect the use case

**1. `includeReferences=false` produces a null response body**

[`BeanFactoryV2.java:147-153`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L147-L153)

When `includeReferences=false` is passed, no routes are added to the references collection. `BeanFactoryV2.getResponse(StopSearchResultBean)` then calls `.sort()` unconditionally on the references route list, which is `null`, throwing a NullPointerException. The exception propagates before the response bean is set. The serialised response is the JSON literal `null` with HTTP 200. The intended behaviour is to return the stop list without a references block (as `includeReferences=false` works on other endpoints). *Maglev intentionally corrects this — see [Implementation Decisions](#implementation-decisions).*

**2. `limitExceeded` reflects the pre-filter cap, not the post-filter result count**

[`StopAction.java:55-66`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/search/StopAction.java#L55-L66)

The `maxCount` cap is applied before the route-type filter. A response may therefore contain `limitExceeded=true` alongside a list with fewer than `maxCount` entries (because some of the capped entries were subsequently filtered out). A caller cannot determine from `limitExceeded` alone whether there are additional unfiltered results beyond what was returned.

**3. Route-type filter does not apply to stops with multiple routes**

[`StopAction.java:61-65`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/search/StopAction.java#L61-L65)

The intent of the filter appears to be to suppress stops that serve only special vehicle types (school buses, etc.). However, the implementation passes any stop with two or more routes unconditionally, regardless of whether both routes are excluded types. Only single-route stops are subject to the type check. A stop with two school-bus routes would therefore appear in results.

### Implementation defects only

**4. Latent NullPointerException in route-type filter predicate**

[`StopAction.java:61-65`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/search/StopAction.java#L61-L65)

Due to Java's `&&`/`||` operator precedence, the `routes != null` guard only protects the `size() > 1` branch. If `routes` were `null`, evaluating `routes.size() == 1` in the second branch would throw a NullPointerException. This does not occur in practice because the service layer always initialises the routes list to an empty ArrayList, but the predicate is incorrectly written. A clean reimplementation should guard against null or rely on an always-initialised collection.

**5. Unused `ArrivalsAndDeparturesQueryBean` field in `StopAction`**

[`StopAction.java:38`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/search/StopAction.java#L38)

`StopAction` declares and initialises a `_query` field of type `ArrivalsAndDeparturesQueryBean` that is never read or passed to any service method. It has no observable effect and can be omitted in a reimplementation.

**6. `agencyId` parameter ignored throughout**

[`StopAction.java:55`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/search/StopAction.java#L55)

The call to `_service.getStopSuggestions(null, _input, maxCount)` hardcodes `null` as the agency ID. The service implementation ignores agency ID entirely; the search is always across all agencies. If per-agency scope was intended for future use, it is not implemented.

## Implementation Decisions

**`includeReferences=false` returns the stop list without a references block, rather than reproducing the legacy NullPointerException (deviates from legacy).**

The legacy implementation throws a NullPointerException when `includeReferences=false` is passed, serialising the response as the JSON literal `null` with HTTP 200 (see [Suspected Defects](#suspected-defects)). Maglev intentionally corrects this: passing `includeReferences=false` returns the standard envelope with `data.list` populated as usual and `data.references` present but with all arrays (`agencies`, `routes`, `situations`, `stops`) empty, consistent with how `includeReferences=false` behaves on other Maglev endpoints.

**`data.references.routes` is sorted using natural (digit-aware) ordering with a short name → long name → agency ID → route ID fallback chain, rather than legacy's plain lexicographic short name → route ID comparator (deviates from legacy).**

The legacy implementation (`BeanFactoryV2.getResponse(StopSearchResultBean)`) sorts `references.routes` via an injected `RouteSorting` strategy; the default implementation (`DefaultRouteSort.compareRoutes`) does a plain `String.compareTo` on each route's `nullSafeShortName` — `shortName` if set, otherwise the combined route ID directly. This is plain alphabetical ordering (e.g. route `"10"` sorts before route `"9"`) and never considers `longName`.

Maglev instead sorts `references.routes` using the same natural-sort, multi-field comparator (`utils.SortModelRoutesByName`) it already uses for route references on the `stop` and `schedule-for-stop` endpoints: sort by `shortName`, falling back to `longName`, then `agencyId`, then `id`, with natural string ordering (embedded digit runs compared numerically, so `"9"` sorts before `"10"`). This favors one consistent, predictable route-reference sort behavior across all Maglev endpoints over exact, endpoint-by-endpoint conformance with each legacy comparator — `stop` and `schedule-for-stop` each have their own distinct legacy sort routines that this comparator already matches; `search-stop`'s legacy routine happens to be simpler (lexicographic, no long-name fallback), but Maglev applies the shared rule here too rather than special-casing this endpoint.

Practical effect versus legacy: routes with purely numeric short names sort in numeric rather than lexicographic order, and a route with an empty `shortName` but a non-empty `longName` sorts by that `longName` text rather than falling straight to its ID.

**`data.references.situations` is populated for stop alerts, rather than preserving Java's empty situation-reference list for this endpoint (deviates from legacy implementation).**

The legacy response model has a generic `references.situations` field, and Java's custom JSON serializer writes null `List` properties as empty arrays. However, the `search-stop` path never adds service alerts to that list: `StopAction.index()` wraps only the stop suggestions in `StopSearchResultBean`, `BeanFactoryV2.getResponse(StopSearchResultBean)` converts each stop with `getStop(StopBean)`, and `getStop` adds route references, agency references through those routes, and parent-stop references, but no `ServiceAlertBean` references.

Maglev treats that as an omission in the legacy endpoint implementation and populates `references.situations` with service alerts affecting the returned stops when references are included. This lets clients resolve stop-level alerts directly from a stop-search response instead of issuing a follow-up stop or arrivals request.

Practical effect versus legacy: Java exposes an empty `situations` array for this endpoint even when matching stops have active alerts; Maglev may return real situation objects. Because those objects come from real-time alert state, the endpoint is not purely static when situation references are included.

**A no-match prefix query returns HTTP 200 with an empty list, rather than HTTP 404 (deviates from legacy).**

The legacy implementation ([`StopAction.java#L56-L57`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/search/StopAction.java#L56-L57)) explicitly checks whether the pre-filter suggestion list is empty and short-circuits to a 404 in that case:

```java
ListBean<StopBean> stopSuggestions = _service.getStopSuggestions(null, _input, maxCount);
if (stopSuggestions == null || stopSuggestions.getList().isEmpty())
    return setResourceNotFoundResponse();
```

Maglev instead treats a query that matches no stops as a successful, empty result (HTTP 200, `data.list: []`) — consistent with how an all-filtered result is already handled under Extension 2b. This mirrors the same decision made for the sibling `search/route.json` endpoint (see [search-route § Implementation Decisions](search-route.md#implementation-decisions)).

Practical effect versus legacy: callers that used to receive HTTP 404 with the standard error envelope (`code: 404`, `text: "resource not found"`) for a no-match query now receive HTTP 200 with `data.list: []`, `data.references` present with empty arrays, and `data.limitExceeded: false`.

**A parent station shared by children under different agencies gets one `data.references.stops` entry per (parent station, resolving agency) pair, rather than legacy's single, agency-stable entry (deviates from legacy, out of structural necessity).**

The legacy implementation sets a stop's `parent` from the parent `StopBean`'s own `AgencyAndId` — a compound key every GTFS entity is assigned once, at bundle-ingestion time, based on which federated bundle it was loaded from ([`BeanFactoryV2.getStop()`, lines 488–500](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L488-L500), the same converter `getResponse(StopSearchResultBean)` uses to build `search-stop`'s results). A physical station's agency identity is therefore fixed upstream of any request: one station always yields exactly one ID and one `references.stops` entry, regardless of which agency's child stop or route is asking.

Maglev has no equivalent mechanism to borrow this from. Legacy's per-stop agency identity is a byproduct of federating multiple separate bundles, each pre-assigned an agency prefix at ingestion; Maglev loads a single static bundle, so that axis of identity doesn't exist. Consistent with that, Maglev's `stops` table has no `agency_id` column at all (`gtfsdb/schema.sql`), matching raw GTFS's own `stops.txt`, which has no such field either — only `routes.txt` does. GTFS still permits one bundle to define multiple agencies (`agency.txt`) whose routes can serve a shared physical `stop_id`, so a parent station genuinely reachable through two agencies' children is a real scenario for a single-bundle Maglev deployment on a consolidated regional feed, not a hypothetical one.

Maglev instead resolves each stop's agency contextually, from its own serving routes (deterministically: the lowest agency ID among `GetRoutesForStops`' agency-then-route-ID-ordered results), and buckets that stop's parent-station ID under the resolved agency. A parent station reached through children under two different agencies therefore gets two separate combined IDs and two separate `references.stops` entries. This was raised and explicitly endorsed as one of two acceptable fixes during review — "[consider] emitting one entry per (parent, agency) pair actually referenced" ([PR #1221 review comment](https://github.com/OneBusAway/maglev/pull/1221#issuecomment-5133651781)) — for an earlier version of this code that instead let one arbitrary child's agency win for the whole parent, silently orphaning the others' `parent` references.

Practical effect versus legacy: a client walking `references.stops` sees what is physically one station appear as multiple distinct-looking entries (e.g. `999_parent_stat_1` and `888_parent_stat_1`), with no signal they're the same place — unlike legacy's single stable identity per station. If this proves too surprising in practice, the fallback is to instead pick one canonical agency per parent station (e.g. lowest agency ID, mirroring the tie-break Maglev already uses for a stop with no route of its own), trading a single stable ID per station for children whose `parent` field may then point at an ID prefixed by a different agency than their own.

## Open Questions

None.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["input", "key"],
  "properties": {
    "input": {
      "type": "string",
      "description": "Prefix to search for. Case-insensitive. Must be non-empty."
    },
    "maxCount": {
      "type": "integer",
      "default": 20,
      "description": "Maximum number of stops to return."
    },
    "key": {
      "type": "string",
      "description": "API key."
    },
    "version": {
      "type": "integer",
      "default": 2
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "Whether to include the references block. Note: passing false triggers a server bug (see Suspected Defects)."
    }
  }
}
```

**`input`** — The string to match against stop names. The server converts it to lower case before lookup. Matching is prefix-based: a value of `pine` matches any stop whose name contains a token starting with `pine`. Multi-word prefixes (e.g. `pine st`) are also matched as long as they correspond to a prefix of the full stop name up to 32 characters. Does not match against stop codes or stop entity IDs.

**`maxCount`** — Upper bound on the number of results returned. Defaults to 20. Applied before the route-type filter, so the actual list may be shorter.

**`key`** — API authentication key. Missing or invalid key returns HTTP 401.

**`version`** — Response version selector. Defaults to 2. Only version 2 is supported.

**`includeReferences`** — When `true` (default), the `references` block is populated with route and agency objects. Passing `false` causes a server-side bug that returns a null response body; do not use.

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
    "data":        { "type": "object" }
  }
}
```

**`code`** — HTTP status mirrored in the body: 200 for success, 404 if no stops matched. *Maglev deviates from this — see [Implementation Decisions](#implementation-decisions).*

**`currentTime`** — Server time at the moment the response was generated, in Unix milliseconds.

**`text`** — Human-readable status message (e.g. `"OK"`, `"resource not found"`).

**`version`** — API version used for the response (always 2).

**`data`** — The result payload; absent when `code` is 404.

---

### `data`

```json
{
  "type": "object",
  "properties": {
    "list":         { "type": "array", "items": { "type": "object" } },
    "limitExceeded": { "type": "boolean" },
    "outOfRange":   { "type": "boolean" },
    "references":   { "type": "object" }
  }
}
```

**`data.list`** — Array of stop objects matching the query. May be empty if all matched stops were excluded by the route-type filter. *Maglev also returns an empty list rather than a 404 when no stops matched the prefix at all — see [Implementation Decisions](#implementation-decisions).*

**`data.limitExceeded`** — `true` if the raw number of index matches exceeded `maxCount` before filtering. The actual list may contain fewer than `maxCount` entries.

**`data.outOfRange`** — Always `false` for this endpoint.

**`data.references`** — Full objects for all routes and agencies referenced in the list. Present when `includeReferences=true`. Legacy Java exposes an empty `situations` array for this endpoint; Maglev populates it with stop-alert references when applicable — see [Implementation Decisions](#implementation-decisions).

---

### `data.list[]` (stop entry)

```json
{
  "type": "object",
  "properties": {
    "id":                 { "type": "string" },
    "code":               { "type": "string" },
    "name":               { "type": "string" },
    "lat":                { "type": "number" },
    "lon":                { "type": "number" },
    "direction":          { "type": "string" },
    "locationType":       { "type": "integer" },
    "wheelchairBoarding": { "type": "string" },
    "routeIds":           { "type": "array", "items": { "type": "string" } },
    "staticRouteIds":     { "type": "array", "items": { "type": "string" } },
    "parent":             { "type": "string" }
  }
}
```

**`data.list[].id`** — Combined stop ID in `{agencyId}_{entityId}` form (e.g. `1_1085`).

**`data.list[].code`** — The human-readable stop code as published in the feed (e.g. `"1085"`). Typically shown on physical stop signs. Falls back to the entity portion of the combined ID when the feed supplies no code, matching `data.entry.code` on [`stop`](stop.md) — legacy's stop-suggestion index is built from `StopBean` objects fetched via the same `TransitDataService.getStop()` call that backs the single-stop endpoint ([`BundleSearchServiceImpl.java:100`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/bundle/BundleSearchServiceImpl.java#L100)), which ultimately resolves through [`fillStopBean`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopBeanServiceImpl.java#L173-L185)'s `StringLibrary.getBestName(narrative.getCode(), stop.getId().getId())`.

**`data.list[].name`** — The stop name as it appears in the feed (e.g. `"Pine St & 9th Ave"`).

**`data.list[].lat`** — Latitude of the stop in decimal degrees.

**`data.list[].lon`** — Longitude of the stop in decimal degrees.

**`data.list[].direction`** — Cardinal or intercardinal direction of travel at this stop (e.g. `"SW"`, `"N"`). Taken from the feed when defined; otherwise inferred from shape geometry, or empty/absent if neither is available. See [[Stop-Direction-Calculation]] for the inference algorithm.

**`data.list[].locationType`** — GTFS `location_type` value: `0` = stop/platform, `1` = station, `2` = station entrance/exit, `3` = generic node, `4` = boarding area.

**`data.list[].wheelchairBoarding`** — Accessibility status: `"ACCESSIBLE"`, `"NOT_ACCESSIBLE"`, `"UNKNOWN"`, or `"PARTIALLY_ACCESSIBLE"`.

**`data.list[].routeIds`** — Combined IDs of routes currently serving this stop. Each ID corresponds to an entry in `data.references.routes`.

**`data.list[].staticRouteIds`** — Combined IDs of routes statically configured to serve this stop. In most deployments this is identical to `routeIds`; it can differ when the feed includes explicit static route overrides for the stop.

**`data.list[].parent`** — Combined ID of the parent station if this stop is a sub-component of a station; empty string if there is no parent.

---

### `data.references`

```json
{
  "type": "object",
  "properties": {
    "agencies":   { "type": "array", "items": { "type": "object" } },
    "routes":     { "type": "array", "items": { "type": "object" } },
    "situations": { "type": "array", "items": { "type": "object" } },
    "stops":      { "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.references.agencies`** — Full agency objects for all agencies that own a route appearing in `data.references.routes`. Sorted with the primary agency first (when a custom route sort is configured), then alphabetically by agency ID.

**`data.references.routes`** — Full route objects for all routes referenced by `routeIds` or `staticRouteIds` across the returned stops. Sorted by route short name (alphabetically). If no short name is set, the combined route ID is used as the sort key. *Maglev deviates from this sort rule — see [Implementation Decisions](#implementation-decisions).*

**`data.references.stops`** — Parent station objects for any stop in the list that has a non-empty `parent` field.

**`data.references.situations`** — Service-alert situation references. Legacy Java leaves this array empty for `search-stop`; Maglev populates it with alerts affecting returned stops — see [Implementation Decisions](#implementation-decisions).

---

### `data.references.routes[]`

```json
{
  "type": "object",
  "properties": {
    "id":          { "type": "string" },
    "agencyId":    { "type": "string" },
    "shortName":   { "type": "string" },
    "longName":    { "type": "string" },
    "description": { "type": "string" },
    "type":        { "type": "integer" },
    "url":         { "type": "string" },
    "color":       { "type": "string" },
    "textColor":   { "type": "string" }
  }
}
```

**`data.references.routes[].id`** — Combined route ID in `{agencyId}_{entityId}` form.

**`data.references.routes[].agencyId`** — Plain agency ID (not combined).

**`data.references.routes[].shortName`** — Public-facing route number or short label (e.g. `"10"`). May be absent.

**`data.references.routes[].longName`** — Full route name (e.g. `"Capitol Hill - Downtown Seattle"`). May be absent.

**`data.references.routes[].description`** — Supplementary description from the feed. May be absent.

**`data.references.routes[].type`** — GTFS `route_type` integer (e.g. `3` for bus).

**`data.references.routes[].url`** — URL to the route's timetable or information page. May be absent.

**`data.references.routes[].color`** — Route colour as a 6-digit hex string without `#` (e.g. `"FDB71A"`). May be absent.

**`data.references.routes[].textColor`** — Text colour for use on the route colour background, as a 6-digit hex string. May be absent.

---

### `data.references.agencies[]`

```json
{
  "type": "object",
  "properties": {
    "id":             { "type": "string" },
    "name":           { "type": "string" },
    "url":            { "type": "string" },
    "timezone":       { "type": "string" },
    "lang":           { "type": "string" },
    "phone":          { "type": "string" },
    "fareUrl":        { "type": "string" },
    "email":          { "type": "string" },
    "disclaimer":     { "type": "string" },
    "privateService": { "type": "boolean" }
  }
}
```

**`data.references.agencies[].id`** — Plain agency ID.

**`data.references.agencies[].name`** — Agency name (e.g. `"Metro Transit"`).

**`data.references.agencies[].url`** — Agency website URL.

**`data.references.agencies[].timezone`** — IANA timezone identifier (e.g. `"America/Los_Angeles"`).

**`data.references.agencies[].lang`** — Primary language code (e.g. `"en"`).

**`data.references.agencies[].phone`** — Rider contact phone number.

**`data.references.agencies[].fareUrl`** — URL to the agency's fare information page.

**`data.references.agencies[].email`** — Contact email address.

**`data.references.agencies[].disclaimer`** — Optional disclaimer text.

**`data.references.agencies[].privateService`** — Whether the agency operates a private (non-public) service.
