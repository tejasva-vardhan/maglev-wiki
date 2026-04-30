** DO NOT IMPLEMENT - DRAFT **

# agency/{id}

**Retrieve a single transit agency record by its identifier.**

## Goal in Context

A rider's client application needs the name, website, timezone, and contact details for a specific transit agency — for example, to display an attribution footer, link to the agency's fare page, or interpret schedule times in the correct timezone.

## Scope

OBA REST API — single resource retrieval.

## Level

User goal.

## Primary Actor

Rider (via a client application).

## Stakeholders and Interests

- **Rider** — wants accurate, complete agency information: the agency's public name, website URL, timezone (needed to interpret schedule times), contact phone number, fare URL, and any legal disclaimer the agency requires to be displayed when its data is used.

## Preconditions

- The agency ID is known to the caller (e.g. obtained from the `agencies-with-coverage` endpoint).
- The server has a loaded transit data bundle that contains the requested agency.

## Minimal Guarantees

- All responses include the envelope fields `version`, `code`, `currentTime`, and `text`.
- The `code` field in the envelope matches the HTTP status code, giving callers a machine-readable status regardless of transport-level error handling.

## Success Guarantees

- The response contains the agency record in `data.entry`, populated with all fields present in the source GTFS feed.
- HTTP status is 200 and `code` is 200.

## Trigger

Client sends `GET /api/where/agency/{id}.json`.

## Main Success Scenario

1. Client sends `GET /api/where/agency/{id}.json?key={key}`, where `{id}` is the agency identifier encoded directly in the URL path.
2. The system looks up the agency in the loaded transit data bundle using the supplied identifier.
3. The system returns HTTP 200. The response envelope contains the agency record in `data.entry` and an empty `data.references` block.

## Extensions

**2a. Agency not found:**
- 2a1. No agency with the given identifier exists in the loaded bundle. The system returns HTTP 404 with `code` 404 and `text` "resource not found". The `data` field is absent from the response body.

**1a. Unsupported version number:**
- 1a1. The caller supplies a `version` query parameter with a value other than 2. The system returns HTTP 500 with `code` 500 and `text` of the form "unknown version: N". The `data` field is absent from the response body.

---

## Suspected Defects

### Implementation defects only

**Double assignment of `email` in `AgencyBeanServiceImpl`** (`AgencyBeanServiceImpl.java`, line 57).
`bean.setEmail(agency.getEmail())` is called twice: once at line 51 and again at line 57. The second call overwrites the first with the same value. There is no observable effect at the API boundary. A clean reimplementation needs only a single assignment.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Agency identifier, encoded directly in the URL path segment."
    },
    "key": {
      "type": "string",
      "description": "API access key."
    },
    "version": {
      "type": "integer",
      "default": 2,
      "description": "Response format version. Only version 2 is currently supported."
    },
    "includeReferences": {
      "type": "boolean",
      "default": true,
      "description": "Accepted for compatibility with other endpoints; has no observable effect here because the agency entry is the primary resource and is never placed in the references block."
    },
    "time": {
      "type": "integer",
      "description": "Unix ms. Accepted for compatibility with other endpoints; has no observable effect here because this endpoint performs no time-sensitive computation.",
      "default": null
    }
  }
}
```

**`id`** — The agency identifier, taken directly from the URL path. Agency IDs correspond to the `agency_id` values in the GTFS feed loaded by the server. IDs are case-sensitive and must match exactly.

**`key`** — API access key issued by the server operator. Required by convention for all API calls.

**`version`** — Selects the response format version. Only version 2 is currently supported; any other value produces an HTTP 500 error.

**`includeReferences`** — When `false`, suppresses population of the shared references block on endpoints that embed cross-references. This endpoint does not embed any cross-references, so the parameter has no effect and the references block is always returned empty.

**`time`** — Overrides the server's current time for endpoints that perform time-sensitive queries. This endpoint does not use the current time, so the parameter has no effect.

---

## Response Structure

### Envelope

```json
{
  "type": "object",
  "properties": {
    "version": { "type": "integer" },
    "code":    { "type": "integer" },
    "currentTime": { "type": "integer", "description": "Unix ms" },
    "text":    { "type": "string" },
    "data":    { "type": "object" }
  }
}
```

**`version`** — The response format version in use (reflects the requested `version` parameter, or 2 by default).

**`code`** — HTTP-equivalent status code embedded in the response body (200, 404, or 500 for the scenarios described above).

**`currentTime`** — Server wall-clock time at the moment the response was generated, in Unix milliseconds.

**`text`** — Human-readable status message: `"OK"` on success, `"resource not found"` on 404, or `"unknown version: N"` on an unsupported version.

**`data`** — Present only on success (HTTP 200). Absent on error responses.

---

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

**`data.entry`** — The agency record. See `data.entry` schema below.

**`data.references`** — Always present and always empty for this endpoint. Contains the sub-keys `agencies`, `routes`, `stops`, `trips`, `situations`, and `stopTimes`, each an empty array.

---

### data.entry

```json
{
  "type": "object",
  "required": ["id", "name", "url", "timezone", "privateService"],
  "properties": {
    "id":             { "type": "string" },
    "name":           { "type": "string" },
    "url":            { "type": "string" },
    "timezone":       { "type": "string" },
    "lang":           { "type": "string" },
    "phone":          { "type": "string" },
    "email":          { "type": "string" },
    "fareUrl":        { "type": "string" },
    "disclaimer":     { "type": "string" },
    "privateService": { "type": "boolean" }
  }
}
```

**`data.entry.id`** — The agency identifier. Matches the `agency_id` value from the GTFS feed.

**`data.entry.name`** — The full public name of the agency (GTFS `agency_name`). Always present.

**`data.entry.url`** — The agency's public website URL (GTFS `agency_url`). Always present.

**`data.entry.timezone`** — IANA timezone name for the agency's operating area (GTFS `agency_timezone`), e.g. `"America/Los_Angeles"`. All scheduled times in the feed are interpreted relative to this timezone. Always present.

**`data.entry.lang`** — ISO 639-1 two-letter language code for the primary language of the agency's rider-facing text (GTFS `agency_lang`). Optional in GTFS; may be `null` or an empty string when not supplied.

**`data.entry.phone`** — Customer-service phone number for the agency (GTFS `agency_phone`). Optional in GTFS; may be `null` or an empty string when not supplied.

**`data.entry.email`** — Customer-service email address (GTFS `agency_email`). Optional in GTFS; may be `null` or an empty string when not supplied.

**`data.entry.fareUrl`** — URL to the agency's fare information page (GTFS `agency_fare_url`). Optional in GTFS; may be `null` or an empty string when not supplied.

**`data.entry.disclaimer`** — A legal or attribution disclaimer the agency requires to be displayed when its data is used in an application. Not a GTFS standard field; sourced from OBA's supplemental agency narrative data. May be `null` or an empty string when not configured.

**`data.entry.privateService`** — `true` if the agency provides private service not available to the general public. Not a GTFS standard field; sourced from OBA's supplemental agency narrative data. Always present; defaults to `false`.
