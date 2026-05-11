# Use Case: Retrieve Stop IDs for Agency

## Goal in Context

A rider's client application wants the complete set of stop identifiers served by a specific transit agency, without needing full stop details, so it can efficiently enumerate stops for further lookup or local caching.

## Scope

OBA REST API — `GET /api/where/stop-ids-for-agency/{id}`

## Level

User goal

## Primary Actor

Rider (via a client application)

## Stakeholders and Interests

- **Rider** — wants a lightweight list of all stop IDs for a given agency so the client can query individual stops or build a local index without fetching full stop beans for every stop.

## Preconditions

- A valid API key is supplied.
- The transit data bundle is loaded and the server is ready.

## Minimal Guarantees

- All responses include a well-formed JSON envelope with `version`, `code`, `currentTime`, and `text` fields.
- An invalid or missing API key is rejected with HTTP 401 before any agency lookup is attempted.

## Success Guarantees

- The response body contains a list of all stop IDs for stops associated with the requested agency, each in combined `{agencyId}_{entityId}` form.
- The `limitExceeded` flag is present and is always `false`.
- The `references` block is present and empty (no stop beans are materialised for a stop-IDs-only response).

## Trigger

A client issues `GET /api/where/stop-ids-for-agency/{id}.json?key={apiKey}`.

## Main Success Scenario

1. The framework validates that the API key is present; if absent the request is rejected with HTTP 401.
2. The framework validates that a non-empty agency ID is present in the path; if absent the request is rejected with HTTP 400.
3. The server resolves the agency ID to an agency record in the transit graph. The ID is a plain agency identifier — not the `{agencyId}_{entityId}` combined form used for stops, routes, and trips.
4. For each stop associated with the agency, the server converts the stop's internal `{agencyId}_{entityId}` identifier into its string form and adds it to the result list. The list order reflects the order in which stops are stored in the transit graph for that agency; no explicit sorting is applied. ([`StopsBeanServiceImpl.java` L259–268](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L259-L268))
5. The server returns HTTP 200 with a list-with-references envelope. `limitExceeded` is always `false` — no cap is applied to this result set. The references block is always returned but always empty because no entity beans are constructed; only string IDs are collected.

## Extensions

**2a. API key is missing or invalid:**
- The server returns HTTP 401 with `code: 401` and `text: "permission denied"`. No agency lookup is attempted.

**2b. Agency ID path segment is missing or empty:**
- The Struts2 `@RequiredFieldValidator` fires, and the server returns HTTP 400 with `code: 400` and a validation-error body.

## Suspected Defects

### Defects that affect the use case

**Unknown agency ID returns HTTP 200 with a null body instead of HTTP 404.**

When the supplied agency ID does not match any agency in the transit graph, `StopsBeanServiceImpl` throws a `NoSuchAgencyServiceException` ([`StopsBeanServiceImpl.java` L261–262](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L261-L262)). The `ExceptionInterceptor` does not match `NoSuchAgencyServiceException` in its explicit type checks — it handles `NoSuchStopServiceException`, `NoSuchTripServiceException`, and `NoSuchRouteServiceException`, but not the agency equivalent ([`ExceptionInterceptor.java` L69–73](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ExceptionInterceptor.java#L69-L73)). The exception propagates past the interceptor's explicit handling and the action's response object remains null, causing the framework to serialise the JSON literal `null` with HTTP 200. The intended behaviour, consistent with other resource-not-found cases, is HTTP 404 with a structured error response.

The Go implementation must decide whether to replicate the null-body 200 response or return a proper 404 for unrecognised agency IDs.

## Open Questions

*(none)*

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Plain agency identifier, supplied as the path segment after stop-ids-for-agency/"
    },
    "key": {
      "type": "string",
      "description": "API authentication key"
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "Whether to populate the references block. Has no observable effect for this endpoint because no entity beans are constructed and the block is always empty."
    }
  }
}
```

**`id`** — The plain identifier of the transit agency whose stop IDs are requested. This is the bare agency ID, not the `{agencyId}_{entityId}` combined form. The agencies available on this server can be discovered via `agencies-with-coverage`.

**`key`** — API authentication key. Required on every request. An absent or unrecognised key returns HTTP 401.

**`includeReferences`** — Controls whether the `references` block is populated. Because this endpoint returns string IDs rather than entity beans, no entities are ever added to the references collection; the block is always an object of empty arrays regardless of this flag.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "version":     { "type": "integer" },
    "code":        { "type": "integer" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "text":        { "type": "string" },
    "data":        { "type": "object" }
  }
}
```

**`version`** — API version negotiated for this response. Always `2` for this endpoint.

**`code`** — HTTP-equivalent status code echoed in the body. `200` on success.

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`text`** — Human-readable status message. `"OK"` on success.

**`data`** — The list-with-references payload; see below.

### data

```json
{
  "type": "object",
  "properties": {
    "list":           { "type": "array",   "items": { "type": "string" } },
    "limitExceeded":  { "type": "boolean" },
    "references":     { "type": "object" }
  }
}
```

**`data.list`** — Array of stop identifiers for all stops associated with the requested agency. Each element is a combined `{agencyId}_{entityId}` string (e.g. `"1_29223"`). The list order reflects internal transit-graph iteration order for the agency and is not guaranteed to be stable across data bundle rebuilds or sorted in any documented way.

**`data.limitExceeded`** — Always `false`. No result cap is applied to this endpoint; all stop IDs for the agency are returned.

**`data.references`** — Always present; always an object of empty arrays (`agencies`, `routes`, `situations`, `stopTimes`, `stops`, `trips`). No entity beans are constructed for a stop-IDs-only response, so nothing is ever added to the references block.
