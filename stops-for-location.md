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
   - If `radius` is provided and greater than zero, the box encloses a circle of that radius centred on `lat`/`lon`. ([`StopsForLocationAction.java#L187`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L187))
   - Otherwise, if both `latSpan` and `lonSpan` are provided and greater than zero, the box is centred on `lat`/`lon` with half-dimensions `latSpan/2` and `lonSpan/2`. ([`StopsForLocationAction.java#L189–L191`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L189))
   - Otherwise, if `query` is provided, a 10,000-metre radius is used. ([`StopsForLocationAction.java#L194–L196`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L194))
   - Otherwise, a 500-metre radius is used. ([`StopsForLocationAction.java#L197–L199`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L197))

3. The server checks whether the derived bounds intersect at least one agency's coverage area. If not, it returns an empty list with `outOfRange: true`. ([`TransitDataServiceTemplateImpl.java#L849–L863`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/federated/TransitDataServiceTemplateImpl.java#L849))

4. The server establishes a time context. If `time` is provided, it is used; otherwise the current server time is used. The time is snapped to a 15-minute bin for caching purposes. This time context determines which routes are considered actively serving each stop. ([`ApiIntervalFactory.java#L45–L52`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/services/ApiIntervalFactory.java#L45))

5. **Without `query`**: The server retrieves all stops from the geospatial index that fall within the search bounds. ([`WhereGeospatialServiceImpl.java#L85–L100`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/WhereGeospatialServiceImpl.java#L85))

   - If the raw candidate count exceeds `maxCount` and `routeType` is not specified, the candidate list is randomly shuffled and truncated to `maxCount` before any further processing; `limitExceeded` is set to `true`. This truncation happens before route filtering, so the final returned count may be lower than `maxCount`. ([`StopsBeanServiceImpl.java#L130–L134`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L130))

   - For each candidate stop, the server loads its full stop record, populating `routeIds` with only those routes that have scheduled service within the 15-minute time window. Stops that have no routes active in that window are excluded from results. ([`StopsBeanServiceImpl.java#L143–L161`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L143))

   - If `routeType` is specified, the server additionally filters the remaining stops to keep only those served by at least one route of one of the requested GTFS route types, checking against the time-filtered route set. ([`StopsBeanServiceImpl.java#L163–L168`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L163))

6. **With `query`**: The server searches a stop-code index for stops whose code matches the query string, considering at most 10 candidates. ([`StopsBeanServiceImpl.java#L191`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L191))

   - From those candidates, only those whose coordinates fall within the derived search bounds are kept. ([`StopsBeanServiceImpl.java#L204–L206`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L204))

   - If more stops match than `maxCount`, the matched stops are randomly shuffled and truncated to `maxCount`; `limitExceeded` is set to `true`. ([`BeanServiceSupport.java#L23–L31`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/BeanServiceSupport.java#L23))

   - If no candidates fall within the bounds, the single geographically closest candidate from the full set of 10 is returned regardless of its distance from the centre point. ([`StopsBeanServiceImpl.java#L228–L229`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L228))

   - Routes for each stop are loaded without time filtering: `routeIds` includes all routes associated with the stop across all service dates.

7. The result list is sorted lexicographically by combined stop ID. ([`StopsBeanServiceImpl.java#L274`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L274))

8. The server builds the response envelope. Each stop entry in the list has its associated routes and agencies added to the `references` block (unless `includeReferences=false`).

9. The server returns HTTP 200 with the complete response.

## Extensions

**2a. maxCount is not positive:**
The server returns HTTP 400 with a field-level validation message: `{"fieldErrors":{"maxCount":["must be greater than zero"]}}`. ([`StopsForLocationAction.java#L125–L129`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L125))

**2b. maxCount exceeds the absolute cap of 250:**
The effective limit is silently capped at 250. The caller's value is reduced to 250 without any error or indication in the response. ([`MaxCountSupport.java#L37–L39`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/impl/MaxCountSupport.java#L37))

**3a. Search bounds do not intersect any agency coverage area:**
The server returns HTTP 200 with `outOfRange: true` and an empty `list`. ([`StopsForLocationAction.java#L158–L160`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForLocationAction.java#L158))

**5a. No stops in the area have any active routes for the given time (without-query mode):**
All candidates are filtered out in step 5. The server returns an empty list with `limitExceeded: false` and `outOfRange: false`.

**6a. Query matches no stop codes anywhere in the feed:**
The stop-code search returns no candidates. No closest-fallback is possible. The server returns an empty list.

**8a. includeReferences=false:**
The references block is present in the response but all its sub-arrays (`routes`, `agencies`, `stops`, `trips`, `situations`) are empty.

## Suspected Defects

### Defects that affect the use case

**1. routeType filter does not truncate results when the limit is exceeded.**

`StopsBeanServiceImpl` ([`StopsBeanServiceImpl.java#L171–L177`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L171))

When `routeType` is specified, the code correctly sets `limitExceeded = true` when the post-filter count exceeds `maxCount`, but the list is passed to `constructResult` without being truncated. The comment in the code ("constructResults will perform the truncation if necessary") is incorrect: `constructResult` only sorts by stop ID. As a result, a caller requesting `maxCount=10` with `routeType=3` may receive hundreds of stops.

**2. The `time` parameter has no effect on route filtering when `query` is provided.**

`StopsBeanServiceImpl` ([`StopsBeanServiceImpl.java#L203`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L203))

In the stop-code search path, `null` is passed as the service interval when loading each stop's routes, so `routeIds` reflects all routes across all service dates regardless of the `time` parameter. In the geospatial search path (without `query`), the time context is passed and routes are filtered accordingly. The inconsistency means a caller cannot rely on `routeIds` to reflect time-active routes when supplying a stop-code query.

**3. The `routeType` filter is silently ignored when `query` is also provided.**

`StopsBeanServiceImpl` ([`StopsBeanServiceImpl.java#L180–L232`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L180))

When both `query` and `routeType` are provided, the route-type filter is added to the search query's filter chain by the action class but `getStopsByBoundsAndQuery` never evaluates that filter chain. All stops matching the stop-code query are returned regardless of their route types.

### Implementation defects only

**4. Limit check in the geospatial path runs before route filtering.**

`StopsBeanServiceImpl` ([`StopsBeanServiceImpl.java#L130–L134`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd68b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L130))

When `routeType` is not specified, the raw geospatial candidate IDs are shuffled and truncated before stop beans are loaded and filtered by active routes. This means `limitExceeded=true` is set based on the raw count in the area, not the count after route filtering. A clean reimplementation would not have this ordering issue.

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

**`query`** — A stop code to match against. When supplied, the search uses a text index rather than a pure geospatial lookup. At most 10 code matches are considered, and the results are filtered to those within the derived bounds. If none fall within the bounds, the geographically closest code match is returned as a fallback. Note that `time` has no effect on route filtering in this mode, and `routeType` is silently ignored.

**`maxCount`** — Maximum number of stops to return. Default 100. The server silently caps this at 250; values above 250 are treated as 250. A value of zero or less causes a 400 validation error. When the geospatial candidate pool exceeds this limit (without `routeType`), the pool is randomly shuffled before truncation, making results non-deterministic.

**`routeType`** — One or more GTFS route type integers, comma-separated. Restricts results to stops served by at least one route of a listed type. The type is matched against the routes active for the `time` context. When combined with `query`, this parameter is silently ignored (see Suspected Defects).

**`time`** — Reference time for determining which routes are actively serving each stop. Accepts either a Unix millisecond timestamp as a plain integer string, or a formatted string in the pattern `yyyy-MM-dd_HH-mm-ss`. Defaults to the current server time. Has no effect on route filtering when `query` is supplied.

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

**`data.limitExceeded`** — `true` if the candidate pool in the area exceeded `maxCount`. When `true` without `routeType`, the result was randomly sampled; repeated requests may return different stops. When `true` with `routeType`, all matching stops are returned despite the flag (see Suspected Defects).

**`data.outOfRange`** — `true` when the search area does not intersect any agency's coverage area. The list is empty when this flag is `true`.

**`data.references`** — Deduplicated objects referenced by IDs in the list. Contains sub-arrays `agencies`, `routes`, `stops`, `trips`, `situations`, and `stopTimes`. All are empty when `includeReferences=false`.

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
