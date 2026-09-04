# stops-for-location

## Goal in Context

A rider's client app needs to display transit stops near a map location — either all stops within a geographic area, or stops matching a particular stop code — so the rider can select a stop to view departure times.

## Scope

OneBusAway REST API — the `GET /api/where/stops-for-location.json` endpoint.

## Level

User goal.

## Primary Actor

Rider (via a client app).

## Stakeholders and Interests

- **Rider**: wants accurate, nearby stops with correct route associations so they can plan their journey.

## Preconditions

- The server has at least one agency's transit data loaded.
- The caller supplies a valid `lat` and `lon` coordinate pair.

## Minimal Guarantees

- The server returns a well-formed JSON envelope with a `code` field.
- If the search area falls entirely outside the coverage area of every loaded agency, the response includes `outOfRange: true` and an empty list.
- If `maxCount` is not positive, the server responds with HTTP 400 and a field-level validation error.

## Success Guarantees

- The response contains a list of stops whose geographic coordinates fall within the derived search bounds, with each stop including its routes, location type, and accessibility information.
- The list is sorted lexicographically by combined stop ID.
- The `limitExceeded` flag accurately indicates whether the candidate pool in the search area exceeded the requested limit.
- Referenced routes and agencies are populated in the `references` block (unless suppressed by `includeReferences=false`).

## Trigger

A client sends `GET /api/where/stops-for-location.json` with `lat` and `lon` query parameters.

## Main Success Scenario

1. The client sends a request with at minimum a `lat`/`lon` coordinate pair. Optional parameters narrow the search area, filter by stop code, or filter by route type.

2. The server derives a search bounding box from the parameters:
   - If `radius` is provided and greater than zero, the box encloses a circle of that radius centred on `lat`/`lon`. ([`StopsForLocationAction.java#L187`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L187))
   - Otherwise, if both `latSpan` and `lonSpan` are provided and greater than zero, the box is centred on `lat`/`lon` with half-dimensions `latSpan/2` and `lonSpan/2`. ([`StopsForLocationAction.java#L189–L191`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L189))
   - Otherwise, if `query` is provided, a 10,000-metre radius is used. ([`StopsForLocationAction.java#L194–L196`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L194))
   - Otherwise, a 500-metre radius is used. ([`StopsForLocationAction.java#L197–L199`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L197))

3. The server checks whether the derived bounds intersect at least one agency's coverage area. If not, it returns an empty list with `outOfRange: true`. ([`TransitDataServiceTemplateImpl.java#L849–L863`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/federated/TransitDataServiceTemplateImpl.java#L849))

4. The server establishes a time context. If `time` is provided, it is used; otherwise the current server time is used. The time is snapped to a 15-minute bin for caching purposes. This time context determines which routes are considered actively serving each stop. ([`ApiIntervalFactory.java#L41–L46`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/services/ApiIntervalFactory.java#L41))

   The calendar day this instant resolves to is computed using the server JVM's own default timezone — a single process-wide setting, not one resolved per agency or per query. ([`AgencyServiceInterval.java#L47–L49`](https://github.com/OneBusAway/onebusaway-gtfs-modules/blob/8784ee50f31a0f95cedee7e4eb382ba58ae14eec/onebusaway-gtfs/src/main/java/org/onebusaway/gtfs/model/calendar/AgencyServiceInterval.java#L47), [`ServiceDate.java#L82–L89`](https://github.com/OneBusAway/onebusaway-gtfs-modules/blob/8784ee50f31a0f95cedee7e4eb382ba58ae14eec/onebusaway-gtfs/src/main/java/org/onebusaway/gtfs/model/calendar/ServiceDate.java#L82)) Maglev instead resolves this per query against the queried agency's own IANA timezone — see [Implementation Decisions](#implementation-decisions).

5. **Without `query`**: The server retrieves all stops from the geospatial index that fall within the search bounds. ([`WhereGeospatialServiceImpl.java#L85–L100`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/WhereGeospatialServiceImpl.java#L85))

   - If the raw candidate count exceeds `maxCount` and `routeType` is not specified, the candidate list is randomly shuffled and truncated to `maxCount` before any further processing; `limitExceeded` is set to `true`. This truncation happens before route filtering, so the final returned count may be lower than `maxCount`. ([`StopsBeanServiceImpl.java#L130–L134`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L130))

   - For each candidate stop, the server loads its full stop record, populating `routeIds` with only those routes that have scheduled service within the 15-minute time window. Stops that have no routes active in that window are excluded from results. ([`StopsBeanServiceImpl.java#L143–L161`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L143))

   - If `routeType` is specified, the server additionally filters the remaining stops to keep only those served by at least one route of one of the requested GTFS route types, checking against the time-filtered route set. ([`StopsBeanServiceImpl.java#L163–L168`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L163))

6. **With `query`**: The server searches a Lucene-backed stop-code index — a relevance-scored, tokenized text search, not an exact-match lookup — for stops whose code matches the query string, considering at most 10 candidates. ([`StopsBeanServiceImpl.java#L191`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L191), [`StopSearchServiceImpl.java#L81–L118`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/StopSearchServiceImpl.java#L81)) Maglev matches stop codes exactly instead — see [Implementation Decisions](#implementation-decisions).

   - From those candidates, only those whose coordinates fall within the derived search bounds are kept. ([`StopsBeanServiceImpl.java#L204–L206`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L204))

   - If more stops match than `maxCount`, the matched stops are randomly shuffled and truncated to `maxCount`; `limitExceeded` is set to `true`. ([`BeanServiceSupport.java#L23–L31`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/BeanServiceSupport.java#L23)) Maglev does not reproduce this shuffle — see [Implementation Decisions](#implementation-decisions).

   - If no candidates fall within the bounds, the single geographically closest candidate from the full set of 10 is returned regardless of its distance from the centre point. ([`StopsBeanServiceImpl.java#L228–L229`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L228))

   - Routes for each stop are loaded without time filtering: `routeIds` includes all routes associated with the stop across all service dates.

7. The result list is sorted lexicographically by combined stop ID. ([`StopsBeanServiceImpl.java#L274`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L274))

8. The server builds the response envelope. Each stop entry in the list has its associated routes and agencies added to the `references` block (unless `includeReferences=false`).

9. The server returns HTTP 200 with the complete response.

## Extensions

**2a. maxCount is not positive:**
The server returns HTTP 400 with a field-level validation message: `{"fieldErrors":{"maxCount":["must be greater than zero"]}}`. ([`StopsForLocationAction.java#L125–L129`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L125))

**2b. maxCount exceeds the absolute cap of 250:**
The effective limit is silently capped at 250. The caller's value is reduced to 250 without any error or indication in the response. ([`MaxCountSupport.java#L37–L39`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/impl/MaxCountSupport.java#L37))

**3a. Search bounds do not intersect any agency coverage area:**
The server returns HTTP 200 with `outOfRange: true` and an empty `list`. ([`StopsForLocationAction.java#L158–L160`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L158))

**4a. `time` fails to parse (neither a digit string nor `yyyy-MM-dd_HH-mm-ss`):**
The property-setter's conversion throws a `TypeConversionException`. ([`DateTimeConverter.java#L52–L68`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-presentation/src/main/java/org/onebusaway/presentation/impl/conversion/DateTimeConverter.java#L52)) Confirmed via live testing against legacy 2.7.1: this does not block the request or surface as a field error — the request is served using the current-time fallback, the same as an absent `time`. Maglev matches this — see the `queryTimeForStops` fallback in [Implementation Decisions](#implementation-decisions).

**5a. No stops in the area have any active routes for the given time (without-query mode):**
All candidates are filtered out in step 5. The server returns an empty list with `limitExceeded: false` and `outOfRange: false`.

**6a. Query matches no stop codes anywhere in the feed:**
The stop-code search returns no candidates. No closest-fallback is possible. The server returns an empty list.

**6b. A code-matched stop has no routes associated with it at all, not just none active today:**
Legacy still returns the stop — the stop-code path loads each match's routes with no routes-empty filter (contrast MSS step 5's geospatial path, which does exclude routeless stops). Maglev currently drops it instead — see [Implementation Decisions](#implementation-decisions).

**8a. includeReferences=false:**
The references block is present in the response but all its sub-arrays (`routes`, `agencies`, `stops`, `trips`, `situations`) are empty.

## Suspected Defects

### Defects that affect the use case

**1. routeType filter does not truncate results when the limit is exceeded.**

`StopsBeanServiceImpl` ([`StopsBeanServiceImpl.java#L171–L177`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L171))

When `routeType` is specified, the code correctly sets `limitExceeded = true` when the post-filter count exceeds `maxCount`, but the list is passed to `constructResult` without being truncated. The comment in the code ("constructResults will perform the truncation if necessary") is incorrect: `constructResult` only sorts by stop ID. As a result, a caller requesting `maxCount=10` with `routeType=3` may receive hundreds of stops.

**2. The `time` parameter has no effect on route filtering when `query` is provided.**

`StopsBeanServiceImpl` ([`StopsBeanServiceImpl.java#L203`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L203))

In the stop-code search path, `null` is passed as the service interval when loading each stop's routes, so `routeIds` reflects all routes across all service dates regardless of the `time` parameter. In the geospatial search path (without `query`), the time context is passed and routes are filtered accordingly. The inconsistency means a caller cannot rely on `routeIds` to reflect time-active routes when supplying a stop-code query.

**3. The `routeType` filter is silently ignored when `query` is also provided.**

`StopsBeanServiceImpl` ([`StopsBeanServiceImpl.java#L180–L232`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L180))

When both `query` and `routeType` are provided, the route-type filter is added to the search query's filter chain by the action class but `getStopsByBoundsAndQuery` never evaluates that filter chain. All stops matching the stop-code query are returned regardless of their route types.

### Implementation defects only

**4. Limit check in the geospatial path runs before route filtering.**

`StopsBeanServiceImpl` ([`StopsBeanServiceImpl.java#L130–L134`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L130))

When `routeType` is not specified, the raw geospatial candidate IDs are shuffled and truncated before stop beans are loaded and filtered by active routes. This means `limitExceeded=true` is set based on the raw count in the area, not the count after route filtering. A clean reimplementation would not have this ordering issue.

## Implementation Decisions

**Stop-code (`query`) truncation is deterministic, nearest-first, rather than legacy's random shuffle (deviates from legacy).**

Legacy shuffles the in-bounds candidate list before truncating to `maxCount` in the stop-code search path too, not just the geospatial path. `StopsForLocationAction` sets query type `BOUNDS_OR_CLOSEST` whenever `query` is supplied ([`StopsForLocationAction.java#L150–L152`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L150)), and `StopsBeanServiceImpl.getStopsByBoundsAndQuery` only takes the distance-sorted, deterministic-truncation branch when the query type is `ORDERED_BY_CLOSEST` — that type is reserved for the `routeType` path, not this one. For `BOUNDS_OR_CLOSEST` it falls through to `BeanServiceSupport.checkLimitExceeded`, the same `Collections.shuffle`-before-truncate helper used by the geospatial path (see MSS step 6, third bullet, and [`StopsBeanServiceImpl.java#L213–L225`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L213)).

Maglev instead sorts the in-bounds candidates by distance to the search centre and truncates deterministically, keeping the nearest `maxCount` matches. This follows the same choice already shipped for `routes-for-location`'s truncation, which orders by distance in SQL rather than shuffling — see [routes-for-location § Suspected Defects, #1](routes-for-location.md#suspected-defects) and [DC-6a/DC-6d in the defect workshop](OBA-API-Defect-Workshop.md#dc-6-non-deterministic-or-incorrect-result-truncation). Determinism also fits this endpoint's purpose better than legacy's behaviour here: a stop-code lookup is trying to find *a* specific stop, so keeping the nearest candidates on truncation is more useful to a caller than an arbitrary random subset, and it makes repeated identical requests return identical results.

Practical effect versus legacy: repeated identical requests that exceed `maxCount` in-bounds matches for a stop code now consistently return the same nearest stops, rather than a random subset that can vary between requests. No client inspected during this decision (Wayfinder, iOS, Android, the JS SDK) relies on the random-sampling behaviour here. If this proves wrong, the fallback is to reproduce legacy's `Collections.shuffle`-before-truncate exactly, matching the geospatial (non-`query`) path's existing shuffle a few lines above this code in the same function (`GetStopsForLocation` in `internal/gtfs/gtfs_manager.go`).

**Stop-code (`query`) matching is an exact-equality lookup against `stops.code`, rather than legacy's relevance-scored Lucene text search (deviates from legacy).**

Legacy resolves `query` through a Lucene full-text index, not a direct string comparison: `StopsBeanServiceImpl.getStopsByBoundsAndQuery` calls `_searchService.searchForStopsByCode(query, 10, MIN_SCORE)` ([`StopsBeanServiceImpl.java#L189–L191`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L189), `MIN_SCORE = 1.0` at [`StopsBeanServiceImpl.java#L57`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L57)). That method runs the query through a `StandardAnalyzer` and `MultiFieldQueryParser` against a pre-built on-disk Lucene index of stop codes, using classic TF-IDF scoring (`ClassicSimilarity`) and discarding any hit below the minimum score ([`StopSearchServiceImpl.java#L53–L118`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/StopSearchServiceImpl.java#L53)). This is a tokenized, relevance-ranked search — closer to a fuzzy/partial match than an exact lookup — though exactly how stop codes are tokenized at index-build time (which determines how "fuzzy" a match can be) wasn't traced as part of this decision.

Maglev's `GetStopsByCode` (`gtfsdb/query.sql`) instead does a direct `WHERE code = ?` equality match. This predates [PR #1270](https://github.com/OneBusAway/maglev/pull/1270) — the original in-bounds-only stop-code lookup was already exact-match before that PR extended the search feed-wide — and was deliberately left unaddressed there as a separate, larger gap: reproducing Lucene-equivalent relevance search would mean building or embedding a comparable text-search engine (e.g. SQLite FTS5 with a scoring/ranking scheme, as already used for the `search-stop` endpoint's prefix matching in this codebase), not a small query change.

Practical effect versus legacy: a caller whose `query` doesn't exactly match a stop's `code` field gets no results from Maglev, where legacy might still surface a fuzzy/tokenized match. No client behaviour around this was assessed as part of this decision. If this proves to matter in practice, the fallback is to build a proper text-search index over `stops.code` (SQLite FTS5, mirroring the codebase's existing `search-stop` approach) rather than trying to hand-roll Lucene-equivalent scoring.

**A stop-code (`query`) match with no associated routes is dropped instead of returned, due to Maglev's single-feed agency-resolution limitation (deviates from legacy, out of structural necessity).**

Legacy's stop-code path loads each matched stop's routes via `_stopBeanService.getStopForId(aid, null)` with no routes-empty filter (contrast the geospatial path's explicit `if (stopBean.getRoutes().isEmpty()) continue`, MSS step 5) — so a code match is returned regardless of whether it currently has any associated routes. This works unconditionally in legacy because every entity in a federated bundle is pre-assigned its owning agency at ingest (an `AgencyAndId` stamped by the loader); returning a routeless stop never requires resolving an agency after the fact, since the ID is already fixed.

Maglev has no equivalent to borrow from. Its `stops` table has no `agency_id` column at all — matching raw GTFS's own `stops.txt`, which also lacks one — so a stop's agency is derived transitively, per response, from the routes serving it (`stop_times → trips → routes → agencies`, via `GetAgenciesForStops`). A stop with zero `stop_times` — exactly the scenario a routeless code match implies — has no row in that join, so no agency can be resolved for it, and `stopsForLocationHandler`'s results loop (the `agency == nil` check) drops it rather than returning it with an empty `routeIds`. This is the same structural gap already documented in [search-stop § Implementation Decisions](search-stop.md#implementation-decisions) (there, for parent stations reached through multiple agencies' children) and in [`Multi-Agency-Static-Feeds-Stop-Consolidation`](Multi-Agency-Static-Feeds-Stop-Consolidation.md) §2.1 — Maglev's stop-to-agency association is transitive rather than pre-assigned, a direct consequence of supporting only a single static feed today.

Practical effect versus legacy: a rider searching for a genuinely valid stop code gets an empty result from Maglev instead of the stop, including when it's the only or closest candidate the new fallback logic (see #1239, above) would otherwise have surfaced. The most plausible real case is a parent station, whose own `stop_times` are typically empty while its child platforms carry the actual service. This isn't fixed here: doing so would mean adding a reverse lookup (find the matched stop's children via `parent_station`, then resolve an agency from their routes) that doesn't exist in this code path today, and a genuinely childless routeless stop (no trips, no children either) would still have no resolvable agency under any approach short of storing one directly on `stops`. The actual fix is federated multi-feed support — see the deviation-tracking table in [`Multi-Agency-Static-Feeds-Stop-Consolidation`](Multi-Agency-Static-Feeds-Stop-Consolidation.md#10-deviations-from-legacy-due-to-the-single-feed-limitation), which pre-assigns each stop's agency at ingest exactly as legacy does; this deviation should be revisited once that lands.

---

**Current-time fallback timezone is the queried agency's IANA zone, not the server JVM's default timezone (deviates from legacy).**

In Java, an absent `time` parameter resolves to `SystemTime.currentTimeMillis()`, and the identical downstream conversion path also handles an explicitly supplied value. ([`StopsForLocationAction.java#L117–L137`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L117)) Neither this action nor `ApiIntervalFactory.constructForDate`/`constructDefault` attaches a timezone to that instant. ([`ApiIntervalFactory.java#L41–L46`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/services/ApiIntervalFactory.java#L41)) It's converted into a calendar day only inside `AgencyServiceInterval`'s constructor, via `new ServiceDate(new Date(referenceTime))` ([`AgencyServiceInterval.java#L47–L49`](https://github.com/OneBusAway/onebusaway-gtfs-modules/blob/8784ee50f31a0f95cedee7e4eb382ba58ae14eec/onebusaway-gtfs/src/main/java/org/onebusaway/gtfs/model/calendar/AgencyServiceInterval.java#L47)), whose own constructor javadoc states it "uses the default `TimeZone` object for the current VM to localize the date." ([`ServiceDate.java#L82–L89`](https://github.com/OneBusAway/onebusaway-gtfs-modules/blob/8784ee50f31a0f95cedee7e4eb382ba58ae14eec/onebusaway-gtfs/src/main/java/org/onebusaway/gtfs/model/calendar/ServiceDate.java#L82)) That default is a single, process-wide JVM setting (`Calendar.getInstance()`), shared by every agency the server hosts — Java never re-resolves it per query or per agency. (`AgencyServiceInterval`'s per-agency `agencyIdOverrides` map only shifts the *window width* in minutes after the reference time; it has no effect on which timezone computes the calendar day itself. [`AgencyServiceInterval.java#L38–L42`](https://github.com/OneBusAway/onebusaway-gtfs-modules/blob/8784ee50f31a0f95cedee7e4eb382ba58ae14eec/onebusaway-gtfs/src/main/java/org/onebusaway/gtfs/model/calendar/AgencyServiceInterval.java#L38))

Maglev's `queryTimeForStops` (`internal/restapi/stops_for_location_handler.go`) instead loads the queried agency's own IANA timezone (`loadAgencyLocation`) and resolves both the explicit `time` parameter and the absent/malformed fallback against it, matching the approach already used by [trips-for-location § Implementation Decisions](trips-for-location.md#implementation-decisions).

This is a deliberate improvement, not a port of Java's literal mechanism. Java's correctness in a single-timezone deployment depends entirely on an operational convention — ops setting the JVM's `-Duser.timezone` to match the (usually single) agency being served — that this code path neither enforces nor references. A Maglev instance whose container/host clock defaults to UTC, serving a non-UTC agency, previously selected the wrong service date for hours at a time around local midnight; resolving against the agency's actual `agency_timezone` fixes this without depending on deployment configuration being correct. In the conventional case (JVM timezone already matches the agency), both approaches produce identical results — this is why the malformed/absent-`time` fallback (see Extensions 4a and MSS step 4) was independently confirmed via live testing against legacy 2.7.1 to behave the same in that configuration.

Trade-off: under Maglev's current single-feed limit, "the queried agency" collapses to `agencies[0]`, since a query's bounding box can't be mapped to a specific owning agency without per-stop agency assignment — see the next decision below and [Multi-Agency-Static-Feeds-Stop-Consolidation.md §10](Multi-Agency-Static-Feeds-Stop-Consolidation.md#10-deviations-from-legacy-due-to-the-single-feed-limitation).

If this is reconsidered, the fallback is Java's literal behavior: resolve "now" from the server process's own configured default timezone rather than the agency's, accepting the UTC-container failure mode this decision exists to avoid unless ops-level timezone configuration is guaranteed correct.

---

**Time-context agency is `agencies[0]`, not a location-specific agency, due to the single-feed limitation (deviates from legacy, out of structural necessity).**

In Java, every entity is pre-assigned its owning agency at ingest via `AgencyAndId`, so any per-agency-scoped code — such as `CalendarServiceDataFactoryImpl`'s use of each service's own `agency_timezone` to build its active-date list — always resolves an entity's real, fixed agency, never a guess across a federated bundle of several agencies.

Maglev has no equivalent to borrow from for this query: `queryTimeForStops` needs *an* agency's timezone before it knows which stops the search will return, and a queried lat/lon can't be mapped to an owning agency ahead of time without per-stop agency assignment (§2.1 of the multi-agency spec). It uses `agencies[0]` — the first agency in `GetAgencies`'s result set — as a stand-in.

Practical effect versus legacy: under Maglev's current single-static-feed limitation this is unconditionally correct, since exactly one agency is ever loaded — `agencies[0]` and "the agency actually being queried" are the same agency by construction. It would stop being correct once multi-feed support lands and a bounding box could span two agencies in different timezones, unlike `trips-for-location`'s per-trip service-date resolution, which at least keys its own agency lookup per trip rather than computing one global date for an entire response.

This is tracked as a row in [Multi-Agency-Static-Feeds-Stop-Consolidation.md §10](Multi-Agency-Static-Feeds-Stop-Consolidation.md#10-deviations-from-legacy-due-to-the-single-feed-limitation) rather than treated as a standalone decision to revisit independently — the fix is the same federated multi-feed support already tracked for that whole table.

## Open Questions

None. All observable behaviours were resolved through static source analysis and live server testing.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["lat", "lon"],
  "properties": {
    "lat": {
      "type": "number",
      "description": "WGS-84 latitude of the search centre, in decimal degrees."
    },
    "lon": {
      "type": "number",
      "description": "WGS-84 longitude of the search centre, in decimal degrees."
    },
    "radius": {
      "type": "number",
      "description": "Search radius in metres. Takes precedence over latSpan/lonSpan when greater than zero."
    },
    "latSpan": {
      "type": "number",
      "description": "Latitudinal extent of the search bounding box in decimal degrees. Used together with lonSpan when radius is not supplied."
    },
    "lonSpan": {
      "type": "number",
      "description": "Longitudinal extent of the search bounding box in decimal degrees. Used together with latSpan when radius is not supplied."
    },
    "query": {
      "type": "string",
      "description": "Stop code to search for. When supplied, the search switches to a code-matching mode with a 10 km default radius."
    },
    "maxCount": {
      "type": "integer",
      "default": 100,
      "description": "Maximum number of stops to return. Must be greater than zero. Silently capped at 250 regardless of the supplied value."
    },
    "routeType": {
      "type": "string",
      "description": "Comma-separated list of GTFS route type integers (e.g. '3' for bus, '0' for tram). When supplied, only stops served by at least one route of a matching type are returned."
    },
    "time": {
      "type": "string",
      "description": "Point in time used to determine which routes are actively serving each stop. Accepts Unix milliseconds as a digit string, or the format yyyy-MM-dd_HH-mm-ss. Defaults to the current server time."
    },
    "key": {
      "type": "string",
      "description": "API key. Required by convention."
    },
    "version": {
      "type": "integer",
      "default": 2,
      "description": "API version. Only version 2 is in scope for Maglev."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "When false, the references block is present but all sub-arrays are empty."
    }
  }
}
```

**`lat`** — WGS-84 latitude of the search centre in decimal degrees. Required.

**`lon`** — WGS-84 longitude of the search centre in decimal degrees. Required.

**`radius`** — Search radius in metres. When greater than zero it takes precedence over `latSpan`/`lonSpan`. Defaults to 500 m (no query) or 10 000 m (with `query`) when omitted.

**`latSpan`** — Latitudinal span of the search bounding box in decimal degrees. Both `latSpan` and `lonSpan` must be greater than zero for the span-based area to take effect. The box is centred on `lat`/`lon`, with the span halved in each direction.

**`lonSpan`** — Longitudinal span of the search bounding box in decimal degrees. See `latSpan`.

**`query`** — A stop code to match against. When supplied, legacy searches a Lucene relevance-scored text index rather than doing a pure geospatial lookup or an exact match; Maglev matches `stops.code` exactly instead — see [Implementation Decisions](#implementation-decisions). At most 10 code matches are considered, and the results are filtered to those within the derived bounds. If none fall within the bounds, the geographically closest code match is returned as a fallback. Note that `time` has no effect on route filtering in this mode, and `routeType` is silently ignored.

**`maxCount`** — Maximum number of stops to return. Default 100. The server silently caps this at 250; values above 250 are treated as 250. A value of zero or less causes a 400 validation error. When the geospatial candidate pool exceeds this limit (without `routeType`), the pool is randomly shuffled before truncation, making results non-deterministic. The stop-code (`query`) candidate pool is also randomly shuffled before truncation in legacy; Maglev truncates it deterministically by distance instead — see [Implementation Decisions](#implementation-decisions).

**`routeType`** — One or more GTFS route type integers, comma-separated. Restricts results to stops served by at least one route of a listed type. The type is matched against the routes active for the `time` context. When combined with `query`, this parameter is silently ignored (see Suspected Defects).

**`time`** — Reference time for determining which routes are actively serving each stop. Accepts either a Unix millisecond timestamp as a plain integer string, or a formatted string in the pattern `yyyy-MM-dd_HH-mm-ss`. Defaults to the current server time; a value that fails to parse also falls back to the current time rather than erroring (Extension 4a). In legacy, the resulting calendar day is computed in the server JVM's own default timezone; Maglev resolves it against the queried agency's timezone instead — see [Implementation Decisions](#implementation-decisions). Has no effect on route filtering when `query` is supplied.

**`key`** — API key passed with all requests.

**`version`** — API version selector. Defaults to 2. Maglev implements only version 2.

**`includeReferences`** — Controls whether the `references` block is populated. Defaults to `true`. When `false`, the references object is present but empty.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "version": { "type": "integer" },
    "code": { "type": "integer" },
    "text": { "type": "string" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "data": { "type": "object" }
  }
}
```

**`version`** — API version used for this response (2).

**`code`** — HTTP-style status code: 200 for success.

**`text`** — Human-readable status: `"OK"` on success.

**`currentTime`** — Server's current wall-clock time as Unix milliseconds.

**`data`** — The list result object described below.

---

### data

```json
{
  "type": "object",
  "properties": {
    "list": {
      "type": "array",
      "items": { "type": "object" }
    },
    "limitExceeded": { "type": "boolean" },
    "outOfRange": { "type": "boolean" },
    "references": { "type": "object" }
  }
}
```

**`data.list`** — Array of stop objects matching the query. Sorted lexicographically by combined stop ID.

**`data.limitExceeded`** — `true` if the candidate pool exceeded `maxCount`. In the geospatial (no-`query`) path, when `true` without `routeType`, legacy's result was randomly sampled and repeated requests may return different stops; when `true` with `routeType`, all matching stops are returned despite the flag (see Suspected Defects). In the stop-code (`query`) path, legacy also randomly samples before truncating; Maglev instead truncates deterministically by distance — see [Implementation Decisions](#implementation-decisions).

**`data.outOfRange`** — `true` when the search area does not intersect any agency's coverage area. The list is empty when this flag is `true`.

**`data.references`** — Deduplicated objects referenced by IDs in the list. Contains sub-arrays `agencies`, `routes`, `stops`, `trips`, `situations`, and `stopTimes`. `situations` is empty because this endpoint has no `situationIds` field; see [[Situation-References]]. All arrays are empty when `includeReferences=false`.

---

### data.list[]

```json
{
  "type": "object",
  "properties": {
    "id": { "type": "string" },
    "lat": { "type": "number" },
    "lon": { "type": "number" },
    "name": { "type": "string" },
    "code": { "type": "string" },
    "direction": { "type": "string" },
    "locationType": { "type": "integer" },
    "wheelchairBoarding": { "type": "string" },
    "routeIds": {
      "type": "array",
      "items": { "type": "string" }
    },
    "staticRouteIds": {
      "type": "array",
      "items": { "type": "string" }
    },
    "parent": { "type": "string" }
  }
}
```

**`data.list[].id`** — Combined stop ID in `{agencyId}_{stopId}` format, e.g. `"1_75403"`.

**`data.list[].lat`** — Stop latitude in decimal degrees.

**`data.list[].lon`** — Stop longitude in decimal degrees.

**`data.list[].name`** — Human-readable stop name, e.g. `"Stevens Way & BENTON LANE"`.

**`data.list[].code`** — Passenger-facing stop code, e.g. `"75403"`. Corresponds to the value in the feed's `stop_code` field, or falls back to the raw stop entity ID if no code is defined. May be an empty string if absent.

**`data.list[].direction`** — Compass direction indicating which way a vehicle is travelling when serving this stop (e.g. `"N"`, `"SE"`). Taken from the feed when defined; otherwise inferred from shape geometry, or empty string if neither is available. See [[Stop-Direction-Calculation]] for the inference algorithm.

**`data.list[].locationType`** — GTFS `location_type` value: `0` = boarding/alighting stop, `1` = station, `2` = entrance/exit. The majority of stops are `0`.

**`data.list[].wheelchairBoarding`** — Wheelchair accessibility status: `"ACCESSIBLE"`, `"NOT_ACCESSIBLE"`, or `"UNKNOWN"`. May be absent if not specified in the feed.

**`data.list[].routeIds`** — Combined route IDs (`{agencyId}_{routeId}`) of routes actively serving this stop. When `query` is absent, only routes with service scheduled within the 15-minute window containing the `time` parameter are included. When `query` is present, routes for all service dates are included. Each referenced route appears in `data.references.routes`.

**`data.list[].staticRouteIds`** — Combined route IDs representing a manually configured static route list for the stop. When no static override is configured, this is identical to `routeIds`. When an override exists, it may differ from `routeIds`.

**`data.list[].parent`** — Combined ID of the parent station if the stop is a sub-component of a station structure (GTFS `parent_station`). Empty string when the stop has no parent. The parent stop object is included in `data.references.stops`.

---

### data.references.routes[]

```json
{
  "type": "object",
  "properties": {
    "id": { "type": "string" },
    "agencyId": { "type": "string" },
    "shortName": { "type": "string" },
    "longName": { "type": "string" },
    "description": { "type": "string" },
    "type": { "type": "integer" },
    "url": { "type": "string" },
    "color": { "type": "string" },
    "textColor": { "type": "string" },
    "nullSafeShortName": { "type": "string" }
  }
}
```

**`data.references.routes[].id`** — Combined route ID in `{agencyId}_{routeId}` format.

**`data.references.routes[].agencyId`** — Plain agency ID (not combined) of the agency operating this route.

**`data.references.routes[].shortName`** — Route short name from the feed (e.g. `"40"`). May be absent.

**`data.references.routes[].longName`** — Route long name from the feed. May be absent.

**`data.references.routes[].description`** — Route description from the feed. May be absent.

**`data.references.routes[].type`** — GTFS route type integer (e.g. `3` for bus, `0` for light rail).

**`data.references.routes[].url`** — URL to the route's schedule or information page. May be absent.

**`data.references.routes[].color`** — Route colour as a six-character hex string without `#` (e.g. `"FDB71A"`). May be absent.

**`data.references.routes[].textColor`** — Foreground colour for text on the route colour background. May be absent.

**`data.references.routes[].nullSafeShortName`** — The short name if present, otherwise the long name. Never absent.
