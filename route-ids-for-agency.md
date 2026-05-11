# route-ids-for-agency

## Goal in Context

A client application needs the complete set of route identifiers operated by a specific agency so it can construct follow-up requests (such as fetching timetables or stop lists) without first enumerating every possible route.

## Scope

OneBusAway REST API — `GET /api/where/route-ids-for-agency/{id}`

## Level

User goal

## Primary Actor

Rider (via a client application)

## Stakeholders and Interests

- **Rider** — wants a complete, correct list of route identifiers so that follow-up lookups return meaningful results without silent gaps.

## Preconditions

- The server holds a transit data bundle containing at least one agency.
- The caller supplies a valid API key.

## Minimal Guarantees

- The response envelope always carries a `version`, `code`, `currentTime`, and `text` field.

## Success Guarantees

- The response body contains a list of all route identifiers operated by the specified agency, each in combined `{agencyId}_{entityId}` form.
- The `limitExceeded` flag is `false`.
- The `references` block is present but all its sub-lists are empty (route objects are intentionally omitted to avoid transmitting potentially large payloads).

## Trigger

An HTTP GET request to `/api/where/route-ids-for-agency/{id}.json?key=<key>` where `{id}` is the bare agency identifier (e.g. `1`, `40`).

## Main Success Scenario

1. The caller issues `GET /api/where/route-ids-for-agency/{id}.json?key=<key>`.
2. The server validates the API key via the key interceptor.
3. The server validates that `id` is present; returns HTTP 400 if absent. ([`RouteIdsForAgencyAction.java#L42`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/RouteIdsForAgencyAction.java#L42))
4. The server looks up the agency record for `id` and iterates over its associated route collections, converting each route collection identifier to combined `{agencyId}_{entityId}` string form. ([`RoutesBeanServiceImpl.java#L121–L131`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RoutesBeanServiceImpl.java#L121))
5. The server returns HTTP 200 with the full list of route identifiers. No upper bound is applied; `limitExceeded` is always `false`. ([`RoutesBeanServiceImpl.java#L130`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RoutesBeanServiceImpl.java#L130))
6. The `references` block is included in the response but all sub-lists are empty — route objects are deliberately not populated. ([`RouteIdsForAgencyAction.java#L59–L62`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/RouteIdsForAgencyAction.java#L59))

## Extensions

**3a. Missing `id` parameter:**
The `@RequiredFieldValidator` fires; the server returns HTTP 400 with a validation error body. ([`RouteIdsForAgencyAction.java#L42`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/RouteIdsForAgencyAction.java#L42))

**4a. Unknown agency identifier:**
The service layer raises a "no such agency" error. Due to a defect in how this exception propagates through the remote call layer, the server returns HTTP 200 with the response body `null` instead of a proper error. See Suspected Defects.

**4b. Agency exists but has no routes:**
The route collection list for the agency is empty; the server returns HTTP 200 with an empty `list` array.

## Suspected Defects

### Defects that affect the use case

**Unknown agency returns HTTP 200 with null body instead of a 404 error.**

`RoutesBeanServiceImpl` ([`RoutesBeanServiceImpl.java#L122–L124`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/RoutesBeanServiceImpl.java#L122)) throws `NoSuchAgencyServiceException` when the agency is not found. The `ExceptionInterceptor` ([`ExceptionInterceptor.java#L69–L84`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ExceptionInterceptor.java#L69)) handles `NoSuchStopServiceException`, `NoSuchTripServiceException`, and `NoSuchRouteServiceException` explicitly (returning 404), and all other exceptions with a 500, but the `NoSuchAgencyServiceException` does not surface correctly through the Hessian RPC layer that separates the API webapp from the federation service. Instead of producing a 404 or 500, the server responds with HTTP 200 and the literal body `null`. Any client calling this endpoint with an unknown agency receives a malformed successful response with no parseable data.

The intended behaviour is a 404 response with `code: 404` and an appropriate error message, consistent with how `NoSuchStopServiceException`, `NoSuchTripServiceException`, and `NoSuchRouteServiceException` are handled.

### Implementation defects only

None identified.

## Open Questions

None.

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Path parameter. The bare agency identifier (e.g. '1', '40'). Not in combined agencyId_entityId form — the agency ID itself is the full identifier."
    },
    "key": {
      "type": "string",
      "description": "API authentication key."
    },
    "version": {
      "type": "integer",
      "default": 2,
      "description": "API version selector. Must be 2; any other value returns a 500 error."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "Whether to populate the references block with related entity objects. Has no observable effect on this endpoint because route objects are never added to references regardless of this setting."
    }
  }
}
```

**`id`** — The plain agency identifier. Must match an agency known to the server. Supplied as a URL path segment: `/api/where/route-ids-for-agency/{id}.json`.

**`key`** — API authentication key. Validated by the key interceptor before any business logic runs.

**`version`** — API version number. Defaults to 2 when omitted. Passing any value other than 2 causes the server to return a 500 error with the message "unknown version: {n}".

**`includeReferences`** — Controls whether the response's `references` block is populated with full entity objects for any IDs mentioned in the response. This endpoint never populates references with route objects (by design), so the setting has no visible effect.

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "version": { "type": "integer" },
    "code":    { "type": "integer" },
    "text":    { "type": "string" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "data": { "type": "object" }
  }
}
```

**`version`** — Echo of the API version negotiated for the request (always `2`).

**`code`** — HTTP-style status code mirrored inside the body (`200` on success).

**`text`** — Human-readable status message (e.g. `"OK"`).

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`data`** — The payload object described below.

### data

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

**`data.list`** — Array of route identifiers, each in combined `{agencyId}_{entityId}` form (e.g. `"1_100016"`). The ordering is not specified and should not be relied upon. The result is the full set of routes for the agency; no truncation is applied.

**`data.limitExceeded`** — Always `false`. No upper bound is imposed on the number of returned identifiers.

**`data.references`** — Always present. Contains empty sub-arrays for `agencies`, `routes`, `stops`, `trips`, `situations`, and `stopTimes`. Route objects are intentionally omitted from this block even though their IDs appear in `data.list`.
