# block / block-instance

## Goal in Context

Retrieve the service schedule for a vehicle block. The `block` endpoint returns all schedule variants (configurations) that apply to the block across different service calendars. The `block-instance` endpoint returns the single schedule variant active on a specific service date.

## Scope

OBA REST API — block data sub-resources.

## Level

Sub-function

## Primary Actor

Rider (accessing the system via a client app)

## Stakeholders and Interests

**Rider** — wants to understand which trips a vehicle will execute in sequence throughout a service day, including the stop times and distances along each trip, to plan connecting journeys or trace vehicle continuity across routes that interline within the same block.

## Preconditions

- The caller has a combined block ID (typically obtained from the `blockId` field in a `trip` response).
- For the block-instance endpoint, the caller additionally knows the service date for which they want the active schedule.

## Minimal Guarantees

The response envelope always contains `code`, `currentTime`, `text`, and `version` fields.

## Success Guarantees

- **Block endpoint:** the response entry contains the block's combined ID and the complete list of block configurations, each with its ordered sequence of trips and stop times.
- **Block-instance endpoint:** the response entry contains the combined block ID, the service date as a Unix millisecond timestamp, and the single block configuration active on that date.

## Trigger

A client app requests block data — for example, to display a vehicle's complete day itinerary, to resolve interlining continuity between routes, or to retrieve stop times for every stop in a block.

## Main Success Scenario

_Applies to `GET /api/where/block/{id}`._

1. The client sends a GET request to `/api/where/block/{id}` supplying the combined block ID in the path.

2. The server looks up the block entry by its parsed combined ID. ([`BlockBeanServiceImpl.getBlockForId` L99](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/BlockBeanServiceImpl.java#L99))

3. The server converts every block configuration into a response bean ([`BeanFactoryV2.getBlock` L698](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L698)). For each configuration, the server records:
   - The list of active service calendar IDs — calendar IDs that must all be active on a date for this configuration to apply.
   - The list of inactive service calendar IDs — calendar IDs that must be inactive on a date for this configuration to apply.
   - An ordered list of block trips (see step 4).

   Configurations are ordered from the one with the greatest number of service calendar IDs to the fewest; ties are broken alphabetically by service ID. ([`BlockEntry.getConfigurations()` contract](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/services/transit_graph/BlockEntry.java#L28))

4. For each trip in a configuration ([`BeanFactoryV2.getBlockTrip` L720](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L720)), the server:
   - Adds the trip to the references block.
   - Records the trip's combined ID, the accumulated layover time in seconds from all preceding layovers in the block before this trip starts, and the distance in metres from the block origin to the start of this trip.
   - Records an ordered list of block stop times (see step 5).

5. For each stop in a trip ([`BeanFactoryV2.getBlockStopTime` L739](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L739), [`getStopTime` L748](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L748)), the server:
   - Adds the stop to the references block.
   - Records: the zero-based sequence index across all stops in the entire block configuration, the distance in metres from the block origin, the accumulated layover time in seconds from all preceding layovers before this stop, and a stop-time sub-object containing the stop's combined ID, scheduled arrival and departure times in seconds since midnight of the service date, and GTFS pickup/drop-off type codes.

6. The server returns HTTP 200 with `code: 200`, the entry, and a references block containing all referenced trips, stops, routes, and agencies.

## Extensions

**2a. Block not found:**
The block ID does not exist in the transit graph. The server returns HTTP 404 with `code: 404` and `text: "resource not found"`. ([`BlockAction.show()` L64](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/BlockAction.java#L64))

**1b. `id` parameter absent:**
The framework detects the missing required `id` path parameter and the server returns HTTP 400 with `code: 400` and `text: "validation error"`. ([`BlockAction.setId` L45](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/BlockAction.java#L45))

**1c. Block-instance variant — `GET /api/where/block-instance/{id}`:**
1. The client sends a GET request to `/api/where/block-instance/{id}` supplying the combined block ID in the path and a `serviceDate` query parameter. The service date must be supplied as either a Unix millisecond timestamp or a datetime string in `yyyy-MM-dd_HH-mm-ss` format. The timestamp must correspond exactly to midnight in the agency's local timezone on the target service date. ([`DateTimeConverter` L32](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-presentation/src/main/java/org/onebusaway/presentation/impl/conversion/DateTimeConverter.java#L32))
2. The server iterates through the block's configurations in their standard sort order (most service calendar IDs first). The first configuration for which all active service calendar IDs are active on the given date is selected. ([`BlockCalendarServiceImpl.getBlockInstance` L89](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/blocks/BlockCalendarServiceImpl.java#L89), [L111](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/blocks/BlockCalendarServiceImpl.java#L111))
3. The server constructs the response entry with the combined block ID, the service date as a Unix millisecond timestamp, and the single matching configuration object (structure identical to a configuration in the block response). ([`BeanFactoryV2.getBlockInstance` L690](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L690))
4. The server returns HTTP 200 with `code: 200`, the entry, and references.

**1c–2a. No configuration active on the given date:**
None of the block's configurations has all of its service calendar IDs active on the supplied service date. The server returns HTTP 404 with `code: 404`. ([`BlockInstanceAction.show()` L75](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/BlockInstanceAction.java#L75))

**1c–2b. Block not found (block-instance):**
Identical to extension 2a — returns HTTP 404.

## Suspected Defects

### Defects that affect the use case

**1. `BlockConfigurationV2Bean` omits the agency timezone.**
`BlockBeanServiceImpl.getBlockConfigurationAsBean` (lines [L236–L239](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/BlockBeanServiceImpl.java#L236)) populates `BlockConfigurationBean.timeZone` with the IANA timezone ID of the owning agency (e.g., `"America/Los_Angeles"`). `BeanFactoryV2.getBlockConfig` (lines [L708–L718](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L708)) maps this to `BlockConfigurationV2Bean`, which has no `timeZone` field, so the timezone is silently dropped from the API response. Stop times in the block response are expressed as seconds since midnight of the service date. Without the agency timezone a client cannot determine the absolute instant that midnight represents, making it impossible to convert block stop times to absolute timestamps. The intended response probably includes a `timeZone` (IANA string) in the configuration object.

**2. `serviceDate` is not validated as a required parameter on the block-instance endpoint.**
`BlockInstanceAction` applies `@RequiredFieldValidator` only to the `id` parameter ([L45](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/BlockInstanceAction.java#L45)). The `serviceDate` setter at [L59–L62](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/BlockInstanceAction.java#L59) carries no such annotation. When `serviceDate` is absent, the framework does not invoke the setter and the field defaults to `0` (the Unix epoch). The query then asks for the block configuration active on 1 January 1970, which never matches, and the server returns HTTP 404 instead of HTTP 400. A service date is semantically required for this endpoint; the intended behaviour was probably to return 400 when it is absent.

### Implementation defects only

**3. `BlockInstanceV2Bean.frequency` is never populated.**
`BlockInstanceV2Bean` declares a `frequency` field ([`BlockInstanceV2Bean.java` L22](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/blocks/BlockInstanceV2Bean.java#L22)), but `BeanFactoryV2.getBlockInstance` ([L690–L696](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L690)) never sets it, and the underlying `BlockInstanceBean` has no frequency field. The `frequency` property is always `null` in the response. This is an incomplete mapping for headway-based services; a clean reimplementation would either omit this field or populate it from the underlying block instance data when present.

## Open Questions

None. All observable behaviours were resolved through static analysis of the full call chain and confirmed against the live server.

---

## Request Parameters

### `GET /api/where/block/{id}`

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Path parameter. Combined block ID in {agencyId}_{entityId} format."
    },
    "key": {
      "type": "string",
      "description": "API authentication key."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "Whether to populate the references block in the response."
    }
  }
}
```

**`id`** — Combined block ID. The portion before the first `_` is the agency ID; everything after is the entity ID. Typically obtained from the `blockId` field of a trip response.

**`key`** — API authentication key.

**`includeReferences`** — When `false`, the `references` object in the response is present but all lists are empty. Defaults to `true`.

---

### `GET /api/where/block-instance/{id}`

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Path parameter. Combined block ID in {agencyId}_{entityId} format."
    },
    "serviceDate": {
      "type": "string",
      "description": "The service date for which the active block configuration is requested. Accepted formats: a Unix millisecond timestamp (digits only) or a datetime string yyyy-MM-dd_HH-mm-ss. Must represent exactly midnight in the agency's local timezone. If omitted the server uses the Unix epoch (1970-01-01), which will not match any block, resulting in a 404 — see Suspected Defects."
    },
    "key": {
      "type": "string",
      "description": "API authentication key."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "Whether to populate the references block in the response."
    }
  }
}
```

**`id`** — Combined block ID.

**`serviceDate`** — The service date to query. Must correspond exactly to midnight in the agency's local timezone on the calendar date of interest. Accepts either a decimal Unix millisecond timestamp or a string in `yyyy-MM-dd_HH-mm-ss` format (the time component is typically `00-00-00`). See Suspected Defects for the missing-parameter behaviour.

**`key`** — API authentication key.

**`includeReferences`** — When `false`, the references lists are present but empty.

---

## Response Structure

### Envelope (both endpoints)

```json
{
  "type": "object",
  "properties": {
    "code":        { "type": "integer" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "text":        { "type": "string" },
    "version":     { "type": "integer" },
    "data":        { "type": "object" }
  }
}
```

**`code`** — HTTP-equivalent status code mirrored inside the body (200, 400, 404, 500).

**`currentTime`** — Server wall-clock time at the moment of the response, in Unix milliseconds.

**`text`** — Human-readable status message (e.g., `"OK"`, `"resource not found"`).

**`version`** — API version negotiated for this request (always `2`).

**`data`** — Response payload; structure depends on the endpoint (see below).

---

### `data` — block endpoint

```json
{
  "type": "object",
  "properties": {
    "entry": { "$ref": "#/definitions/BlockEntry" },
    "references": { "$ref": "#/definitions/References" }
  }
}
```

---

### `data.entry` — block endpoint

```json
{
  "type": "object",
  "properties": {
    "id":             { "type": "string" },
    "configurations": { "type": "array", "items": { "$ref": "#/definitions/BlockConfiguration" } }
  }
}
```

**`data.entry.id`** — Combined block ID in `{agencyId}_{entityId}` format.

**`data.entry.configurations`** — All block configurations, ordered from the one with the greatest number of active service calendar IDs to the fewest. Within the same count, configurations are ordered by service ID alphabetically.

---

### `data.entry` — block-instance endpoint

```json
{
  "type": "object",
  "properties": {
    "blockId":            { "type": "string" },
    "serviceDate":        { "type": "integer", "description": "Unix ms" },
    "frequency":          { "type": "object", "description": "Always null — see Suspected Defects." },
    "blockConfiguration": { "$ref": "#/definitions/BlockConfiguration" }
  }
}
```

**`data.entry.blockId`** — Combined block ID.

**`data.entry.serviceDate`** — The service date for which this block instance is active, as a Unix millisecond timestamp representing midnight in the agency's local timezone.

**`data.entry.frequency`** — Always `null` in the current implementation. See Suspected Defects.

**`data.entry.blockConfiguration`** — The single block configuration active on the requested service date. Same structure as an element in `configurations[]` on the block endpoint.

---

### `data.entry.configurations[]` / `data.entry.blockConfiguration` — BlockConfiguration

```json
{
  "type": "object",
  "properties": {
    "activeServiceIds":   { "type": "array", "items": { "type": "string" } },
    "inactiveServiceIds": { "type": "array", "items": { "type": "string" } },
    "trips":              { "type": "array", "items": { "$ref": "#/definitions/BlockTrip" } }
  }
}
```

**`activeServiceIds[]`** — Combined service calendar IDs that must all be active on a given date for this configuration to apply. Each element is in `{agencyId}_{entityId}` format.

**`inactiveServiceIds[]`** — Combined service calendar IDs that must all be inactive on a given date for this configuration to apply. Typically empty; populated when a configuration only applies on days where a "holiday" or "exception" calendar is not running.

**`trips[]`** — The ordered sequence of trips the vehicle operates under this configuration, from the first trip of the day to the last. See BlockTrip below.

---

### `data.entry.configurations[].trips[]` — BlockTrip

```json
{
  "type": "object",
  "properties": {
    "tripId":               { "type": "string" },
    "accumulatedSlackTime": { "type": "integer" },
    "distanceAlongBlock":   { "type": "number" },
    "blockStopTimes":       { "type": "array", "items": { "$ref": "#/definitions/BlockStopTime" } }
  }
}
```

**`trips[].tripId`** — Combined trip ID referencing the `trips[]` list in the response references block.

**`trips[].accumulatedSlackTime`** — Total layover time in seconds accumulated from all inter-trip gaps in the block before the start of this trip. Zero for the first trip. For example, if the block has a 24-minute layover between trips 1 and 2, `accumulatedSlackTime` on trip 3 includes that 24 minutes plus any earlier layovers.

**`trips[].distanceAlongBlock`** — Distance in metres from the origin of the block (the first stop of the first trip) to the first stop of this trip.

**`trips[].blockStopTimes[]`** — Ordered list of scheduled stops for this trip within the block. See BlockStopTime below.

---

### `data.entry.configurations[].trips[].blockStopTimes[]` — BlockStopTime

```json
{
  "type": "object",
  "properties": {
    "blockSequence":        { "type": "integer" },
    "distanceAlongBlock":   { "type": "number" },
    "accumulatedSlackTime": { "type": "number" },
    "stopTime":             { "$ref": "#/definitions/StopTime" }
  }
}
```

**`blockStopTimes[].blockSequence`** — Zero-based index of this stop across all stops in the entire block configuration, counting across all trips in order. The first stop of the first trip is `0`.

**`blockStopTimes[].distanceAlongBlock`** — Distance in metres from the block origin (first stop of the first trip) to this stop.

**`blockStopTimes[].accumulatedSlackTime`** — Layover time in seconds accumulated from all preceding inter-trip gaps in the block before this stop is reached. Stops within the first trip before any layover have `0.0`.

**`blockStopTimes[].stopTime`** — Scheduled stop details. See StopTime below.

---

### `data.entry.configurations[].trips[].blockStopTimes[].stopTime` — StopTime

```json
{
  "type": "object",
  "properties": {
    "stopId":        { "type": "string" },
    "arrivalTime":   { "type": "integer" },
    "departureTime": { "type": "integer" },
    "pickupType":    { "type": "integer" },
    "dropOffType":   { "type": "integer" }
  }
}
```

**`stopTime.stopId`** — Combined stop ID referencing the `stops[]` list in the response references block.

**`stopTime.arrivalTime`** — Scheduled arrival time in seconds elapsed since midnight of the service date. Values greater than 86 400 indicate times after midnight that belong to the preceding service date (e.g., a trip that runs until 01:30 the next calendar day has arrival times up to approximately 91 800).

**`stopTime.departureTime`** — Scheduled departure time in seconds elapsed since midnight of the service date. Same scale as `arrivalTime`.

**`stopTime.pickupType`** — GTFS pickup type: `0` = regularly scheduled, `1` = no pickup, `2` = must phone agency, `3` = must coordinate with driver.

**`stopTime.dropOffType`** — GTFS drop-off type: same values as `pickupType`.

---

### `data.references` — References (both endpoints)

```json
{
  "type": "object",
  "properties": {
    "agencies":   { "type": "array" },
    "routes":     { "type": "array" },
    "trips":      { "type": "array" },
    "stops":      { "type": "array" },
    "situations": { "type": "array" }
  }
}
```

**`references.trips[]`** — Full trip objects for every trip referenced in the block. Each trip record includes the trip's combined ID, route ID, service ID, direction, headsign, shape ID, and block ID.

**`references.stops[]`** — Full stop objects for every stop referenced in the block stop times. Each stop record includes the stop's combined ID, name, coordinates, and code.

**`references.routes[]`** — Full route objects for every route referenced through the trips in the block.

**`references.agencies[]`** — Full agency objects for every agency referenced through the routes.

**`references.situations[]`** — Service alerts relevant to this block (typically empty for block queries).
