# config

## Goal in Context

Return the active data bundle's identity, service date range, and server build information so that clients can verify data currency, detect version drift, and confirm the API is operational.

## Scope

OBA REST API — this endpoint only.

## Level

Sub-function.

## Primary Actor

Rider (accessing the system via a client app).

## Stakeholders and Interests

- **Rider** — wants assurance that the transit data being served is current and that the server is reachable.

## Preconditions

- The API server is running and a data bundle is loaded.
- The caller holds a valid, non-rate-limited API key.

## Minimal Guarantees

- Every response includes `code` and `currentTime`.
- Key-validation failures return HTTP 401 or 429 with an explanatory message.

## Success Guarantees

- The caller receives the active bundle's identifier, name, service date range, and the server's software build properties.
- All four string metadata fields in the response entry are always present and non-null (they are empty strings when the bundle has no metadata file, never absent).

## Trigger

A client issues `GET /api/where/config.json` with a valid API key.

## Main Success Scenario

1. The server validates the API key via its key-validation interceptor.
2. The server reads the active bundle's metadata from a JSON file in the bundle directory. ([`BundleConfigDao.setup`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/config/BundleConfigDao.java#L52-L76))
3. The server loads its software build metadata from the `git.properties` resource on its classpath. ([`BeanFactoryV2.getGitProperties`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1638-L1649))
4. The server assembles a configuration entry from the bundle identity fields and build properties, then returns HTTP 200 with the entry in a standard JSON envelope. ([`ConfigAction.index`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ConfigAction.java#L54-L62))
5. The caller reads `data.entry.id` and `data.entry.name` to identify the bundle, `data.entry.serviceDateFrom` and `data.entry.serviceDateTo` to determine its temporal scope, and `data.entry.gitProperties` to identify the server software version.

## Extensions

**1a. Key absent or invalid:**
The key-validation interceptor returns HTTP 401 with `{"code":401,"text":"permission denied","version":1}`. The `data` field is absent from the response.

**1b. Key rate limit exceeded:**
The key-validation interceptor returns HTTP 429 with `{"code":429,"text":"rate limit exceeded","version":1}`. The `data` field is absent from the response.

**2a. Bundle metadata file absent or unparseable:**
The metadata loading step returns null. The response entry's `id`, `name`, `serviceDateFrom`, and `serviceDateTo` fields are all serialized as empty strings `""`. Build properties are unaffected. ([`BeanFactoryV2.getConfig`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L450-L458))

**3a. Build properties resource absent from classpath:**
`gitProperties` is present in the response as an empty object `{}`.

**3b. IO error reading build properties:**
`gitProperties` is absent from the response entry entirely.

## Suspected Defects

### Defects that affect the use case

**`ApiKeyInterceptor` — key-validation error responses hardcode version 1**

[`ApiKeyInterceptor.java` line 107](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/ApiKeyInterceptor.java#L107)

When the key-validation interceptor rejects a request (401 or 429), it constructs the error response with a hard-coded version of 1, regardless of the `version` parameter supplied by the caller. All other error responses (validation errors, server exceptions) echo back the version the caller requested. A caller requesting the default version receives `"version":1` in 401 and 429 responses, inconsistent with the `"version":2` in the success response for the same request. The intended behaviour is to echo the caller's version consistently.

### Implementation defects only

**`ConfigAction` — debug log left at error level in `getId`**

[`ConfigAction.java` line 50](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/ConfigAction.java#L50)

The `getId` method emits `_log.error("in id!")` unconditionally. This is a debug statement accidentally committed at error level. The method exists as a Struts2 REST framework property accessor and is not invoked during normal `index` action processing, but if the framework ever calls it, it spams the server error log.

**`BeanFactoryV2` — inconsistent handling of absent vs. unreadable build-properties resource**

[`BeanFactoryV2.java` lines 1638–1649](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/transit/BeanFactoryV2.java#L1638-L1649)

When the `git.properties` classpath resource is absent, `getGitProperties` returns an empty `Properties` object, so `gitProperties` appears in the response as `{}`. When an `IOException` occurs while reading the resource, the method returns `null`, so `gitProperties` is absent from the response entirely. Both paths indicate that build information is unavailable, but they produce different response shapes.

## Open Questions

None.

---

## Request Parameters

```json
{
  "type": "object",
  "properties": {
    "key":               { "type": "string" },
    "version":           { "type": "integer", "default": 2 },
    "includeReferences": { "type": "boolean", "default": true },
    "callback":          { "type": "string" },
    "time":              { "type": "integer", "description": "Unix ms" }
  },
  "required": ["key"]
}
```

**`key`** — API key. All requests without a valid key are rejected before the action runs.

**`version`** — Protocol version. Any integer is accepted and echoed back in the response `version` field. This endpoint does not vary its behaviour by version; the parameter exists for consistency with other endpoints.

**`includeReferences`** — Whether to populate the references block with referenced transit entities. Has no observable effect here because the references block is always empty for this endpoint.

**`callback`** — If present, the response body is wrapped as a JSONP call — `<callback>(…)` — and the `Content-Type` changes to `application/javascript`.

**`time`** — Request time as Unix milliseconds. Accepted by the framework but not used by this endpoint.

---

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

**`code`** — HTTP status code echoed in the body: 200 on success.

**`currentTime`** — Server time at the moment the response was assembled, in Unix milliseconds.

**`text`** — Human-readable status: `"OK"` on success.

**`version`** — The value of the `version` request parameter, or 2 when none was supplied.

**`data.entry`** — Configuration object; see schema below.

**`data.references`** — Always present. Contains empty arrays for all reference types (`agencies`, `routes`, `stops`, `trips`, `situations`, `stopTimes`). This endpoint does not reference any transit entities.

### data.entry

```json
{
  "type": "object",
  "properties": {
    "id":              { "type": "string" },
    "name":            { "type": "string" },
    "serviceDateFrom": { "type": "string" },
    "serviceDateTo":   { "type": "string" },
    "gitProperties": {
      "type": "object",
      "additionalProperties": { "type": "string" }
    }
  }
}
```

**`data.entry.id`** — Unique identifier of the active data bundle. Empty string when the bundle has no metadata file.

**`data.entry.name`** — Human-readable name of the active data bundle. Empty string when the bundle has no metadata file.

**`data.entry.serviceDateFrom`** — Start of the date range covered by the active bundle's transit data. The string format is set by the bundle builder; it is not parsed or normalised at the API layer. Empty string when the bundle has no metadata file.

**`data.entry.serviceDateTo`** — End of the date range covered by the active bundle's transit data. Same format caveats as `serviceDateFrom`. Empty string when the bundle has no metadata file.

**`data.entry.gitProperties`** — Key-value map of server software build properties, loaded from the `git.properties` classpath resource at response time. Keys are sorted alphabetically. Typical keys include `git.commit.id`, `git.branch`, `git.build.version`, `git.build.time`, and `git.commit.message.short`. Present as `{}` when the resource is absent from the classpath; absent from the response entirely when an IO error occurs while reading it (see Extension 3b).
