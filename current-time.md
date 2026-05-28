# Use Case: current-time

## Goal in Context

A rider's client app needs to know the server's current wall-clock time so that it can anchor schedule queries and determine which service day is currently active, even if the device's own clock is inaccurate.

## Scope

OneBusAway REST API — `GET /api/where/current-time.json`

## Level

User goal

## Primary Actor

Rider (via a client application)

## Stakeholders and Interests

**Rider** — wants the current time as understood by the transit data server, expressed both as a machine-readable Unix timestamp and as a human-readable string, so that subsequent schedule queries are aligned with the server's view of time.

## Preconditions

- The caller supplies a valid API key.

## Minimal Guarantees

- The response always carries HTTP status 200.
- The response body always includes a JSON envelope with `version`, `code`, `text`, and `currentTime` fields.

## Success Guarantees

- `data.entry.time` contains the server's current time as a Unix millisecond timestamp.
- `data.entry.readableTime` contains the same instant formatted as an ISO 8601 string.
- `data.references` is present and contains empty arrays for all entity types.

## Trigger

The client sends `GET /api/where/current-time.json?key=<key>`.

## Main Success Scenario

1. The client sends a request with a valid API key.
2. The server reads the current system time as a Unix millisecond value. ([`CurrentTimeAction.java#L43`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/CurrentTimeAction.java#L43))
3. The server formats the same instant as an ISO 8601 string, including a UTC offset drawn from the JVM's configured default timezone. ([`CurrentTimeAction.java#L44`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/CurrentTimeAction.java#L44), [`DateLibrary.java#L37-L38`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-core/src/main/java/org/onebusaway/utility/DateLibrary.java#L37-L38))
4. The server returns HTTP 200 with an envelope wrapping the two time values and an empty references block.

## Extensions

_None._ This endpoint has no error paths beyond those handled by the common infrastructure (e.g. an unrecognised API version returns 500; a missing `key` may be rejected by API key validation middleware upstream of the action).

---

## Suspected Defects

### Defects that affect the use case

**1. `readableTime` is formatted in the JVM's default timezone, not the feed's timezone.**

`CurrentTimeAction` calls [`DateLibrary.getTimeAsIso8601String(date)`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/CurrentTimeAction.java#L44), which delegates to [`DateLibrary.getTimeAsIso8601String(date, TimeZone.getDefault())`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-core/src/main/java/org/onebusaway/utility/DateLibrary.java#L37-L38). This uses whatever timezone the JVM process happens to be running in, which may differ from the transit feed's configured local timezone. The endpoint-transit-concepts document states that the purpose of the readable time field is to help clients determine the active service day; this requires the feed's own timezone, not the server process's timezone. A deployment whose JVM timezone differs from the feed's timezone would produce a `readableTime` with the wrong UTC offset for service-day calculations. The intended timezone for `readableTime` was probably the feed's configured local timezone.

**2. The envelope's `currentTime` and `data.entry.time` are sampled at different points, so they can differ.**

`data.entry.time` is sampled inside the action method ([`CurrentTimeAction.java#L43`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/CurrentTimeAction.java#L43)). The envelope's `currentTime` field is sampled separately when `ResponseBean` is constructed ([`ResponseBean.java#L32`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-core/src/main/java/org/onebusaway/api/model/ResponseBean.java#L32)), which happens a few milliseconds later. For most endpoints this discrepancy is invisible, but for `current-time` it means the two fields that both purport to express "now" may disagree by a small amount (observed: 2 ms). The intended behaviour was probably for both values to reflect the same instant.

### Implementation defects only

**3. `CurrentTimeAction` allocates a `Date` object and then immediately discards its value.**

[`CurrentTimeAction.java#L42-L43`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-api-webapp/src/main/java/org/onebusaway/api/actions/api/where/CurrentTimeAction.java#L42-L43): `new Date()` internally samples `System.currentTimeMillis()`, and the very next line overwrites the date's value with `SystemTime.currentTimeMillis()`. The first sample is never used. This has no observable effect because both calls return the same value in normal operation, but the intermediate allocation is unnecessary.

---

## Open Questions

_None._

---

## Request Parameters

```json
{
  "type": "object",
  "properties": {
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
  "required": ["key"]
}
```

**`key`** — API access key. Required on every request.

**`version`** — Selects the response envelope format. Only version 2 is supported; passing any other value returns a 500 error.

**`includeReferences`** — When `false`, the `data.references` block is omitted from the response. Because this endpoint never populates references, the only practical effect is a marginally smaller payload.

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
    "data": { "type": "object" }
  }
}
```

**`version`** — Always `2`.

**`code`** — HTTP-style status code mirrored inside the body. Always `200` for this endpoint.

**`text`** — Always `"OK"` for this endpoint.

**`currentTime`** — The server's current time as Unix milliseconds, sampled at response-construction time. See Suspected Defect 2 for the subtle difference between this value and `data.entry.time`.

**`data`** — The payload object containing `entry` and `references`.

### data.entry

```json
{
  "type": "object",
  "properties": {
    "time":         { "type": "integer", "description": "Unix ms" },
    "readableTime": { "type": "string" }
  }
}
```

**`data.entry.time`** — The server's current time as Unix milliseconds, sampled when the action executes.

**`data.entry.readableTime`** — The same instant formatted as an ISO 8601 string (e.g. `2026-05-08T15:44:29-07:00`). The UTC offset reflects the JVM's default timezone, not necessarily the feed's timezone — see Suspected Defect 1.

### data.references

```json
{
  "type": "object",
  "properties": {
    "agencies":   { "type": "array", "items": { "type": "object" } },
    "routes":     { "type": "array", "items": { "type": "object" } },
    "stops":      { "type": "array", "items": { "type": "object" } },
    "trips":      { "type": "array", "items": { "type": "object" } },
    "situations": { "type": "array", "items": { "type": "object" } },
    "stopTimes":  { "type": "array", "items": { "type": "object" } }
  }
}
```

**`data.references`** — Always present (unless suppressed by `includeReferences=false`). All arrays are always empty for this endpoint because no transit entities are resolved.
