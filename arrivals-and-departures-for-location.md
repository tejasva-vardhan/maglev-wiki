# arrivals-and-departures-for-location

`GET /api/where/arrivals-and-departures-for-location.json`

## Goal in Context

A rider's client application shows a combined departure board for every stop in an area around a point, such as the rider's current position or the visible map viewport. The system finds the stops in that area and returns the upcoming arrivals and departures at all of them, merged into one list. It also returns the stops very close to the found stops, each with its distance from the search centre, so the client can offer quick navigation to "the stop across the street". The rider can then see what is coming soon without first choosing a single stop.

## Scope

OBA REST API: `GET /api/where/arrivals-and-departures-for-location.json`

## Level

User goal

## Primary Actor

Rider (via a client application)

## Stakeholders and Interests

- **Rider**: wants to see, in one request, which vehicles are about to arrive at or depart from nearby stops, with predicted times where real-time data exists, ordered so the closest stops come first.

## Preconditions

- A valid API key is supplied.
- The caller knows a latitude/longitude inside the transit system's coverage area, plus either a search radius or a latitude/longitude span.

## Minimal Guarantees

- Every response that reaches the endpoint logic uses the standard envelope with `code`, `version`, `currentTime`, `text` and `data`.
- A malformed numeric parameter produces HTTP 400 and no transit data.
- A search that finds no stops (or falls outside the service area) returns HTTP 200 with empty lists. It never returns an error. See Extensions 5a and Suspected Defects for the shape of that empty body.

## Success Guarantees

- `data.entry.arrivalsAndDepartures` holds arrivals and departures from the stops in the search area whose scheduled or predicted time falls inside the requested window. Arrivals are grouped by stop, generally nearest stop first, and ordered by best arrival time within each stop. The list is capped at the effective maximum count.
- `data.entry.stopIds` lists the stops found in the search area. The maximum count applies to this list too.
- `data.entry.nearbyStopIds` lists stops within roughly 100 metres of any found stop. Each entry carries its straight-line distance from the search centre. The list is sorted nearest first and capped at the maximum count.
- `data.entry.limitExceeded` is `true` when any of these three lists was truncated.
- The references block contains full objects for the routes, trips, stops, agencies and service alerts referenced by the arrivals and the nearby-stop list.

## Trigger

A GET request to `/api/where/arrivals-and-departures-for-location.json` with a valid API key.

## Main Success Scenario

1. The caller sends a GET request with `lat`, `lon`, and either `radius` or both `latSpan` and `lonSpan`. It can also send any of: time-window parameters, `time`, `maxCount`, and `routeType`.

2. The system checks that every supplied numeric parameter can be parsed. If any cannot, it returns HTTP 400 ([`ArrivalsAndDeparturesForLocationAction.java#L146-L147`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L146-L147)).

3. The system builds the search rectangle ([`SearchBoundsFactory.java#L73-L106`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/impl/SearchBoundsFactory.java#L73-L106)):
   - If `radius` is greater than zero, the rectangle is the square that encloses a circle of that radius around (`lat`, `lon`). The search is a box, not a circle, so stops near the corners can be up to about 1.41 × `radius` from the centre.
   - Otherwise, if both `latSpan` and `lonSpan` are greater than zero, the rectangle is centred on (`lat`, `lon`) with those total spans in degrees.
   - Otherwise the rectangle has zero size, and the search finds no stops. This endpoint has no default radius.
   - The system is meant to cap the rectangle at the square enclosing a 20,000-metre radius. Because of a defect, the cap does not work (see Suspected Defects).

4. The system works out the effective maximum count: `maxCount` if supplied, otherwise 250, and never more than 1,000 ([`ArrivalsAndDeparturesForLocationAction.java#L77`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L77), [`MaxCountSupport.java#L41-L43`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/impl/MaxCountSupport.java#L41-L43)). There is no lower bound (see Extensions 4a and 4b).

5. The system finds every stop inside the rectangle through the geospatial index. It drops any stop that no route has ever served, and any stop rejected by server-configured stop filters ([`StopsBeanServiceImpl.java#L143-L170`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L143-L170)). The system does **not** cap this list at the maximum count. It computes a limit-exceeded flag, but nothing truncates the list, and the endpoint never reads the flag ([`StopsBeanServiceImpl.java#L171-L177`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L171-L177)). The `routeType` parameter does not affect which stops are found.

6. The system sets the query time: the `time` parameter if it was supplied and could be parsed, otherwise the current server time ([`ArrivalsAndDeparturesForLocationAction.java#L151-L152`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L151-L152)). From the query time it builds a service interval. The interval starts at the query time rounded down to the nearest 15 minutes and ends at the end of that service day ([`ApiIntervalFactory.java#L45-L52`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/services/ApiIntervalFactory.java#L45-L52)). Each stop's route list (the `routeIds` in the references block, and the routes the route-type filter checks) contains only routes with service in this interval.

7. For each found stop, in an order that is arbitrary but repeatable (the stop IDs are held in an unordered hash set; [`TransitDataServiceTemplateImpl.java#L818-L823`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/federated/TransitDataServiceTemplateImpl.java#L818-L823)), the system does the following ([`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L106-L132`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L106-L132)):
   - Adds the stop to the stop list. It adds the stop a **second** time if the stop has at least one arrival or departure in the window (see Suspected Defects).
   - Computes the stop's arrivals and departures exactly as `arrivals-and-departures-for-stop` does ([`ArrivalsAndDeparturesBeanServiceImpl.java#L181-L249`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/ArrivalsAndDeparturesBeanServiceImpl.java#L181-L249)):
     - Regular trips use the window [query time − `minutesBefore`, query time + `minutesAfter`].
     - Frequency-based trips use [query time − `frequencyMinutesBefore`, query time + `frequencyMinutesAfter`].
     - An arrival is kept if any of its scheduled or predicted arrival or departure times falls inside its window.
     - Agencies configured to hide schedule-only data lose their unpredicted arrivals.
     - The per-agency rule for suppressing arrivals the vehicle has already passed applies.
     - Server-configured arrival filters and the `routeType` filter apply (see step 9).
     - The arrivals for each stop are sorted by best arrival time, ascending: predicted if available, otherwise scheduled.
   - Drops any arrival whose route has no short name ([`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L234-L253`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L234-L253)), and adds the remaining arrivals to one combined list.
   - Collects the IDs of all other stops whose position falls inside a 100-metre square box around this stop. The stop itself is excluded ([`NearbyStopsBeanServiceImpl.java#L56-L74`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/NearbyStopsBeanServiceImpl.java#L56-L74)).
   - Collects the service alerts that currently affect the stop.

8. The system builds the nearby-stop list from the union of all collected nearby IDs. Found stops are **not** removed, so a found stop appears in the nearby list whenever another found stop lies within 100 metres of it ([`ArrivalsAndDeparturesForLocationAction.java#L163`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L163), [`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L134-L136`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L134-L136)). Nearby stops can lie outside the search rectangle. For each nearby stop ([`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L143-L167`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L143-L167)):
   - The system computes the straight-line distance in metres from the centre of the search rectangle to the stop.
   - If `routeType` was supplied, the stop's routes are reduced to those of the requested types. A stop left with no routes is dropped.
   - The list is then sorted by distance, ascending, and cut to the maximum count.

9. **Route-type filter.** When `routeType` is supplied, an arrival is kept only if the **stop** is served by at least one route of a requested type. The type of the arrival's own route is not checked ([`ArrivalAndDepartureFilterByRouteType.java#L48-L62`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data/src/main/java/org/onebusaway/transit_data/model/ArrivalAndDepartureFilterByRouteType.java#L48-L62)). Values that are not integers are silently ignored. If no valid integers remain, no filtering happens ([`ArrivalAndDepartureFilterByRouteType.java#L32-L42`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data/src/main/java/org/onebusaway/transit_data/model/ArrivalAndDepartureFilterByRouteType.java#L32-L42)). See Suspected Defects for two problems with this filter.

10. The system cuts the stop list to the maximum count by removing entries from the end. The duplicate entries from step 7 count toward the limit ([`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L175-L178`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L175-L178)).

11. The system sorts the combined arrivals list by each arrival's stop distance from the search centre, ascending ([`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L180`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L180)). The sort is stable, so arrivals at the same stop keep their time order. A stop's distance is known only if the stop was processed as a nearby stop in step 8. Stops with no known distance sort **before** all others, in the stop order from step 7 ([`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L255-L272`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L255-L272)). In practice, the output starts with isolated stops (no other stop within 100 metres) in arbitrary order, followed by clustered stops, nearest first. See Suspected Defects.

12. The system adds service alerts that affect any route serving the stops left after step 10 ([`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L182`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L182), [`#L204-L232`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L204-L232)). This includes routes that serve the stop in the published schedule but have no active service in the interval. Alerts are de-duplicated by ID.

13. The system cuts the combined arrivals list to the maximum count by removing entries from the end ([`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L185-L188`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L185-L188)). The arrivals at the farthest stops are therefore dropped first, and one busy stop can use up the whole allowance. `limitExceeded` is set to `true` if the nearby-stop list, the stop list or the arrivals list was truncated.

14. The system builds the response entry ([`BeanFactoryV2.java#L1226-L1263`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1226-L1263)):
    - `stopIds`: the IDs from the stop list, duplicates included. The stops in this list are **not** added to the references block.
    - `arrivalsAndDepartures`: each arrival is built the same way as in `arrivals-and-departures-for-stop`, and its trip, route and stop are added to references.
    - `nearbyStopIds`: stop ID and distance pairs. Each nearby stop is added to references.
    - `situationIds`: IDs of the collected alerts, which are added to references.
    - `limitExceeded`.

15. The system sorts the references block ([`BeanFactoryV2.java#L223-L253`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L223-L253)):
    - Routes are ordered by short name using a deployment-configurable ordering. With the default configuration this is plain lexicographic order, so `"10"` comes before `"2"`. Routes of a configured primary agency, if any, come first.
    - Agencies are ordered by ID, with the configured primary agency first.

16. The system returns HTTP 200 with the entry and the references block.

## Extensions

**2a. A numeric parameter (`lat`, `lon`, `radius`, `latSpan`, `lonSpan`, `maxCount`, `minutesBefore`, `minutesAfter`, `frequencyMinutesBefore`, `frequencyMinutesAfter`) cannot be parsed:**
The system returns HTTP 400. The body is a bare validation object with no envelope, for example `{"fieldErrors":{"lat":["Invalid field value for field \"lat\"."]}}` ([`ApiActionSupport.java#L135-L141`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ApiActionSupport.java#L135-L141)).

**2b. `key` is missing or invalid:**
The system returns HTTP 401: `{"code":401,"text":"permission denied","version":1,...}`.

**2c. `lat` or `lon` is omitted:**
The missing value defaults to 0. The rectangle is placed at that coordinate, normally far outside the service area, and the flow continues at 5a.

**3a. Neither `radius` nor both spans are supplied (including only one of `latSpan`/`lonSpan`):**
The rectangle has zero size, no stops are found, and the flow continues at 5a.

**3b. `radius` is greater than 20,000 metres:**
Because of the clamping defect, the rectangle collapses to a point. No stops are found, and the flow continues at 5a.

**3c. `latSpan`/`lonSpan` describe a box larger than the 20,000-metre cap:**
Because of the clamping defect, the box is used unchanged. The search can cover the whole region.

**4a. `maxCount` is 0:**
The system returns HTTP 200 with empty `stopIds`, `arrivalsAndDepartures` and `nearbyStopIds`, and `limitExceeded: true`.

**4b. `maxCount` is negative:**
Truncating the lists fails with an internal error that the endpoint does not handle. The live server returns HTTP 200 with the literal body `null`.

**5a. No stops are found in the rectangle, or the location is outside the service area:**
The system returns HTTP 200 with `code` 200 and `text` "OK". However, `data` is an unwrapped object with a **different shape** from the success response: `{"arrivalsAndDepartures":[],"limitExceeded":false,"nearbyStops":[],"situations":[],"stops":[],"timeZone":""}`. It has no `entry`, no `references`, no `stopIds` and no `nearbyStopIds` ([`ArrivalsAndDeparturesForLocationAction.java#L170-L186`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L170-L186), [`#L196-L200`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L196-L200)). The `emptyReturnsNotFound` parameter is meant to turn this into a 404, but it has no effect (see Suspected Defects).

**6a. `time` is supplied as all digits:**
The value is treated as Unix milliseconds.

**6b. `time` is supplied in the form `yyyy-MM-dd_HH-mm-ss`:**
The value is read in the server's default time zone ([`DateTimeConverter.java#L32`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-presentation/src/main/java/org/onebusaway/presentation/impl/conversion/DateTimeConverter.java#L32), [`#L52-L70`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-presentation/src/main/java/org/onebusaway/presentation/impl/conversion/DateTimeConverter.java#L52-L70)).

**6c. `time` is in any other format (for example `2026-09-23` or `garbage`):**
The value is silently ignored and the current server time is used. No validation error is returned.

**7a. A found stop has no arrivals in the window:**
The stop appears once in `stopIds`, not twice, and adds nothing to `arrivalsAndDepartures`. It can be missing from the references block (see Suspected Defects).

**7b. A trip was cancelled, or a stop was skipped, in real-time data:**
The handling is the same as in `arrivals-and-departures-for-stop`, because the same per-stop arrival computation is used.

**13a. The total number of arrivals exceeds the maximum count:**
Arrivals at the stops sorted last are dropped, and `limitExceeded` is `true`. This can happen with the default 250 at a radius of only a few hundred metres downtown.

**14a. Building the response fails unexpectedly:**
The system returns HTTP 500 with `text` "internal error" and no data ([`ArrivalsAndDeparturesForLocationAction.java#L188-L193`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L188-L193)).

**14b. `includeReferences=false`:**
The `references` object is still present, but all its lists are empty.

## Suspected Defects

### Defects that affect the use case

**`StopWithArrivalsAndDeparturesBeanServiceImpl`: stops with arrivals appear twice in `stopIds`.**
[`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L108-L117`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L108-L117)
Every stop is added to the stop list unconditionally at line 109, then added again at line 116 if it has arrivals. The comment on line 115 ("we only add stopBean if it actually has results") shows the intent: add only stops that have arrivals, once each. On the live server, a 400-metre query downtown returned 63 `stopIds` for 32 distinct stops. The duplicates also count toward the maximum count, so `limitExceeded` can be `true` even when the number of distinct stops is within the limit. The likely intended behaviour is to list each stop once, and probably only stops with arrivals.

**`StopWithArrivalsAndDeparturesBeanServiceImpl` / `StopBeanServiceImpl`: the route-type filter permanently changes cached stop data, which affects later requests and other endpoints.**
[`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L152-L160`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L152-L160), [`StopBeanServiceImpl.java#L85-L91`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopBeanServiceImpl.java#L85-L91)
The stop lookup returns a cached, shared object. The cache key is the stop ID plus the service interval, and the interval is rounded to 15 minutes. When `routeType` is supplied, the nearby-stop loop replaces the cached stop's route list with only the matching routes. Every later request in the same 15-minute bin then sees the reduced list. This includes requests to this endpoint without `routeType`, and requests to `arrivals-and-departures-for-stop`. Confirmed on the live server:
1. `routeType=0` was requested around stop `1_26645` (served by two bus routes and one streetcar route).
2. A following `routeType=3` request returned **no** bus arrivals at that stop.
3. After that, both this endpoint and `arrivals-and-departures-for-stop/1_26645` listed only the streetcar route in the stop's `routeIds`.

The Go implementation should never change shared stop data while filtering for a request.

**`ArrivalAndDepartureFilterByRouteType`: the route-type filter tests the stop's routes, not the arrival's route.**
[`ArrivalAndDepartureFilterByRouteType.java#L52-L58`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data/src/main/java/org/onebusaway/transit_data/model/ArrivalAndDepartureFilterByRouteType.java#L52-L58)
An arrival passes if *any* route serving its stop matches a requested type. At a stop shared by buses and a streetcar, `routeType=0` (tram) returned 6 bus arrivals alongside 4 streetcar arrivals on the live server. The documented intent ("filter by GTFS route types") implies testing the type of the arrival's own route.

**`StopWithArrivalsAndDeparturesBeanServiceImpl`: arrivals are ordered by a distance that is only sometimes known, and can be stale.**
[`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L180`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L180), [`#L255-L265`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L255-L265)

The arrivals list is sorted by stop distance from the search centre, but nothing in this request computes that distance for the arrival stops. The distance is present only because the nearby-stop loop writes it onto the same cached stop object. This has three consequences:
- Stops with no other stop within 100 metres never get a distance, and the comparator puts missing distances **first**. On the live server, a stop 419 metres from the centre came before one 162 metres away.
- Because the stop object is cached across requests (see above), a stop that is not in this request's nearby set can keep a distance calculated for a *different* search centre earlier in the same 15-minute bin.
- Truncating the arrivals list (step 13) therefore does not reliably drop the farthest stops first.

The likely intent is a nearest-stop-first ordering with the distance computed for every found stop in the current request.

**`StopWithArrivalsAndDeparturesBeanServiceImpl`: the stop list is cut in hash order, not by distance.**
[`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L175-L178`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L175-L178)
The stop search asks for nearest-first ordering so that a truncated result stays deterministic ([`ArrivalsAndDeparturesForLocationAction.java#L157-L159`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L157-L159)). That order is then lost when the IDs are put into a hash set ([`TransitDataServiceTemplateImpl.java#L818-L823`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/federated/TransitDataServiceTemplateImpl.java#L818-L823)), so the entries removed by truncation are arbitrary. With `maxCount=5` the live server returned `["1_101","1_101","1_102","1_102","1_103"]`, which are not the five nearest stops. The stop search itself never truncates its result ([`StopsBeanServiceImpl.java#L171-L176`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L171-L176)), despite the comment saying it will.

**`SearchBoundsFactory`: clamping to the maximum radius uses the wrong values.**
[`SearchBoundsFactory.java#L87-L91`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/impl/SearchBoundsFactory.java#L87-L91)
When the requested box is larger than the 20,000-metre cap, the code computes capped spans into local variables. It then builds the box from the *original* span fields and ignores the capped values. The result:
- A `radius` over 20,000 metres produces a zero-size box, because the span fields are 0, so the search finds nothing. Confirmed with `radius=30000`.
- Oversized `latSpan`/`lonSpan` values are not capped at all. A 1° × 1° box returned 507 stops.

The intended behaviour is to cap the box at the square enclosing a 20,000-metre radius.

**`ArrivalsAndDeparturesForLocationAction`: there is no default search radius.**
[`ArrivalsAndDeparturesForLocationAction.java#L76`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L76), [`SearchBoundsFactory.java#L104-L105`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/impl/SearchBoundsFactory.java#L104-L105)
The bounds helper falls back to a default radius, but this endpoint never sets one, so the default is 0. A request with only `lat`/`lon` silently returns an empty result instead of an error or a sensible default. Other location endpoints use a default radius. The Go implementation should either apply one or reject the request.

**`ArrivalsAndDeparturesForLocationAction`: empty results use a different response shape.**
[`ArrivalsAndDeparturesForLocationAction.java#L182`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L182), [`#L199`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L199)
The empty and out-of-service-area paths put the raw internal result object into `data`, instead of the `entry`/`references` structure. Clients then see `stops`, `nearbyStops`, `situations` and `timeZone` rather than `entry.stopIds`, `entry.nearbyStopIds` and so on. The likely intent is the normal response shape with empty lists.

**`ArrivalsAndDeparturesForLocationAction`: `emptyReturnsNotFound` has no effect.**
[`ArrivalsAndDeparturesForLocationAction.java#L196-L200`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L196-L200)
The code sets a 404 response and then immediately overwrites it with a 200 OK. The out-of-service-area path does not check the flag at all. Confirmed: `emptyReturnsNotFound=true` returned HTTP 200. The intended behaviour is HTTP 404 when no stops are found.

**`ArrivalsAndDeparturesForLocationAction`: a negative `maxCount` is not rejected.**
[`ArrivalsAndDeparturesForLocationAction.java#L141-L143`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L141-L143), [`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L175-L177`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L175-L177)
The truncation loops try to remove an element from an empty list, which raises an exception that the endpoint does not handle. The observed result is HTTP 200 with body `null`. `stops-for-location` rejects values ≤ 0 with a 400. A similar check is the likely intent here.

**`ArrivalsAndDeparturesForLocationAction`: an unparseable `time` is silently ignored.**
[`ArrivalsAndDeparturesForLocationAction.java#L127-L129`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ArrivalsAndDeparturesForLocationAction.java#L127-L129)
The parse error is swallowed during parameter binding, and the query runs at the current time. A client that sends a badly formatted historical time gets live data with no indication that anything was wrong.

**`BeanFactoryV2`: stops in `stopIds` are not guaranteed to be in the references block.**
[`BeanFactoryV2.java#L1230-L1233`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1230-L1233)
The stop IDs are written without adding the stops to the references block. A stop gets there only indirectly, through an arrival or the nearby list. A found stop with no arrivals and no other stop within 100 metres is left without a reference. Observed: stop `1_312` in a 03:00 query.

**`BeanFactoryV2.isSituationExcludedForApplication()`: always returns `false` when an API key is present.**
[`BeanFactoryV2.java#L1381`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1381)
The same defect as in `arrivals-and-departures-for-stop`: `!_applicationKey.contains(_applicationKey)` is always false. As a result, alerts targeted at specific applications are shown to all callers.

### Implementation defects only

**`StopWithArrivalsAndDeparturesBeanServiceImpl.filter()`: the null-route log statement would throw.**
[`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L237-L240`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L237-L240)
Inside the branch that runs when the route is null, the log statement reads the null route's ID, which would throw a null-pointer exception. In practice the route is never null here, because missing route data is filled in upstream.

**`NearbyStopsBeanServiceImpl.getNearbyStops()`: the stop filter is applied to the wrong stop.**
[`NearbyStopsBeanServiceImpl.java#L68`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/NearbyStopsBeanServiceImpl.java#L68)
The filter checks the source stop instead of each candidate neighbour. This endpoint passes only arrival filters, never stop filters, so the check always passes and nothing is observable.

**`StopWithArrivalsAndDeparturesBeanServiceImpl.StopDistanceComparator`: boxed distances are compared by identity.**
[`StopWithArrivalsAndDeparturesBeanServiceImpl.java#L260`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopWithArrivalsAndDeparturesBeanServiceImpl.java#L260)
The equality check on two boxed distance values compares object identity, not numeric value. Equal distances still fall through to the numeric comparison, so the result is correct.

## Open Questions

- **Server-configured filters and flags.** The system-level stop and arrival filters, the per-agency setting to hide schedule-only data, the per-agency rule for suppressing arrivals the vehicle has already passed, and the primary route-sort agency all come from deployment configuration. The test deployment configures none of them, so their effect on real deployments cannot be observed here.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["key", "lat", "lon"],
  "properties": {
    "key":                    { "type": "string" },
    "lat":                    { "type": "number", "default": 0 },
    "lon":                    { "type": "number", "default": 0 },
    "radius":                 { "type": "number", "default": 0 },
    "latSpan":                { "type": "number", "default": 0 },
    "lonSpan":                { "type": "number", "default": 0 },
    "maxCount":               { "type": "integer", "default": 250 },
    "time":                   { "type": "string" },
    "minutesBefore":          { "type": "integer", "default": 5 },
    "minutesAfter":           { "type": "integer", "default": 35 },
    "frequencyMinutesBefore": { "type": "integer", "default": 2 },
    "frequencyMinutesAfter":  { "type": "integer", "default": 30 },
    "routeType":              { "type": "string" },
    "emptyReturnsNotFound":   { "type": "boolean", "default": false },
    "includeReferences":      { "type": "boolean", "default": true }
  }
}
```

**`key`**: API key. Required.

**`lat`**: Latitude of the search centre, in decimal degrees. Required in practice: if omitted it defaults to 0, and the search then finds nothing.

**`lon`**: Longitude of the search centre, in decimal degrees. Required in practice, for the same reason as `lat`.

**`radius`**: Search radius in metres. The search area is the square that encloses this circle. It takes priority over `latSpan`/`lonSpan` when greater than 0. It is meant to be capped at 20,000 metres, but values above that currently return nothing. Not mentioned in the legacy method documentation.

**`latSpan`**: Total height of the search box in degrees of latitude, centred on `lat`. Used only when `radius` is not supplied, and only when `lonSpan` is also greater than 0.

**`lonSpan`**: Total width of the search box in degrees of longitude, centred on `lon`. The same conditions as `latSpan` apply.

**`maxCount`**: Maximum length of each of `stopIds`, `arrivalsAndDepartures` and `nearbyStopIds`, applied to each list separately. Default 250, silently capped at 1,000. 0 gives empty lists. Negative values cause an error that is not handled.

**`time`**: Query time. Either Unix milliseconds (all digits), or `yyyy-MM-dd_HH-mm-ss` in the server's time zone. Any other value is silently ignored. Defaults to the current server time.

**`minutesBefore`**: Minutes before the query time to include arrivals and departures of regular trips. Default 5.

**`minutesAfter`**: Minutes after the query time to include arrivals and departures of regular trips. Default 35.

**`frequencyMinutesBefore`**: Equivalent of `minutesBefore` for frequency-based trips. Default 2.

**`frequencyMinutesAfter`**: Equivalent of `minutesAfter` for frequency-based trips. Default 30.

**`routeType`**: Comma-separated list of GTFS route types (for example `0,3`). Arrivals are kept when their stop is served by a route of one of these types. Nearby stops are reduced to routes of these types, and dropped if none remain. Non-integer entries are ignored. The stop search itself is not filtered.

**`emptyReturnsNotFound`**: Intended to make an empty result return 404. It currently has no effect.

**`includeReferences`**: When `false`, the references block is returned with every list empty. Default `true`.

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
        "entry":      { "type": "object" },
        "references": { "type": "object" }
      }
    }
  }
}
```

**`code`**: 200 on success.

**`version`**: 2.

**`currentTime`**: Server time when the response was generated.

**`text`**: `"OK"` on success.

**`data.entry`**: The combined departure board; see below.

**`data.references`**: Contains `agencies`, `routes`, `situations`, `stopTimes` (always empty), `stops` and `trips`. It holds full objects for every entity referenced by the arrivals, by their trip statuses (for example the closest and next stops), and by the nearby-stop list. Routes are sorted by short name (lexicographic by default). Agencies are sorted by ID.

---

### data (empty result)

Returned instead of `entry`/`references` when no stops are found or the location is outside the service area (see Extension 5a).

```json
{
  "type": "object",
  "properties": {
    "arrivalsAndDepartures": { "type": "array" },
    "limitExceeded":         { "type": "boolean" },
    "nearbyStops":           { "type": "array" },
    "situations":            { "type": "array" },
    "stops":                 { "type": "array" },
    "timeZone":              { "type": "string" }
  }
}
```

**`data.arrivalsAndDepartures`**, **`data.nearbyStops`**, **`data.situations`**, **`data.stops`**: Always empty arrays.

**`data.limitExceeded`**: Always `false`.

**`data.timeZone`**: Always the empty string.

---

### data.entry

```json
{
  "type": "object",
  "properties": {
    "stopIds":               { "type": "array", "items": { "type": "string" } },
    "arrivalsAndDepartures": { "type": "array", "items": { "type": "object" } },
    "nearbyStopIds":         { "type": "array", "items": { "type": "object" } },
    "situationIds":          { "type": "array", "items": { "type": "string" } },
    "limitExceeded":         { "type": "boolean" }
  }
}
```

**`data.entry.stopIds[]`**: Combined `{agencyId}_{stopId}` IDs of the stops found in the search area. Stops that have arrivals in the window appear twice (a defect). The order is arbitrary but repeatable. Truncated to `maxCount`. Stops are not guaranteed to have an entry in `references.stops`.

**`data.entry.arrivalsAndDepartures[]`**: Arrivals and departures at the found stops, grouped by stop. Isolated stops come first in arbitrary order, then the remaining stops by distance from the search centre. Within each stop, arrivals are sorted by best arrival time. Truncated to `maxCount`.

**`data.entry.nearbyStopIds[]`**: Stops within a 100-metre box of any found stop, including found stops that are near each other. Sorted by `distanceFromQuery`, ascending, and truncated to `maxCount`. Each has a full object in `references.stops`.

**`data.entry.situationIds[]`**: IDs of service alerts that affect any found stop, or any route serving the stops left after truncation. Empty when there are none. Full objects are in `references.situations`.

**`data.entry.limitExceeded`**: `true` if `stopIds`, `arrivalsAndDepartures` or `nearbyStopIds` was truncated.

---

### data.entry.nearbyStopIds[]

```json
{
  "type": "object",
  "properties": {
    "stopId":            { "type": "string" },
    "distanceFromQuery": { "type": "number" }
  }
}
```

**`data.entry.nearbyStopIds[].stopId`**: Combined `{agencyId}_{stopId}` of the nearby stop.

**`data.entry.nearbyStopIds[].distanceFromQuery`**: Straight-line distance in metres from the centre of the search area to the stop.

---

### data.entry.arrivalsAndDepartures[]

Each element has the same structure and meaning as in `arrivals-and-departures-for-stop`. It is summarised here for completeness.

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
    "scheduledArrivalInterval":   { "type": "object" },
    "scheduledDepartureInterval": { "type": "object" },
    "predictedArrivalInterval":   { "type": "object" },
    "predictedDepartureInterval": { "type": "object" },
    "frequency":                  { "type": "object" },
    "status":                     { "type": "string" },
    "predicted":                  { "type": "boolean" },
    "lastUpdateTime":             { "type": "number", "description": "Unix ms" },
    "distanceFromStop":           { "type": "number" },
    "numberOfStopsAway":          { "type": "integer" },
    "occupancyStatus":            { "type": "string" },
    "historicalOccupancy":        { "type": "string" },
    "predictedOccupancy":         { "type": "string" },
    "scheduledTrack":             { "type": "string" },
    "actualTrack":                { "type": "string" },
    "situationIds":               { "type": "array", "items": { "type": "string" } },
    "tripStatus":                 { "type": "object" }
  }
}
```

**`data.entry.arrivalsAndDepartures[].routeId`**: Combined route ID. The full object is in `references.routes`.

**`data.entry.arrivalsAndDepartures[].tripId`**: Combined ID of the trip that will serve the stop. The vehicle may currently be on an earlier trip in the same block, possibly on another route (interlining). `tripStatus.activeTripId` identifies that trip.

**`data.entry.arrivalsAndDepartures[].serviceDate`**: Midnight of the trip's service date, in Unix ms. Trips running after midnight belong to the previous day's service date.

**`data.entry.arrivalsAndDepartures[].vehicleId`**: Combined vehicle ID when real-time data exists, otherwise empty.

**`data.entry.arrivalsAndDepartures[].stopId`**: Combined ID of the found stop this arrival is at.

**`data.entry.arrivalsAndDepartures[].stopSequence`**: 0-based index of this stop within the trip.

**`data.entry.arrivalsAndDepartures[].blockTripSequence`**: 0-based index of this trip within its block.

**`data.entry.arrivalsAndDepartures[].totalStopsInTrip`**: Number of stop visits in the trip.

**`data.entry.arrivalsAndDepartures[].routeShortName`**: Resolved in this order: the arrival-specific short name, then the trip's route short name, then the route's short name.

**`data.entry.arrivalsAndDepartures[].routeLongName`**: The route's long name.

**`data.entry.arrivalsAndDepartures[].tripHeadsign`**: The stop-level headsign if present, otherwise the trip headsign.

**`data.entry.arrivalsAndDepartures[].arrivalEnabled`**: `false` only at the trip's first stop.

**`data.entry.arrivalsAndDepartures[].departureEnabled`**: `false` only at the trip's last stop.

**`data.entry.arrivalsAndDepartures[].scheduledArrivalTime`**, **`data.entry.arrivalsAndDepartures[].scheduledDepartureTime`**: Scheduled times. For frequency-based trips with a prediction, these are replaced by the predicted time.

**`data.entry.arrivalsAndDepartures[].predictedArrivalTime`**, **`data.entry.arrivalsAndDepartures[].predictedDepartureTime`**: Real-time predictions, or `0` when unavailable.

**`data.entry.arrivalsAndDepartures[].scheduledArrivalInterval`**, **`data.entry.arrivalsAndDepartures[].scheduledDepartureInterval`**, **`data.entry.arrivalsAndDepartures[].predictedArrivalInterval`**, **`data.entry.arrivalsAndDepartures[].predictedDepartureInterval`**: Optional `{from, to}` uncertainty windows in Unix ms, or `null`.

**`data.entry.arrivalsAndDepartures[].frequency`**: `{startTime, endTime, headway, exactTimes}` for frequency-based trips, otherwise `null`.

**`data.entry.arrivalsAndDepartures[].status`**: `"default"`, or a real-time status such as `"CANCELED"`.

**`data.entry.arrivalsAndDepartures[].predicted`**: `true` when real-time data was used.

**`data.entry.arrivalsAndDepartures[].lastUpdateTime`**: Time of the last real-time update, or `0`.

**`data.entry.arrivalsAndDepartures[].distanceFromStop`**: Metres along the route between the vehicle (real-time or schedule-projected) and the stop. Negative once the stop has been passed. Omitted when the vehicle's position is unknown.

**`data.entry.arrivalsAndDepartures[].numberOfStopsAway`**: Stops remaining before this stop. Negative once the stop has been passed.

**`data.entry.arrivalsAndDepartures[].occupancyStatus`**, **`data.entry.arrivalsAndDepartures[].historicalOccupancy`**: Occupancy categories, or the empty string.

**`data.entry.arrivalsAndDepartures[].predictedOccupancy`**: This endpoint never fills this field; it is always the empty string.

**`data.entry.arrivalsAndDepartures[].scheduledTrack`**, **`data.entry.arrivalsAndDepartures[].actualTrack`**: Platform or track identifiers, or the empty string.

**`data.entry.arrivalsAndDepartures[].situationIds[]`**: Alerts that affect this specific arrival.

**`data.entry.arrivalsAndDepartures[].tripStatus`**: Status of the vehicle's active trip. It contains the following fields:
- `activeTripId`, `blockTripSequence` and `serviceDate`
- `position` {`lat`, `lon`} and `orientation`
- `closestStop`, `closestStopTimeOffset`, `nextStop` and `nextStopTimeOffset`
- `scheduledDistanceAlongTrip`, `totalDistanceAlongTrip`, `distanceAlongTrip`, `lastKnownDistanceAlongTrip` and `lastKnownOrientation`
- `phase`, `status`, `predicted`, `scheduleDeviation`, `lastUpdateTime` and `lastLocationUpdateTime`
- `vehicleId`, `occupancyStatus`, `occupancyCount`, `occupancyCapacity`, `vehicleFeatures[]`, `situationIds[]` and `frequency`

These have the same meanings as in `arrivals-and-departures-for-stop`. The field is omitted when the vehicle's position is unknown.
