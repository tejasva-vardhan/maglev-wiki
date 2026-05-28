# Use Case: Get Trip

## Goal in Context

A rider's client application needs the static schedule attributes of a specific trip — route, direction, headsign, shape, block membership, and service calendar — to display trip details or to cross-reference trip IDs obtained from other endpoints.

## Scope

OneBusAway REST API

## Level

User goal

## Primary Actor

Rider (via a client application)

## Stakeholders and Interests

**Rider** — wants accurate trip details so that a client app can identify the route, heading, and scheduled path of a specific vehicle journey.

## Preconditions

- The caller knows the combined `agencyId_entityId` form of the trip ID (e.g. obtained from an arrivals-and-departures query or a trips-for-route query).
- The trip ID refers to a trip present in the currently loaded GTFS bundle.

## Minimal Guarantees

- The response envelope always includes a `version`, `code`, `currentTime`, and `text` field.
- A 404 response is returned when the trip ID is syntactically valid (contains an underscore) but no matching trip exists.

## Success Guarantees

- The response contains the complete static schedule attributes of the requested trip.
- The references block contains the route the trip belongs to and the agency that operates that route (unless `includeReferences=false`).

## Trigger

A client issues `GET /api/where/trip/{id}.json?key={apiKey}` with a valid trip ID.

## Main Success Scenario

1. The client supplies the combined trip ID as the path segment `{id}`.
2. The system validates that `{id}` is non-empty. ([`TripAction.java#L59–60`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripAction.java#L59))
3. The system parses the ID by splitting on the first underscore character to separate agency from entity ID. ([`AgencyAndIdLibrary.java#L46–52`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-util/src/main/java/org/onebusaway/util/AgencyAndIdLibrary.java#L46))
4. The system looks up the trip in the in-memory transit graph and narrative store. ([`TripBeanServiceImpl.java#L63–102`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripBeanServiceImpl.java#L63))
5. The system constructs the trip entry: all string fields (`routeShortName`, `tripShortName`, `tripHeadsign`) are set to their stored values and may be empty strings. The shape ID and direction ID are included only if present in the source data. ([`BeanFactoryV2.java#L506–526`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L506))
6. The system adds the trip's route and the route's owning agency to the references block. ([`BeanFactoryV2.java#L512–513`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L512))
7. The system returns HTTP 200 with the trip entry and references.

## Extensions

**2a. `id` is missing or empty:**
- The system returns HTTP 400 with a validation error envelope. ([`ApiActionSupport.java#L135–140`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ApiActionSupport.java#L135))

**3a. `id` contains no underscore (malformed combined ID):**
- The ID parser throws an internal exception that bypasses the action's error handling. The framework serialises a null result and returns HTTP 200 with the body `null`. See Suspected Defects. ([`AgencyAndIdLibrary.java#L47–48`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-util/src/main/java/org/onebusaway/util/AgencyAndIdLibrary.java#L47))

**4a. No trip found for the given ID:**
- The system returns HTTP 404 with a resource-not-found envelope. ([`TripAction.java#L64–65`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/TripAction.java#L64))

**4b. Trip entry exists in the graph but has no narrative record:**
- `TripBeanServiceImpl.getTripForId` returns null when the narrative is absent, and the action returns HTTP 404 as in extension 4a. ([`TripBeanServiceImpl.java#L79`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/TripBeanServiceImpl.java#L79))

**6a. `includeReferences=false`:**
- The references block is returned but all its sub-arrays are empty. The route and agency are not populated. ([`BeanFactoryV2.java#L1621–1635`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1621))

## Suspected Defects

### Defects that affect the use case

**Malformed ID (no underscore) produces HTTP 200 with null body** — `TripAction` / `TransitDataServiceTemplateImpl`

When the path segment `{id}` contains no underscore character, [`AgencyAndIdLibrary.convertFromString`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-util/src/main/java/org/onebusaway/util/AgencyAndIdLibrary.java#L47) throws an `IllegalStateException`. This propagates through `TransitDataServiceTemplateImpl.getTrip` and out of `TripAction.show` without being caught, causing the Struts2 framework to serialise a null model as the response body with HTTP 200. The intended behaviour is a 400 validation error or a 404 resource-not-found response.

### Implementation defects only

**`timeZone` field is always an empty string** — `BeanFactoryV2`

[`BeanFactoryV2.getTrip`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L506) never calls `bean.setTimeZone()`, even though `TripV2Bean` declares a `timeZone` field and the internal `TripBean` model has no timezone source. The field is serialised as an empty string in every response.

## Request Parameters

```json
{
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Combined agencyId_entityId trip identifier, embedded in the URL path"
    },
    "key": {
      "type": "string",
      "description": "API authentication key"
    },
    "includeReferences": {
      "type": "boolean",
      "default": true
    },
    "version": {
      "type": "integer",
      "default": 2
    }
  },
  "required": ["id", "key"]
}
```

**`id`** — The combined `agencyId_entityId` identifier for the trip, supplied as the final path segment of the URL (e.g. `/api/where/trip/1_664500360.json`). The agency portion precedes the first underscore; the entity portion follows it and may itself contain underscores.

**`key`** — API authentication key, required for all requests.

**`includeReferences`** — When `true` (the default), the response's references block is populated with the route and agency objects referenced by the entry. When `false`, the references block is returned but all sub-arrays are empty.

**`version`** — API version selector. Only version `2` is supported; any other value returns a 500 error.

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "version":     { "type": "integer" },
    "code":        { "type": "integer" },
    "text":        { "type": "string" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
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

**`version`** — Echo of the API version used (always `2`).

**`code`** — HTTP-equivalent status code (`200` on success, `400` for validation error, `404` when trip not found, `500` for unknown version).

**`text`** — Human-readable description of the result (`"OK"`, `"resource not found"`, etc.).

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`data.entry`** — The trip object (present only on HTTP 200 success responses).

**`data.references`** — The references block (always present on HTTP 200 responses).

---

### data.entry

```json
{
  "type": "object",
  "properties": {
    "id":             { "type": "string" },
    "routeId":        { "type": "string" },
    "routeShortName": { "type": "string" },
    "tripShortName":  { "type": "string" },
    "tripHeadsign":   { "type": "string" },
    "serviceId":      { "type": "string" },
    "shapeId":        { "type": "string" },
    "directionId":    { "type": "string" },
    "blockId":        { "type": "string" },
    "peakOffpeak":    { "type": "integer" }
  },
  "required": ["id", "routeId", "serviceId", "blockId"]
}
```

**`data.entry.id`** — The trip's combined `agencyId_entityId` identifier, exactly as supplied in the request.

**`data.entry.routeId`** — Combined `agencyId_entityId` identifier for the route this trip operates on.

**`data.entry.routeShortName`** — Short public name for the route specific to this trip (e.g. a variant name). May be an empty string if the trip does not carry its own route short name; clients should fall back to the route's `shortName` in the references block.

**`data.entry.tripShortName`** — Short public identifier for this individual trip (e.g. a train number). May be an empty string.

**`data.entry.tripHeadsign`** — Destination text displayed on the vehicle for this trip (e.g. `"SE Auburn"`). May be an empty string.

**`data.entry.serviceId`** — Combined `agencyId_entityId` string that identifies the service calendar governing on which dates this trip operates. This is an opaque key; its calendar dates are not returned in this endpoint.

**`data.entry.shapeId`** — Combined `agencyId_entityId` identifier for the geographic path the vehicle follows. Absent (or empty string) when the trip has no associated shape in the GTFS feed. Clients can retrieve the shape coordinates via the `shape/{id}` endpoint.

**`data.entry.directionId`** — Inbound/outbound designation for this trip within its route: `"0"` or `"1"`. May be absent when the GTFS feed does not specify a direction.

**`data.entry.blockId`** — Combined `agencyId_entityId` identifier for the block this trip belongs to. A block is the complete ordered sequence of trips a single vehicle executes on one operating day; use the `block/{id}` endpoint to retrieve the full block structure.

**`data.entry.peakOffpeak`** — Integer flag indicating whether this trip operates during peak hours. `0` indicates the trip is not flagged as a peak service; non-zero values indicate peak designation. This field is primarily used by commuter rail operators and will be `0` for most bus agencies.

---

### data.references

```json
{
  "type": "object",
  "properties": {
    "agencies":   { "type": "array", "items": { "type": "object" } },
    "routes":     { "type": "array", "items": { "type": "object" } },
    "stops":      { "type": "array", "items": { "type": "object" } },
    "trips":      { "type": "array", "items": { "type": "object" } },
    "situations": { "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.references.agencies`** — Array containing the agency that operates the trip's route. Each entry is a full agency object (id, name, timezone, URL, phone, etc.). Empty when `includeReferences=false`.

**`data.references.routes`** — Array containing the route the trip belongs to (id, shortName, longName, type, color, agencyId, etc.). Empty when `includeReferences=false`.

**`data.references.stops`** — Always empty for this endpoint.

**`data.references.trips`** — Always empty for this endpoint.

**`data.references.situations`** — Always empty for this endpoint.
