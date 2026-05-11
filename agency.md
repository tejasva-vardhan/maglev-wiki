# `GET /api/where/agency/{id}`

## Goal in Context

A rider's client app retrieves the full record for one transit agency by its ID — name, contact details, website, timezone, and operating policies — so it can display that information to the rider or use it to configure subsequent API calls.

## Scope

OneBusAway REST API.

## Level

User goal.

## Primary Actor

Rider.

## Stakeholders and Interests

- **Rider** — wants complete, accurate agency details: name, website URL, phone number, fare page URL, email address, timezone, and any disclaimer the agency requires to be displayed when its data is used in an application.

## Preconditions

- The server has loaded a valid GTFS feed containing at least one agency record.
- The caller supplies a valid API key.

## Minimal Guarantees

- The response is well-formed JSON with `code`, `text`, `version`, and `currentTime` fields.
- On failure the `data` field is absent from the response body.

## Success Guarantees

- The response body contains a `data.entry` object with all agency fields populated from the GTFS feed.
- A `data.references` block is present but contains only empty arrays, as no other entity types are cross-referenced by this endpoint.

## Trigger

The client sends `GET /api/where/agency/{id}.json?key={key}`.

## Main Success Scenario

1. The client sends `GET /api/where/agency/{id}.json?key={key}`, supplying the plain agency ID in the URL path.
2. The system looks up the agency record for the given ID ([`AgencyAction.show()` line 59](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/AgencyAction.java#L59)).
3. The system returns HTTP 200. `data.entry` contains the full agency record (see Response Structure). `data.references` is present but all its arrays are empty.

## Extensions

- **2a. No agency exists with the given ID** — ([`AgencyAction.show()` line 61](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/AgencyAction.java#L61))
  - 2a1. The system returns HTTP 404. `code` is `404`, `text` is `"resource not found"`. The `data` field is absent.

## Suspected Defects

### Implementation defects only

- **`AgencyBeanServiceImpl`** — [`lines 51 and 57`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/AgencyBeanServiceImpl.java#L51-L57): `bean.setEmail(agency.getEmail())` is called twice in sequence within `getAgencyForId`. The second call immediately overwrites the first with the same value. This is harmless but redundant and would not arise in a clean reimplementation.

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Path parameter — the agency identifier"
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
  }
}
```

**`id`** — The agency's plain identifier, taken directly from the `agency_id` field in the GTFS `agency.txt` file. Unlike route, stop, and trip IDs, agency IDs are not wrapped in the combined `{agencyId}_{entityId}` format — the plain ID is supplied directly in the URL path (e.g., `/api/where/agency/1.json`).

**`key`** — API authentication key. Required on all requests.

**`version`** — Selects the response format version. Defaults to 2. Only version 2 is in scope for the Go rewrite.

**`includeReferences`** — When `true` (the default), the response includes a `data.references` block. Since the agency endpoint does not cross-reference any other entity types, this block is always empty regardless of this parameter's value.

## Response Structure

### Envelope

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

**`code`** — HTTP-equivalent status code mirrored in the body: `200` on success, `404` when no agency is found.

**`currentTime`** — The server's current time at the moment of the response, in Unix milliseconds.

**`text`** — Human-readable status description: `"OK"` on success, `"resource not found"` when the agency does not exist.

**`version`** — The response format version. Always `2` when the default is in effect.

**`data`** — Present only on a 200 response; absent on 404.

### data

```json
{
  "type": "object",
  "properties": {
    "entry":      { "type": "object" },
    "references": { "type": "object" }
  }
}
```

**`data.entry`** — The agency record (see below).

**`data.references`** — Always present on success. Contains empty arrays for `agencies`, `routes`, `stops`, `trips`, `situations`, and `stopTimes`. The agency endpoint adds no cross-references, so this block is structurally always empty regardless of the `includeReferences` parameter.

### data.entry

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
    "disclaimer":     { "type": "string" },
    "email":          { "type": "string" },
    "fareUrl":        { "type": "string" },
    "privateService": { "type": "boolean" }
  }
}
```

**`data.entry.id`** — The plain agency identifier (e.g., `"1"`). Not in the combined `{agencyId}_{entityId}` format.

**`data.entry.name`** — The agency's full public name (e.g., `"Metro Transit"`).

**`data.entry.url`** — The agency's public website URL (e.g., `"https://kingcounty.gov/en/dept/metro"`).

**`data.entry.timezone`** — The IANA timezone identifier for the agency's operating area (e.g., `"America/Los_Angeles"`). All schedule times for this agency are expressed relative to this timezone.

**`data.entry.lang`** — The ISO 639-1 language code for the primary language used by the agency (e.g., `"en"`). May be an empty string if not provided in the GTFS feed.

**`data.entry.phone`** — The agency's public customer service phone number (e.g., `"206-553-3000"`). May be an empty string if not provided in the GTFS feed.

**`data.entry.disclaimer`** — Any legal disclaimer the agency requires to be shown when its data is used in an application. May be an empty string.

**`data.entry.email`** — The agency's public contact email address. May be an empty string if not provided in the GTFS feed.

**`data.entry.fareUrl`** — URL to the agency's fare information page (e.g., `"https://kingcounty.gov/en/dept/metro/fares-and-payment/prices"`). May be an empty string if not provided in the GTFS feed.

**`data.entry.privateService`** — `true` if the agency provides service not available to the general public (e.g., a private shuttle service). `false` for publicly accessible transit agencies.
