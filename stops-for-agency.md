# stops-for-agency

## Goal in Context

A client application needs the complete list of stops served by a specific transit agency — either as full stop records (coordinates, names, accessibility, serving routes) for display or analysis, or as a compact list of stop IDs for index building or bulk prefetching. This is a static schedule data query: the stop catalogue changes only when a new data bundle is loaded, not with real-time conditions.

## Scope

OBA REST API — the `stops-for-agency` and `stop-ids-for-agency` endpoints.

## Level

User goal.

## Primary Actor

Rider (via a client application).

## Stakeholders and Interests

**Rider** — wants a reliable, complete enumeration of an agency's stops, with accurate geographic positions, accessibility status, and the routes that serve each stop.

## Preconditions

- A transit data bundle is loaded and the API is online.
- The caller supplies the ID of an agency known to the system.
- A valid API key is present in the request.

## Minimal Guarantees

- The response body is syntactically valid JSON.
- If the agency ID is recognised, HTTP 200 is returned with a well-formed data envelope.

## Success Guarantees

- **stops-for-agency**: the response list contains exactly the set of stops associated with the named agency, sorted alphabetically by combined stop ID, each with full geographic and route detail. All routes referenced by any stop are present in the references block.
- **stop-ids-for-agency**: the response list contains exactly the combined stop IDs of all stops for the named agency. The references block is empty.
- `limitExceeded` is `false` in both responses; no cap is ever applied to the result set.

## Trigger

The rider's client calls `GET /api/where/stops-for-agency/{id}.json` or `GET /api/where/stop-ids-for-agency/{id}.json` with a known agency ID.

## Main Success Scenario

### stops-for-agency

1. The caller issues `GET /api/where/stops-for-agency/{agencyId}.json?key=<key>`.
2. The server looks up the agency by its plain ID in the transit graph ([`StopsBeanServiceImpl.java:246`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L246)).
3. For each stop belonging to the agency, the server fetches the full stop record: geographic coordinates, human-readable name, passenger-facing stop code, location type, wheelchair boarding accessibility, parent station (if any), and all route collections serving the stop across any service date ([`StopsBeanServiceImpl.java:249–254`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L249-L254)).
4. The collected stops are sorted alphabetically by combined stop ID before the response is assembled ([`StopsBeanServiceImpl.java:274`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L274)).
5. Full route records for every route appearing in any stop's `routeIds` or `staticRouteIds` are included in the references block. If any stop has a parent station, a minimal record for that parent is included in the references block under `stops`.
6. The server returns HTTP 200 with the assembled list and `limitExceeded: false` ([`StopsForAgencyAction.java:59–61`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForAgencyAction.java#L59-L61)).

### stop-ids-for-agency

1. The caller issues `GET /api/where/stop-ids-for-agency/{agencyId}.json?key=<key>`.
2. The server looks up the agency in the transit graph ([`StopsBeanServiceImpl.java:261`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L261)).
3. The combined ID of each stop (`agencyId_entityId`) is collected from the transit graph and placed directly into the response list. No sorting is applied ([`StopsBeanServiceImpl.java:263–267`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L263-L267)).
4. The references block is empty; no route or stop records are resolved.
5. The server returns HTTP 200 with the assembled list and `limitExceeded: false` ([`StopIdsForAgencyAction.java:59–61`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopIdsForAgencyAction.java#L59-L61)).

## Extensions

**1a. The caller supplies `includeReferences=false`:**
The references block is present in the envelope but contains no entries. For `stop-ids-for-agency`, this parameter is accepted but has no effect since the references block is always empty for that endpoint.

**3a. (stops-for-agency) A stop has a parent station:**
The stop's `parent` field contains the combined ID of the enclosing station stop. A minimal record for the parent station is added to `data.references.stops`. *Maglev intentionally expands this to a full record — see [Implementation Decisions](#implementation-decisions).*

**3b. (stops-for-agency) A stop has an explicit static route annotation:**
`staticRouteIds` contains the IDs from that annotation, which may differ from `routeIds`. When no annotation exists (the common case), `staticRouteIds` falls back to the same set as `routeIds` ([`StopBeanServiceImpl.java:137–148`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopBeanServiceImpl.java#L137-L148)).

## Suspected Defects

### Defects that affect the use case

**Unknown agency ID returns HTTP 200 with a `null` body instead of HTTP 404.**

`StopsForAgencyAction` and `StopIdsForAgencyAction` — [`StopsForAgencyAction.java:59`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopsForAgencyAction.java#L59), [`StopIdsForAgencyAction.java:59`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/StopIdsForAgencyAction.java#L59).

When the supplied agency ID is not recognised, the service layer throws an exception indicating the agency does not exist. The global exception interceptor handles the analogous "no such stop", "no such trip", and "no such route" exceptions by returning HTTP 404 ([`ExceptionInterceptor.java:69–74`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ExceptionInterceptor.java#L69-L74)), but does not handle the agency-not-found case. The framework therefore serialises the unset action model as JSON, producing an HTTP 200 response with a body of `null`. Verified at runtime: `GET /api/where/stops-for-agency/UNKNOWN.json` returns HTTP 200, body `null`. The intended behaviour is an HTTP 404 response with a standard error envelope, consistent with the other entity-not-found cases. *Maglev intentionally corrects this — see [Implementation Decisions](#implementation-decisions).*

**`stop-ids-for-agency` returns IDs in undefined order.**

`StopsBeanServiceImpl.getStopsIdsForAgencyId` — [`StopsBeanServiceImpl.java:258–269`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L258-L269).

The `stops-for-agency` endpoint sorts its results alphabetically by combined stop ID before returning ([`StopsBeanServiceImpl.java:274`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/StopsBeanServiceImpl.java#L274)). `stop-ids-for-agency` iterates the same underlying stop entries without any sorting, so the response order reflects the transit graph's internal iteration sequence and may vary across server restarts. Verified at runtime: the IDs returned are not in alphabetical order. Clients cannot rely on a stable or alphabetical ordering. The likely intended behaviour is the same alphabetical sort by combined ID that `stops-for-agency` applies.

## Implementation Decisions

Decisions made for the Maglev implementation that intentionally deviate from the legacy OBA behaviour.

**Unknown agency ID returns HTTP 404 (deviates from legacy).**

The legacy implementation returns HTTP 200 with a null body when the supplied agency ID is not recognised (see [Suspected Defects](#suspected-defects)). Maglev intentionally corrects this: an unknown agency ID returns HTTP 404 with a standard error envelope, consistent with the treatment of other unknown entity IDs across the API.

**Parent station records in `references.stops` are full stop records (deviates from legacy).**

The legacy implementation adds a minimal record for each parent station to `data.references.stops` (see Extension 3a). Maglev expands this to a full stop record with the same structure as entries in `data.list`, including `routeIds` and `staticRouteIds` fetched from the database. This is consistent with how other endpoint specs define stop records in `references.stops`: the `stop` spec states the parent record "follows the same structure as `data.entry`, with all fields populated from the database", and the `arrivals-and-departures-for-stop` spec specifies "full stop objects" throughout.

## Open Questions

None.

---

## Request Parameters

### stops-for-agency

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string"
    },
    "key": {
      "type": "string"
    },
    "includeReferences": {
      "type": "boolean",
      "default": true
    }
  }
}
```

**`id`** — Plain agency ID, supplied as the final path segment before the format extension: `/api/where/stops-for-agency/{id}.json`. This is not a combined `agencyId_entityId` value — it is the agency's own identifier as it appears in the feed (e.g. `"1"`, `"40"`).

**`key`** — API authentication key.

**`includeReferences`** — When `false`, the references block in the response envelope is returned empty. Defaults to `true`.

### stop-ids-for-agency

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string"
    },
    "key": {
      "type": "string"
    },
    "includeReferences": {
      "type": "boolean",
      "default": true
    }
  }
}
```

**`id`** — Plain agency ID, supplied as the final path segment before the format extension: `/api/where/stop-ids-for-agency/{id}.json`.

**`key`** — API authentication key.

**`includeReferences`** — Accepted but has no effect; the references block is always empty for this endpoint.

---

## Response Structure

### stops-for-agency

#### Envelope

```json
{
  "type": "object",
  "properties": {
    "code": { "type": "integer" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "text": { "type": "string" },
    "version": { "type": "integer" },
    "data": {
      "type": "object",
      "properties": {
        "limitExceeded": { "type": "boolean" },
        "list": { "type": "array", "items": { "type": "object" } },
        "references": { "type": "object" }
      }
    }
  }
}
```

**`code`** — HTTP-style status code; `200` on success.
**`currentTime`** — Server wall-clock time at response generation, in Unix milliseconds.
**`text`** — Human-readable status string; `"OK"` on success.
**`version`** — API version; always `2`.
**`data.limitExceeded`** — Always `false`; no result cap is imposed.
**`data.list`** — Array of stop objects sorted alphabetically by combined stop ID.
**`data.references`** — Routes and parent station stops referenced by entries in `data.list`.

#### data.list[]

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
    "routeIds": { "type": "array", "items": { "type": "string" } },
    "staticRouteIds": { "type": "array", "items": { "type": "string" } },
    "parent": { "type": "string" }
  }
}
```

**`data.list[].id`** — Combined stop ID in `agencyId_entityId` form (e.g. `"1_100"`).
**`data.list[].lat`** — Stop latitude in decimal degrees (WGS 84).
**`data.list[].lon`** — Stop longitude in decimal degrees (WGS 84).
**`data.list[].name`** — Human-readable stop name (e.g. `"1st Ave & Spring St"`).
**`data.list[].code`** — Passenger-facing stop code (e.g. `"100"`). Falls back to the entity portion of the combined stop ID when the feed does not define a code.
**`data.list[].direction`** — Cardinal or intercardinal compass direction indicating the direction of travel at this stop (e.g. `"N"`, `"SW"`). Taken from the feed when defined; otherwise inferred from shape geometry, or empty string if neither is available. See [[Stop-Direction-Calculation]] for the inference algorithm.
**`data.list[].locationType`** — GTFS location type: `0` = boarding stop or platform (the default), `1` = station, `2` = entrance or exit.
**`data.list[].wheelchairBoarding`** — One of `"ACCESSIBLE"`, `"NOT_ACCESSIBLE"`, or `"UNKNOWN"`, derived from the GTFS `wheelchair_boarding` field.
**`data.list[].routeIds`** — Combined IDs of all route collections serving this stop across any service date, in `agencyId_entityId` form.
**`data.list[].staticRouteIds`** — Combined IDs of routes from a static route annotation attached to this stop. When no annotation exists, identical to `routeIds`.
**`data.list[].parent`** — Combined ID of the enclosing parent station stop when this stop is part of a station complex. Empty string when there is no parent station.

#### data.references

```json
{
  "type": "object",
  "properties": {
    "agencies": { "type": "array", "items": { "type": "object" } },
    "routes": { "type": "array", "items": { "type": "object" } },
    "stops": { "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.references.agencies`** — Agency records for each agency whose routes appear in the stop list.
**`data.references.routes`** — Full route records for every route ID appearing in any stop's `routeIds` or `staticRouteIds`.
**`data.references.stops`** — Minimal stop records for parent stations referenced by stops in `data.list`. Non-empty only when at least one stop has a parent station.

---

### stop-ids-for-agency

#### Envelope

```json
{
  "type": "object",
  "properties": {
    "code": { "type": "integer" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "text": { "type": "string" },
    "version": { "type": "integer" },
    "data": {
      "type": "object",
      "properties": {
        "limitExceeded": { "type": "boolean" },
        "list": { "type": "array", "items": { "type": "string" } },
        "references": { "type": "object" }
      }
    }
  }
}
```

**`code`** — HTTP-style status code; `200` on success.
**`currentTime`** — Server wall-clock time in Unix milliseconds.
**`text`** — `"OK"` on success.
**`version`** — API version; always `2`.
**`data.limitExceeded`** — Always `false`.
**`data.list`** — Array of combined stop IDs in `agencyId_entityId` form (e.g. `"1_100"`). The order reflects the transit graph's internal iteration sequence; no alphabetical sorting or stability guarantee is provided.
**`data.references`** — Always empty; no entities are resolved for this endpoint.
