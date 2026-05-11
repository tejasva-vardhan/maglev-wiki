# Use Case: Retrieve Shape

## Goal in Context

A rider's client app has obtained a shape ID from a prior API response (e.g. the trip details for a service the rider is considering). The app wants to draw the exact geographic path of that service on a map, so the rider can see where the vehicle will travel.

## Scope

OBA REST API

## Level

User goal

## Primary Actor

Rider (via a client application)

## Stakeholders and Interests

- **Rider**: Wants to see the precise geographic route of a transit service overlaid on a map, to understand where the vehicle travels and to plan boarding and alighting.

## Preconditions

- The server has loaded GTFS data, including shape point sequences.
- The caller holds a valid API key.
- The caller holds a shape ID obtained from another API response (e.g. from a trip record).

## Minimal Guarantees

- The server returns a well-formed response envelope with a machine-readable status code.

## Success Guarantees

- The server returns HTTP 200 with the full geographic path of the requested shape, encoded as a Google Encoded Polyline, together with a count of the coordinate points it contains.

## Trigger

`GET /api/where/shape/{id}.json?key=<key>`

## Main Success Scenario

1. The rider's client app sends a GET request supplying the shape's combined ID as the `id` path segment. The ID takes the form `{agencyId}_{entityId}`, where everything before the first underscore identifies the agency and everything after identifies the shape within that agency.

2. The server locates the ordered sequence of latitude/longitude coordinates for the shape. ([`ShapeBeanServiceImpl.java` L63](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/ShapeBeanServiceImpl.java#L63))

3. The server encodes the entire coordinate sequence — without any simplification or point reduction — using the Google Encoded Polyline Algorithm. All points present in the GTFS feed are included. ([`PolylineEncoder.java` L68–99](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-geospatial/src/main/java/org/onebusaway/geospatial/services/PolylineEncoder.java#L68-L99))

4. The server returns HTTP 200. The response envelope's `data.entry` contains the encoded polyline string (`points`), the number of coordinate points (`length`), and a `levels` field that is always an empty string and carries no useful information.

5. The response also contains a `data.references` block listing related entities; for this endpoint it is always empty, as shape records do not reference any other entity type.

## Extensions

**2a. Shape not found.** The combined ID is syntactically valid but no shape with that ID exists in the loaded data (the agency prefix is unknown, or the shape ID is not present within that agency's data). The server returns HTTP 404. ([`ShapeAction.java` L58–59](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ShapeAction.java#L58-L59))

**1a. Invalid API key.** The server returns HTTP 401 before the action runs.

---

## Suspected Defects

### Defects that affect the use case

**Malformed ID returns HTTP 200 with a null body.**

`AgencyIdSupport` ([`AgencyIdSupport.java` L35–36](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-federations/src/main/java/org/onebusaway/federations/annotations/AgencyIdSupport.java#L35-L36)) throws an internal exception when the supplied `id` contains no underscore, because it cannot extract an agency prefix. This exception escapes the action's error-handling logic (`ShapeAction.java` [L56–63](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ShapeAction.java#L56-L63)) and propagates through the interceptor chain in a way that causes the framework to write an HTTP 200 response with a literal `null` body rather than a structured error envelope.

The intended behaviour is to return HTTP 400 (invalid argument) when the ID is syntactically malformed. The Go implementation should validate that the ID contains an underscore before attempting lookup, and return 400 if it does not.

### Implementation defects only

None identified.

---

## Open Questions

None.

---

## Request Parameters

```json
{
  "type": "object",
  "properties": {
    "id": {
      "type": "string"
    },
    "key": {
      "type": "string"
    },
    "version": {
      "type": "integer",
      "default": 2
    },
    "includeReferences": {
      "type": "boolean",
      "default": true
    }
  },
  "required": ["id", "key"]
}
```

**`id`** — The shape's combined identifier, of the form `{agencyId}_{entityId}`. The part before the first underscore is the agency ID; everything after is the shape's local identifier within that agency. Shape IDs are obtained from trip records returned by other endpoints (e.g. `trip-details`). An ID without an underscore triggers a defect (see Suspected Defects).

**`key`** — API access key, required on every request.

**`version`** — Requested API version. Defaults to 2. This endpoint does not alter its behaviour based on version; the value is reflected back in the response envelope.

**`includeReferences`** — Whether to populate the `data.references` block. Defaults to `true`. This endpoint never adds any entries to the references block regardless of this flag, so the parameter has no observable effect here.

---

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
    "data":    { "type": "object" }
  }
}
```

**`version`** — The API version negotiated for this response (reflects the `version` request parameter, or 2 by default).

**`code`** — HTTP-style status code mirrored inside the body (200 on success, 404 when not found).

**`text`** — Human-readable status phrase, e.g. `"OK"` or `"resource not found"`.

**`currentTime`** — The server's wall-clock time at the moment the response was generated, in Unix milliseconds.

**`data`** — Present on success; absent on error responses.

---

### data.entry

```json
{
  "type": "object",
  "properties": {
    "points": { "type": "string" },
    "length": { "type": "integer" },
    "levels": { "type": "string" }
  }
}
```

**`data.entry.points`** — The full geographic path encoded as a Google Encoded Polyline string. Decoding this string yields an ordered sequence of latitude/longitude pairs tracing the vehicle's path from start to end. The encoding uses the algorithm described at <https://developers.google.com/maps/documentation/utilities/polylinealgorithm>. All coordinate points from the GTFS feed are included without simplification.

**`data.entry.length`** — The number of coordinate points in the encoded path. Equals the number of lat/lon pairs produced by decoding `points`.

**`data.entry.levels`** — Always an empty string. This field corresponds to a zoom-level string from an older Google Maps API and carries no meaningful information for current clients; it should be ignored.

---

### data.references

```json
{
  "type": "object",
  "properties": {
    "agencies":  { "type": "array", "items": { "type": "object" } },
    "routes":    { "type": "array", "items": { "type": "object" } },
    "stops":     { "type": "array", "items": { "type": "object" } },
    "trips":     { "type": "array", "items": { "type": "object" } },
    "situations":{ "type": "array", "items": { "type": "object" } },
    "stopTimes": { "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.references`** — Always present but always empty. Shape records do not reference any other entity type, so all arrays here are empty on every response.
