# routes-for-location

## Goal in Context

A client displaying a transit map calls this endpoint to discover which routes serve the area currently visible on the screen. The endpoint returns a list of routes whose stops fall within a specified geographic bounding box, optionally filtered by route short name or long name.

## Scope

The API server.

## Level

User goal.

## Primary Actor

Rider (accessing the system via a client app).

## Stakeholders and Interests

- **Rider**: wants an accurate, up-to-date list of routes serving the specified area so that the client can display them on a map or populate a selection menu.

## Preconditions

The server has a valid GTFS bundle loaded and the geospatial stop index is populated.

## Minimal Guarantees

The response is a JSON object with `version`, `code`, `text`, `currentTime`, and `data` fields regardless of outcome.

## Success Guarantees

The response contains a list of routes whose stops lie within the search bounding box. The list is sorted by route ID and contains at most `maxCount` entries. If additional matching routes were discarded to meet the limit, `data.limitExceeded` is `true`.

## Trigger

An HTTP GET request to `/api/where/routes-for-location.json` with `lat` and `lon` supplied.

## Main Success Scenario

1. The caller supplies `lat` and `lon` as the search centre, and optionally `radius`, `latSpan`/`lonSpan`, `query`, and `maxCount`.
2. The server computes a rectangular bounding box from the supplied parameters:
   - If `radius > 0` is given, the box is the smallest rectangle enclosing a circle of that radius (in metres) centred at the requested coordinates.
   - Otherwise, if both `latSpan > 0` and `lonSpan > 0` are given, the box spans ±(`latSpan`/2) degrees of latitude and ±(`lonSpan`/2) degrees of longitude from the centre.
   - Otherwise (no sizing parameter supplied), the default radius is applied: **500 metres** when no `query` is present, **10 kilometres** when `query` is present. ([`RoutesForLocationAction.getSearchBounds`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/RoutesForLocationAction.java#L150-L165))
3. The server checks whether the bounding box intersects any agency's declared service area. If it does, processing continues. ([`TransitDataServiceTemplateImpl.checkBounds`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/federated/TransitDataServiceTemplateImpl.java#L849-L863))
4. **Without `query`**: the server queries the geospatial stop index for all stops within the bounding box, then collects every route that serves any of those stops, deduplicating across stops. ([`RoutesBeanServiceImpl.getRoutesWithoutRouteNameQuery`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RoutesBeanServiceImpl.java#L152-L168))
5. **With `query`**: the server searches a text index of route short names and long names for routes matching the query string, considering at most `maxCount + 1` candidates by relevance score. For each candidate, it checks whether any stop belonging to that route lies within the bounding box; only routes with at least one in-bounds stop are kept. ([`RoutesBeanServiceImpl.getRoutesWithRouteNameQuery`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RoutesBeanServiceImpl.java#L170-L198), [`RouteCollectionSearchServiceImpl.searchForRoutesByName`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/RouteCollectionSearchServiceImpl.java#L95-L101))
6. If the number of matching routes exceeds `maxCount`, the list is randomly shuffled and truncated to `maxCount` entries, and `data.limitExceeded` is set to `true`. ([`BeanServiceSupport.checkLimitExceeded`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/BeanServiceSupport.java#L23-L31))
7. The retained routes are sorted by their combined route ID in ascending lexicographic order. ([`RoutesBeanServiceImpl.constructResult`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RoutesBeanServiceImpl.java#L214-L223))
8. The server returns HTTP 200 with `data.list` containing the route objects, `data.outOfRange` set to `false`, and `data.references.agencies` populated with the agencies owning the returned routes.

## Extensions

**1a. `maxCount` is zero or negative:**
The server returns HTTP 400. The response body is a JSON object with a `fieldErrors` map whose `maxCount` entry contains the message `"must be greater than zero"`. ([`RoutesForLocationAction.index`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/RoutesForLocationAction.java#L102-L107))

**3a. The bounding box does not intersect any agency's service area:**
The server returns HTTP 200 with `data.list` as an empty array, `data.limitExceeded` as `false`, and `data.outOfRange` as `true`. ([`RoutesForLocationAction.transformOutOfRangeResult`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/RoutesForLocationAction.java#L139-L148))

**4a–5a. No routes match the location (and query, if given):**
The server returns HTTP 200 with `data.list` as an empty array and `data.limitExceeded` as `false`.

**5a-i. The text index is unavailable (search index not built):**
The text search returns an empty result set; no routes pass the location filter. The response is an empty list with HTTP 200. ([`RouteCollectionSearchServiceImpl.search`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/RouteCollectionSearchServiceImpl.java#L107-L108))

**5a-ii. The `query` value is a stop word (e.g. `"the"`, `"in"`):**
The text index uses a standard English stop-word list. A query consisting entirely of stop words is parsed to an empty query and returns no results. ([`RouteCollectionSearchServiceImpl`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/RouteCollectionSearchServiceImpl.java#L58-L100))

**5a-iii. The `query` value contains Lucene special characters (e.g. `+`, `-`, `[`, `]`):**
The query parser may throw a `ParseException`. The server propagates this as an `InvalidArgumentServiceException` with field `"query"` and code `"queryParseError"`. ([`RoutesBeanServiceImpl.searchForRoutes`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RoutesBeanServiceImpl.java#L201-L212))

**`maxCount` exceeds the absolute maximum:**
The server silently clamps `maxCount` to 50. No error is returned. ([`MaxCountSupport.getMaxCount`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/impl/MaxCountSupport.java#L41-L43))

## Suspected Defects

### Defects that affect the use case

**1. Non-deterministic results when the result set is truncated — `BeanServiceSupport.checkLimitExceeded`**

[`BeanServiceSupport.java#L26-L28`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/BeanServiceSupport.java#L26-L28)

When the number of matching routes exceeds `maxCount`, the list is randomly shuffled before the tail is discarded. Repeated requests with identical parameters return different subsets of routes. The `limitExceeded` flag tells callers that truncation occurred but does not convey that the selection is random. The likely intended behaviour is a deterministic selection — for example, retaining the routes nearest to the search centre, or the routes with the shortest IDs — so that clients can paginate or display consistent results.

**2. `nullSafeShortName` exposed as a response field — `RouteV2Bean.getNullSafeShortName`**

[`RouteV2Bean.java#L62-L66`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/RouteV2Bean.java#L62-L66)

A convenience accessor is serialised as a first-class JSON field named `nullSafeShortName`. When `shortName` is present it duplicates it; when `shortName` is absent it falls back to the route `id`. This field was intended as an internal helper, not a public API surface. Callers already receive both `shortName` and `id` and can apply the same fallback themselves. Removing the field would break any client currently relying on it.

### Implementation defects only

**1. Dead query-type assignment — `RoutesForLocationAction.index`**

[`RoutesForLocationAction.java#L118`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/RoutesForLocationAction.java#L118)

The action sets a query type on the `SearchQueryBean`, but `RoutesBeanServiceImpl.getRoutesForQuery` does not read this field. The service branches solely on whether `query.getQuery()` is null. The type assignment has no effect and need not be replicated in a Go implementation.

## Open Questions

None.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["lat", "lon"],
  "properties": {
    "lat":      { "type": "number", "description": "Latitude of the search centre" },
    "lon":      { "type": "number", "description": "Longitude of the search centre" },
    "radius":   { "type": "number", "description": "Search radius in metres; constructs a bounding box enclosing this circle" },
    "latSpan":  { "type": "number", "description": "Total latitudinal span of the search box in degrees; the box extends ±(latSpan/2) from the centre" },
    "lonSpan":  { "type": "number", "description": "Total longitudinal span of the search box in degrees; the box extends ±(lonSpan/2) from the centre" },
    "query":    { "type": "string",  "description": "Text search string matched against route short names and long names" },
    "maxCount": { "type": "integer", "default": 10,  "description": "Maximum number of routes to return; silently clamped to 50" }
  }
}
```

**`lat`** — Decimal latitude of the geographic centre of the search area. Required.

**`lon`** — Decimal longitude of the geographic centre of the search area. Required.

**`radius`** — Search radius in metres. If positive, a bounding box enclosing a circle of this radius is used as the search area. Mutually exclusive with `latSpan`/`lonSpan` in the sense that `radius` takes precedence when positive.

**`latSpan`** — Total latitudinal extent of the search bounding box in decimal degrees. The box spans from `lat − latSpan/2` to `lat + latSpan/2`. Both `latSpan` and `lonSpan` must be positive for this parameter to take effect. Ignored if `radius > 0`.

**`lonSpan`** — Total longitudinal extent of the search bounding box in decimal degrees. The box spans from `lon − lonSpan/2` to `lon + lonSpan/2`. Both `latSpan` and `lonSpan` must be positive for this parameter to take effect. Ignored if `radius > 0`.

**`query`** — Optional text search string. When supplied, only routes whose short name or long name matches this string (via a scored text index) are returned. Also changes the default search radius to 10 kilometres. The search matches partial tokens; Lucene syntax special characters in the value may cause a parse error (see extension 5a-iii).

**`maxCount`** — Maximum number of routes to include in the response list. Default is 10. Any value above 50 is silently reduced to 50. A value of zero or below causes a validation error.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "version":     { "type": "integer", "description": "API version; always 2" },
    "code":        { "type": "integer", "description": "HTTP status code mirrored in the body" },
    "text":        { "type": "string",  "description": "Human-readable status message" },
    "currentTime": { "type": "integer", "description": "Unix ms; server wall-clock time at response construction" },
    "data":        { "type": "object" }
  }
}
```

**`version`** — Always `2` for this endpoint.

**`code`** — `200` for all outcomes except validation errors (which return `400` via HTTP status; the body in that case does not follow the standard envelope).

**`text`** — `"OK"` on success.

**`currentTime`** — Server wall-clock time in Unix milliseconds at the moment the response bean was constructed.

**`data`** — The payload object; structure varies by outcome (see below for the success shape).

---

### data (success)

```json
{
  "type": "object",
  "properties": {
    "limitExceeded": { "type": "boolean" },
    "outOfRange":    { "type": "boolean" },
    "list":          { "type": "array", "items": { "type": "object" } },
    "references":    { "type": "object" }
  }
}
```

**`data.limitExceeded`** — `true` when the total number of matching routes exceeded `maxCount` and the list has been truncated. When `true`, the specific routes returned are a randomly selected subset of all matches; repeated requests may return different routes.

**`data.outOfRange`** — `true` when the search coordinates fall outside every agency's declared service area. When `true`, `list` is empty and `limitExceeded` is `false`.

**`data.list`** — Array of route objects sorted by route ID in ascending lexicographic order (see `data.list[]` schema below).

**`data.references`** — Object containing referenced entities. For this endpoint only `agencies` is populated; all other arrays (`routes`, `stops`, `trips`, `stopTimes`, `situations`) are present but empty.

---

### data.list[]

```json
{
  "type": "object",
  "required": ["id", "type", "agencyId"],
  "properties": {
    "id":                { "type": "string"  },
    "shortName":         { "type": "string"  },
    "longName":          { "type": "string"  },
    "description":       { "type": "string"  },
    "type":              { "type": "integer" },
    "url":               { "type": "string"  },
    "color":             { "type": "string"  },
    "textColor":         { "type": "string"  },
    "agencyId":          { "type": "string"  },
    "nullSafeShortName": { "type": "string"  }
  }
}
```

**`data.list[].id`** — Combined route ID in `{agencyId}_{entityId}` form, e.g. `"1_100224"`. Globally unique across all agencies in the feed.

**`data.list[].shortName`** — Route short name as published in the GTFS feed (e.g. `"44"`). May be absent if the agency publishes only a long name.

**`data.list[].longName`** — Route long name (e.g. `"Ballard - Montlake"`). May be absent or an empty string if the agency publishes only a short name.

**`data.list[].description`** — Route description from the GTFS feed. May be absent or empty.

**`data.list[].type`** — GTFS route type integer (e.g. `3` = bus, `0` = tram/light rail, `2` = rail, `4` = ferry).

**`data.list[].url`** — URL to the agency's page for this route. May be absent or empty.

**`data.list[].color`** — Route colour as a six-digit hex string without a leading `#` (e.g. `"FDB71A"`). May be absent or empty.

**`data.list[].textColor`** — Colour for text displayed against the route colour, also a six-digit hex string. May be absent or empty.

**`data.list[].agencyId`** — Plain agency ID string (not the combined form), e.g. `"1"`. Matches an entry in `data.references.agencies`.

**`data.list[].nullSafeShortName`** — Equals `shortName` when that field is present and non-null; otherwise equals `id`. This is a computed convenience field serialised by the Java implementation; see Suspected Defects.

---

### data.references.agencies[]

```json
{
  "type": "object",
  "required": ["id", "name", "timezone", "url"],
  "properties": {
    "id":             { "type": "string"  },
    "name":           { "type": "string"  },
    "url":            { "type": "string"  },
    "timezone":       { "type": "string"  },
    "lang":           { "type": "string"  },
    "phone":          { "type": "string"  },
    "fareUrl":        { "type": "string"  },
    "email":          { "type": "string"  },
    "disclaimer":     { "type": "string"  },
    "privateService": { "type": "boolean" }
  }
}
```

**`data.references.agencies[].id`** — Plain agency ID string. Matches `agencyId` fields on route objects in `data.list`.

**`data.references.agencies[].name`** — Human-readable agency name.

**`data.references.agencies[].url`** — Agency website URL.

**`data.references.agencies[].timezone`** — IANA timezone string for the agency (e.g. `"America/Los_Angeles"`).

**`data.references.agencies[].lang`** — ISO 639-1 language code for the agency's primary language. May be empty.

**`data.references.agencies[].phone`** — Agency customer service phone number. May be empty.

**`data.references.agencies[].fareUrl`** — URL for fare information. May be empty.

**`data.references.agencies[].email`** — Agency contact email address. May be empty.

**`data.references.agencies[].disclaimer`** — Legal disclaimer text. May be empty.

**`data.references.agencies[].privateService`** — `true` if the agency operates private (non-public) service.
