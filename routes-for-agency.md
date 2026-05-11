# routes-for-agency / route-ids-for-agency

## Goal in Context

A rider's client application needs to enumerate all routes operated by a specific transit agency — either as complete route records (for display or caching) or as a lightweight list of route identifiers (for set membership checks or pre-flight discovery).

## Scope

OneBusAway REST API, agency-scoped route enumeration.

## Level

User goal.

## Primary Actor

Rider (via client application).

## Stakeholders and Interests

- **Rider**: Wants a complete, accurate list of routes so the client can present a route picker, populate a cache, or determine which routes to query further.

## Preconditions

- The caller knows a valid agency ID, typically obtained from `agencies-with-coverage`.
- The API key is present and authorised.

## Minimal Guarantees

- The response envelope is well-formed JSON with `version`, `code`, `currentTime`, `text`, and `data` fields.

## Success Guarantees

- Every route belonging to the requested agency is present in the response list exactly once.
- For the full-route variant, each entry carries the complete route record including name, type, colour branding, and owning agency ID.
- For the IDs-only variant, each entry is the combined route ID string.
- The owning agency record appears in the references block for the full-route variant.

## Trigger

An HTTP GET request to either:
- `GET /api/where/routes-for-agency/{agencyId}.json` — full route records
- `GET /api/where/route-ids-for-agency/{agencyId}.json` — combined route IDs only

## Main Success Scenario

1. The rider's client sends a GET request with the plain agency ID in the path and a valid API key.
2. The system looks up the agency in the transit graph by the supplied ID. ([`RoutesBeanServiceImpl.java#L136`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RoutesBeanServiceImpl.java#L136))
3. The system iterates over every route collection belonging to that agency. ([`RoutesBeanServiceImpl.java#L140`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RoutesBeanServiceImpl.java#L140))
4. **Full-route variant**: For each route collection the system retrieves the narrative (short name, long name, description, colour, type, URL) and the owning agency record, assembles a route bean, and accumulates the owning agency into the references block. ([`BeanFactoryV2.java#L432-L448`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L432-L448))
   **IDs-only variant**: For each route collection the system converts the internal ID to a combined string; no references are populated. ([`RoutesBeanServiceImpl.java#L121-L131`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RoutesBeanServiceImpl.java#L121-L131))
5. The system returns an HTTP 200 response. `data.limitExceeded` is always `false`; no truncation is applied. ([`RoutesBeanServiceImpl.java#L145`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RoutesBeanServiceImpl.java#L145))

**Ordering**: Results are returned in the order the routes appear in the internal transit graph, which reflects the GTFS data loading order. No alphabetic or numeric sort is applied; callers that need a stable display order must sort client-side.

**Caching**: Both service methods are cached in the federation layer, so repeated requests return identical results until the bundle is refreshed.

## Extensions

**3a. Agency ID is not found in the transit graph.**

The service throws an agency-not-found exception. Due to a defect in the exception interceptor (see Suspected Defects), the HTTP status remains 200 and the response body is the literal JSON value `null` rather than a structured error. Callers must treat a `null` body as an agency-not-found condition.

**1b. `id` path segment is absent.**

The Struts2 validation layer rejects the request before the action executes and returns HTTP 400 with `code: 400` and field-level validation errors in `data`.

**1c. A non-2 version is explicitly requested via the `version` query parameter.**

The action returns HTTP 500 with `code: 500` and `text: "unknown version: <n>"`.

## Suspected Defects

### Defects that affect the use case

**Unknown agency returns HTTP 200 with null body instead of HTTP 404.**

`ExceptionInterceptor` ([`ExceptionInterceptor.java#L68-L85`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ExceptionInterceptor.java#L68-L85)) maps `NoSuchStopServiceException`, `NoSuchTripServiceException`, and `NoSuchRouteServiceException` to HTTP 404, but `NoSuchAgencyServiceException` is absent from this list. The exception falls into the `else` branch, which attempts to produce an HTTP 500 response, but due to the interaction between the Struts2 REST content-type handler and the action's null model at exception time, the observable outcome is HTTP 200 with the body `null`. The intended behaviour is HTTP 404 with a standard resource-not-found envelope (`code: 404, text: "resource not found"`).

**`nullSafeShortName` appears as a route field in the response.**

`RouteV2Bean` ([`RouteV2Bean.java#L62-L66`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/RouteV2Bean.java#L62-L66)) exposes a `getNullSafeShortName()` getter intended for internal route sorting. Because the JSON serialiser reflects all public bean getters, this synthetic field appears in every route object as `nullSafeShortName`. Its value is the route's `shortName` when set, otherwise the full combined route ID. This field is not part of the intended API contract and should not appear in responses. The Go implementation should omit it.

### Implementation defects only

**Exception response version is hard-coded to 1.**

`ExceptionInterceptor` ([`ExceptionInterceptor.java#L45`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ExceptionInterceptor.java#L45)) always constructs exception `ResponseBean` objects with `version = 1`, regardless of the version the action would have returned. This does not affect callers today (because of the null-body defect above), but any clean reimplementation of error handling should use the negotiated version consistently.

## Open Questions

None.

## Request Parameters

```json
{
  "type": "object",
  "required": ["id"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Path parameter. Plain agency ID (not combined form). Example: \"1\""
    },
    "key": {
      "type": "string",
      "description": "API key."
    },
    "version": {
      "type": "integer",
      "description": "API version. Only 2 is supported; omitting this parameter defaults to 2."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "When false, the references block is returned empty."
    }
  }
}
```

**`id`** — Plain agency ID identifying the operator whose routes are requested. Agency IDs are not in combined form; supply the bare string exactly as returned by `agencies-with-coverage` (e.g., `1`).

**`key`** — API authentication key.

**`version`** — Selects the API response version. Only version 2 is implemented; any other explicit value produces an error.

**`includeReferences`** — Controls whether the `references` block is populated. Defaults to `true`. Set to `false` to suppress reference data and reduce response size.

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

**`version`** — Response version; `2` on success.

**`code`** — HTTP status code mirrored in the body; `200` on success.

**`text`** — Human-readable status; `"OK"` on success.

**`currentTime`** — Server time at response generation, in Unix milliseconds.

**`data`** — Container for the list and references (see below).

---

### `data` — routes-for-agency

```json
{
  "type": "object",
  "properties": {
    "list": {
      "type": "array",
      "items": { "$ref": "#/definitions/route" }
    },
    "limitExceeded": { "type": "boolean" },
    "references": { "type": "object" }
  }
}
```

**`data.list`** — Ordered array of route objects, one per route belonging to the requested agency. No truncation is applied; all routes are always returned.

**`data.limitExceeded`** — Always `false`; included for envelope consistency.

**`data.references`** — Contains a single-element `agencies` array with the owning agency record. All other reference sub-arrays (`routes`, `stops`, `trips`, `situations`, `stopTimes`) are empty for this endpoint.

---

### `data.list[]` — route object (routes-for-agency)

```json
{
  "type": "object",
  "required": ["id", "agencyId", "type"],
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

**`data.list[].id`** — Combined route ID in `{agencyId}_{routeId}` form (e.g., `"1_102718"`).

**`data.list[].agencyId`** — Plain agency ID (not combined). Identifies the operator that owns this route.

**`data.list[].shortName`** — The route's short public name (e.g., `"150"`). May be absent or `null` if the agency provides only a long name or description.

**`data.list[].longName`** — The route's full public name (e.g., `"University District / Northgate"`). May be absent, `null`, or an empty string.

**`data.list[].description`** — A descriptive label for the route, sometimes used in place of a long name. May be absent, `null`, or an empty string.

**`data.list[].type`** — GTFS route type integer (e.g., `3` = bus, `0` = tram/streetcar, `1` = subway/metro, `2` = rail).

**`data.list[].url`** — URL of the agency's schedule page for this route. May be absent, `null`, or an empty string.

**`data.list[].color`** — Route colour as a six-character hex string without a leading `#` (e.g., `"FDB71A"`). May be absent, `null`, or an empty string.

**`data.list[].textColor`** — Colour for text drawn against the route background, same hex format. May be absent, `null`, or an empty string.

---

### `data.references.agencies[]` (routes-for-agency)

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

**`data.references.agencies[].name`** — Full agency name (e.g., `"Metro Transit"`).

**`data.references.agencies[].url`** — Agency website URL.

**`data.references.agencies[].timezone`** — IANA timezone string (e.g., `"America/Los_Angeles"`).

**`data.references.agencies[].lang`** — Primary language code (e.g., `"en"`).

**`data.references.agencies[].phone`** — Agency contact phone number. May be empty.

**`data.references.agencies[].fareUrl`** — URL for fare information. May be empty.

**`data.references.agencies[].email`** — Agency contact email. May be empty.

**`data.references.agencies[].disclaimer`** — Agency disclaimer text. May be empty.

**`data.references.agencies[].privateService`** — Whether the agency provides private (non-public) service.

---

### `data` — route-ids-for-agency

```json
{
  "type": "object",
  "properties": {
    "list": {
      "type": "array",
      "items": { "type": "string" }
    },
    "limitExceeded": { "type": "boolean" },
    "references": { "type": "object" }
  }
}
```

**`data.list`** — Ordered array of combined route ID strings (e.g., `["1_102718", "1_102717", ...]`). The order matches the GTFS data loading order; no alphabetic or numeric sort is applied.

**`data.limitExceeded`** — Always `false`.

**`data.references`** — All sub-arrays are empty. Route objects are intentionally not included in the references block for this endpoint, since the list may be large and the caller is assumed to already know the route IDs.
