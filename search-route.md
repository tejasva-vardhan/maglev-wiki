# search/route — Route Name Search

## Goal in Context

A rider (via a client app) types a partial route name or identifier into a search or autocomplete field. The system returns the matching routes so the rider can select one and proceed to view schedules or real-time arrivals.

## Scope

OneBusAway REST API — `GET /api/where/search/route.json`

## Level

User goal

## Primary Actor

Rider (accessing the system via a client app)

## Stakeholders and Interests

- **Rider** — wants to find a specific route quickly by typing part of its name or number, without needing to know the exact identifier in advance.

## Preconditions

- The server has completed its startup index build. The index is built asynchronously after Spring context initialisation; queries made before the index is ready will return no results rather than an error.
- The caller supplies a valid API key.

## Minimal Guarantees

- Any response — success or failure — is well-formed JSON.
- If `input` is omitted, the server returns HTTP 404 with a field-error body (see Suspected Defects).
- If the `input` matches no routes, the server returns HTTP 404 with the standard error envelope.

## Success Guarantees

- The response contains the list of routes whose indexed name or identifier begins with the supplied prefix.
- Each route entry includes enough information (short name, long name, route type, agency, branding) for a client to present a labelled autocomplete suggestion and to navigate to the selected route.
- The `limitExceeded` flag in the response accurately indicates whether additional matching routes exist beyond the returned set.
- The references block contains one agency entry for every agency that owns a route in the list.

## Trigger

The client sends `GET /api/where/search/route.json` with an `input` query parameter carrying the characters the rider has typed so far.

## Main Success Scenario

1. The client sends the request. `input` is required; `maxCount` (default 20) is optional.

2. The server lowercases `input` before lookup ([`ApiSearchAction.java#L40–41`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/search/ApiSearchAction.java#L40-L41)).

3. The server performs an exact-key lookup of the lowercased input against a pre-built route search index ([`BundleSearchServiceImpl.java#L226`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/bundle/BundleSearchServiceImpl.java#L226)).

   **Index construction** — the index is built once at startup and refreshed whenever the transit data bundle is reloaded ([`BundleSearchServiceImpl.java#L68–121`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/bundle/BundleSearchServiceImpl.java#L68-L121)). For each route across all agencies, a *search term* is derived:
   - If the route has a non-null GTFS `route_long_name`, the search term is that name.
   - If the route has no `route_long_name` (null, not empty string), the search term is the route's combined entity id (e.g. `1_100224`) ([`BundleSearchServiceImpl.java#L92`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/bundle/BundleSearchServiceImpl.java#L92)).

   The search term is split on whitespace, hyphens, forward slashes, parentheses, and ampersands ([`BundleSearchServiceImpl.java#L248`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/bundle/BundleSearchServiceImpl.java#L248)). Every prefix of each resulting word is stored as an index key (so "Link" generates keys `l`, `li`, `lin`, `link`). For multi-word terms, additional keys spanning the word boundary — from the start of the term up to 32 characters — are also stored, giving the effect of infix matching across words ([`BundleSearchServiceImpl.java#L169–201`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/bundle/BundleSearchServiceImpl.java#L169-L201)). All keys are lowercased. Within each index key, routes are stored in ascending combined-id order ([`BundleSearchServiceImpl.java#L327–335`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/bundle/BundleSearchServiceImpl.java#L327-L335)).

4. If the index contains more than `maxCount` routes for that key, the list is truncated to `maxCount` and `limitExceeded` is set to `true`; otherwise `limitExceeded` is `false` ([`BundleSearchServiceImpl.java#L225–235`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/bundle/BundleSearchServiceImpl.java#L225-L235)).

5. The server converts each `RouteBean` to a `RouteV2Bean` ([`BeanFactoryV2.java#L164–188`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L164-L188)) and applies a two-phase sort:

   - **Phase 1** — routes are sorted by short name using the deployment's configured comparator. With the default configuration (no explicit ordering map), this is a natural lexicographic comparison of short names (or combined entity id when short name is absent) ([`BeanFactoryV2.java#L176–180`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L176-L180)).
   - **Phase 2** — routes belonging to a deployment-configured primary agency are moved to the front of the list ([`BeanFactoryV2.java#L255–273`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L255-L273)).

   With the default deployment configuration (empty ordering map and no primary agency), phase 2 has no effect and routes remain in lexicographic order.

6. The server collects the owning agency of each route into the references block. Agencies in the references block are sorted by agency id, with the primary agency (if configured) sorted first.

7. The server returns HTTP 200 with the response envelope.

## Extensions

**1a. `input` is not supplied.**
At 1: Struts2's field validator runs before the action and returns HTTP 404 with a non-standard body `{"fieldErrors": {"input": ["missing input"]}}`. This bypasses the standard OBA response envelope entirely (see Suspected Defects).

**3a. No routes match the input.**
At 3: The server returns HTTP 404 with the standard error envelope (`code: 404`, `text: "resource not found"`) ([`RouteAction.java#L46–47`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/search/RouteAction.java#L46-L47)).

**3b. The route's GTFS `route_long_name` is an empty string (not null).**
At 3: The route is never added to the search index. A route with an explicitly empty long name is unsearchable by any prefix. This is a consequence of the null-check in the index builder: the empty-string fallback is not treated the same as the null fallback (see Suspected Defects).

---

## Suspected Defects

### Defects that affect the use case

**Missing `input` returns HTTP 404 with a non-standard error body.**
`RouteAction` inherits `@RequiredFieldValidator` from `ApiSearchAction` ([`ApiSearchAction.java#L38`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/search/ApiSearchAction.java#L38)). When validation fails, Struts2's REST plugin intercepts the request before the action method runs and returns HTTP 404 with `{"fieldErrors": {"input": ["missing input"]}}`. The response carries no `code`, `version`, `currentTime`, or `text` fields. The appropriate behaviour would be HTTP 400 wrapped in the standard OBA envelope (as produced by `setValidationErrorsResponse()`).

**Routes with an explicitly empty `route_long_name` are not indexed.**
In `BundleSearchServiceImpl.init()`, the fallback `if (hint == null) hint = route.getId()` only triggers when `getLongName()` returns Java null ([`BundleSearchServiceImpl.java#L92`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/bundle/BundleSearchServiceImpl.java#L92)). When a GTFS feed sets `route_long_name` to an empty string, the hint is `""`, `splitParts` generates no keys, and the route is silently dropped from the index. The intended behaviour is presumably to fall back to the combined entity id, matching the null case.

**`nullSafeShortName` is exposed as a response field.**
`RouteV2Bean.getNullSafeShortName()` ([`RouteV2Bean.java#L62–66`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/RouteV2Bean.java#L62-L66)) is a helper method that follows the `getX()` naming convention and is therefore serialised as the field `nullSafeShortName` in every list entry. The field is redundant: it equals `shortName` when `shortName` is non-null, and equals `id` when `shortName` is null. Clients should use `shortName` directly and implement the null fallback themselves if needed.

### Implementation defects only

**Dead field in `RouteAction`.**
`RouteAction` declares `private ArrivalsAndDeparturesQueryBean _query = new ArrivalsAndDeparturesQueryBean()` ([`RouteAction.java#L34`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/search/RouteAction.java#L34)) but never reads or writes it. This field has no effect on behaviour and is almost certainly a copy-paste artefact from another action class.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["input"],
  "properties": {
    "input": {
      "type": "string",
      "description": "Lowercased prefix to search for. Must be at least one character."
    },
    "maxCount": {
      "type": "integer",
      "default": 20,
      "description": "Maximum number of routes to return."
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
      "default": true
    }
  }
}
```

**`input`** — The search prefix. The server lowercases this value before looking it up in the index. The match is exact against the lowercased prefix: only routes whose indexed name or identifier begins with exactly this string are returned. Partial matches within a word are supported (e.g., `li` matches routes with long name starting with "li"); partial matches that span word boundaries (e.g., `link light`) are supported up to 32 characters.

**`maxCount`** — The maximum number of matching routes to return. If more routes are indexed under the supplied prefix than `maxCount` allows, the list is truncated and `data.limitExceeded` is set to `true`. Defaults to 20.

**`key`** — API key for authentication. Required.

**`version`** — Selects the API version. Only version 2 is implemented; any other value returns an error.

**`includeReferences`** — When `false`, the `data.references` block is still present in the response but all its arrays are empty. Defaults to `true`.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "version":     { "type": "integer" },
    "code":        { "type": "integer" },
    "text":        { "type": "string" },
    "currentTime": { "type": "number", "description": "Unix ms" },
    "data":        { "type": "object" }
  }
}
```

**`version`** — Always `2` for this endpoint.

**`code`** — `200` on success; `404` when no routes match or when `input` is missing.

**`text`** — `"OK"` on success; `"resource not found"` when no routes match.

**`currentTime`** — Server wall-clock time when the response was generated, in Unix milliseconds.

**`data`** — The payload object described below.

### data

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

**`data.limitExceeded`** — `true` if the index contained more matching routes than `maxCount` and the list was truncated; `false` otherwise.

**`data.outOfRange`** — Always `false` for this endpoint (the field is included because the response uses a shared list type, but this endpoint performs no geographic bounding and the value is hardcoded).

**`data.list`** — Array of matching route objects, sorted and ordered as described in step 5 of the main scenario.

**`data.references`** — Standard OBA references block. `agencies` is populated with one entry per agency that owns a route in the list. `routes` is always empty (routes appear directly in `data.list`, not in the references block).

### data.list[]

```json
{
  "type": "object",
  "properties": {
    "id":               { "type": "string" },
    "agencyId":         { "type": "string" },
    "shortName":        { "type": "string" },
    "longName":         { "type": "string" },
    "description":      { "type": "string" },
    "type":             { "type": "integer" },
    "url":              { "type": "string" },
    "color":            { "type": "string" },
    "textColor":        { "type": "string" },
    "nullSafeShortName":{ "type": "string" }
  }
}
```

**`data.list[].id`** — Combined entity id of the route, in `{agencyId}_{routeId}` form (e.g. `1_100224`).

**`data.list[].agencyId`** — Plain agency id of the owning agency (e.g. `1`). This is the raw agency id, not a combined id.

**`data.list[].shortName`** — GTFS `route_short_name` (e.g. `"44"`). May be absent when not set in the feed.

**`data.list[].longName`** — GTFS `route_long_name`. Returns `""` when the field is null in the underlying data (the serialiser converts null to empty string); an empty string response is indistinguishable from a genuinely empty long name.

**`data.list[].description`** — GTFS `route_desc`. Typically a brief plain-English description of the route's corridor or endpoints.

**`data.list[].type`** — GTFS `route_type` integer (e.g. `3` for bus, `1` for metro/subway).

**`data.list[].url`** — Agency-supplied URL for the route's information page.

**`data.list[].color`** — GTFS `route_color`, as a six-character hex string without a leading `#` (e.g. `"FDB71A"`). May be empty when not set.

**`data.list[].textColor`** — GTFS `route_text_color`, same format as `color`. May be empty when not set.

**`data.list[].nullSafeShortName`** — Equals `shortName` when `shortName` is non-null; equals `id` otherwise. This field is an inadvertent side-effect of serialising a helper method and is considered a defect; Go reimplementations should omit it (see Suspected Defects).
