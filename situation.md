# situation

## Goal in Context

A rider's client app has a service-alert ("situation") ID — typically harvested from another response's `situationIds` field or `references.situations` block — and wants the full alert record: what's affected, why, how severe it is, and the human-readable summary/description text.

## Scope

OneBusAway public REST API, v2.

## Level

User goal.

## Primary Actor

Rider (indirectly, via a client app resolving a situation ID it already has).

## Stakeholders and Interests

**Rider** — wants to read the full detail of a service disruption they've already been told about (e.g. via a trip's `situationIds`), without the client having to carry the alert's full payload everywhere it's referenced.

## Preconditions

- The caller supplies a valid API key.
- The situation ID supplied is in the combined `{agencyId}_{entityId}` form.

## Minimal Guarantees

The API always returns a well-formed JSON (or XML) envelope containing a `code`, `text`, and `currentTime`.

## Success Guarantees

- A 200 response is returned whose `data.entry` describes the matching service alert.
- `data.references` is present but its `agencies`, `routes`, `stops`, `trips`, and `situations` arrays are always empty (see Main Success Scenario, step 5).

## Trigger

An HTTP GET to `/api/where/situation/{id}.json` (or `.xml`).

## Main Success Scenario

1. The caller issues a GET request with the required `{id}` path segment set to the situation's combined ID (`{agencyId}_{entityId}`). [Source: [`SituationAction.java#L43-L57`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/SituationAction.java#L43-L57)]
2. `SituationAction.show()` delegates the raw ID string to `TransitDataService.getServiceAlertForId`, which — after waiting for the active GTFS bundle to be ready — forwards it to the per-bundle `TransitDataServiceTemplateImpl`. [Source: [`SituationAction.java#L57`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/SituationAction.java#L57), [`TransitDataServiceImpl.java#L469-L472`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/federated/TransitDataServiceImpl.java#L469-L472)]
3. `TransitDataServiceTemplateImpl.getServiceAlertForId` parses the raw string into an `AgencyAndId` via `AgencyAndIdLibrary.convertFromString`, splitting on the first `_` into an agency ID and an entity ID. [Source: [`TransitDataServiceTemplateImpl.java#L591-L595`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/federated/TransitDataServiceTemplateImpl.java#L591-L595), [`AgencyAndIdLibrary.java#L40-L53`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-util/src/main/java/org/onebusaway/util/AgencyAndIdLibrary.java#L40-L53)]
4. The parsed `AgencyAndId` is looked up directly against the in-memory service-alert cache — a `Map<AgencyAndId, ServiceAlertRecord>` — with no further scanning or filtering. [Source: [`ServiceAlertsServiceImpl.java#L182-L185`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-alerts-persistence/src/main/java/org/onebusaway/alerts/impl/ServiceAlertsServiceImpl.java#L182-L185), via [`ServiceAlertsBeanServiceImpl.java#L90-L95`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/ServiceAlertsBeanServiceImpl.java#L90-L95)]
5. If found, the record is converted into a `SituationV2Bean` — carrying `id`, `creationTime`, `activeWindows`, `publicationWindows`, `allAffects`, `consequences`, `reason`, `consequenceMessage`, `summary`/`description`/`url` (each as a best-language-match translated string), and `severity` (the first TPEG code for the alert's `ESeverity`) — and wrapped in the standard single-entry envelope. [Source: [`BeanFactoryV2.java#L366-L369`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L366-L369), [`#L1384-L1436`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1384-L1436), [`#L1597-L1599`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1597-L1599)]
   Critically, nothing on this path calls `addToReferences` for any entity named in the alert's `allAffects` list — the response's shared references object is never populated for this request. `data.references` is therefore present in the envelope but every sub-array (`agencies`, `routes`, `stops`, `trips`, `situations`) is empty.
6. The system returns HTTP 200 with the assembled entry.

## Extensions

**2a. Well-formed ID, no matching alert:** the cache lookup in step 4 returns `null`; `SituationAction` explicitly checks for this and returns HTTP 404 with `code: 404`, `text: "resource not found"`. [Source: [`SituationAction.java#L57-L60`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/SituationAction.java#L57-L60)]

**2b. Malformed ID (no `_` separator):** `AgencyAndIdLibrary.convertFromString` throws an unchecked `IllegalStateException` in step 3, before the code in Extension 2a is ever reached. Nothing in the call chain catches it — it escapes `SituationAction.show()` and is handled the same way as every other uncaught exception on this endpoint family: HTTP 200 with a literal `null` body. See [Suspected Defects](#suspected-defects) and [Implementation Decisions](#implementation-decisions).

**2c. Unsupported API version requested via `version` parameter:** the system returns HTTP 500 with `code: 500` and `text: "unknown version: N"`. Only version 2 is supported. [Source: [`SituationAction.java#L62-L65`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/SituationAction.java#L62-L65)]

## Suspected Defects

### Defects that affect the use case

**Malformed ID returns HTTP 200 with a null body instead of an error status**

See Extension 2b. This is the same failure mechanism as the malformed-ID half of [DC-1](OBA-API-Defect-Workshop#dc-1-unknown-entity-or-malformed-id-returns-http-200-with-null-body) in the [[OBA-API-Defect-Workshop]] — `AgencyAndIdLibrary.convertFromString` throwing uncaught rather than being validated or caught. Unlike the endpoints DC-1 already lists, this endpoint's *unknown-but-well-formed-ID* case is not defective: `SituationAction` already explicitly null-checks and returns a correct 404 for that case (Extension 2a). Only the malformed-ID half applies here.

### Implementation defects only

**`@RequiredFieldValidator` on `setId` is unreachable**

The `@RequiredFieldValidator` annotation on `SituationAction.setId` is dead code. [Source: [`SituationAction.java#L43-L46`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/SituationAction.java#L43-L46)] The ID is embedded in the URL path; when it is absent, the Struts2 framework returns a container-level 404 before the action is reached, so `hasErrors()` is never checked for this parameter. Same pattern as [[trip-for-vehicle]]'s equivalent note.

## Implementation Decisions

**Malformed ID returns HTTP 400 and unknown ID returns HTTP 404 (deviates from legacy).**

The legacy implementation returns HTTP 200 with a null body for the malformed-ID condition (see Suspected Defects); it already returns a correct HTTP 404 for a well-formed but unrecognized ID. Maglev intentionally corrects the malformed-ID case: a malformed ID (no underscore) returns HTTP 400 and an unknown ID returns HTTP 404, consistent with the treatment of malformed and unknown IDs elsewhere in the API. See [[OBA-API-Defect-Workshop#dc-1-unknown-entity-or-malformed-id-returns-http-200-with-null-body|DC-1]].

## Open Questions

None.

---

## Request Parameters

```json
{
  "type": "object",
  "required": ["id", "key"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Path parameter. Combined situation (service alert) ID in {agencyId}_{entityId} form."
    },
    "key": {
      "type": "string",
      "description": "API key."
    },
    "version": {
      "type": "integer",
      "default": 2,
      "description": "API version. Only 2 is supported; other values produce a 500 error."
    }
  }
}
```

**`id`** — Combined situation identifier, encoded directly in the URL path: `/api/where/situation/{id}.json`. The agency portion is everything before the first underscore; the rest is the entity ID. Example: `1_92239`.

**`key`** — API authentication key. Required on all requests.

**`version`** — API version. Only `2` is supported.

## Response Structure

### Envelope

```json
{
  "code": 200,
  "currentTime": 1700000000000,
  "text": "OK",
  "version": 2,
  "data": {
    "entry": { "...": "SituationV2Bean, see below" },
    "references": {
      "agencies": [],
      "routes": [],
      "stops": [],
      "trips": [],
      "situations": []
    }
  }
}
```

### `data.entry`

```json
{
  "id": "1_92239",
  "creationTime": 1700000000000,
  "reason": "TECHNICAL_PROBLEM",
  "severity": "noImpact",
  "consequenceMessage": "",
  "summary": { "value": "Service disruption", "lang": "en" },
  "description": { "value": "Detour in effect", "lang": "en" },
  "url": { "value": "https://example.org/alert/1", "lang": "en" },
  "activeWindows": [ { "from": 1700000000000, "to": 1700003600000 } ],
  "publicationWindows": [],
  "allAffects": [
    { "agencyId": "1", "routeId": "1_100", "stopId": "", "tripId": "", "applicationId": "", "directionId": "" }
  ],
  "consequences": []
}
```

**`data.entry.id`** — The situation's combined ID, echoed back from the request.

**`data.entry.reason`** — Machine-readable cause of the disruption.

**`data.entry.severity`** — TPEG severity code (e.g. `noImpact`, `slight`, `severe`).

**`data.entry.summary` / `description` / `url`** — Best-language-match translated strings. Absent (not an empty object) when the alert carries no text for that field.

**`data.entry.activeWindows`** — Time ranges (Unix millisecond `from`/`to`) during which the alert is active. Empty array when the alert has no active-window restriction.

**`data.entry.allAffects`** — The entities (agency, route, stop, trip, application, direction) this alert applies to.

**`data.references`** — Always present with every sub-array empty, regardless of what `allAffects` names (see Main Success Scenario, step 5). Not populated even for the entities named in `data.entry.allAffects`.
