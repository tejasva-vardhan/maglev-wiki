## Bugs in the Implementation

### 1. `references.trips[].routeId` and `references.trips[].serviceId` not combined

**File:** `internal/restapi/schedule_for_route_handler.go`, lines 286–295

```go
tripRef := models.NewTripReference(
    combinedTripID,
    t.RouteID,     // ← raw DB value, missing agency prefix
    t.ServiceID,   // ← raw DB value, missing agency prefix
    ...
)
```

Every other handler that builds trip references (`schedule_for_stop_handler.go:289`, both arrivals handlers) wraps these with `utils.FormCombinedID(agencyID, t.RouteID)`. The trip references emitted by this endpoint have bare route and service IDs instead of `{agencyId}_{id}` strings.

**No existing test catches this** — `TestScheduleForRouteHandler_DirectionIDJavaParity` checks `directionId` inside trip references but not `routeId` or `serviceId`.

---

### 2. `arrivalEnabled` / `departureEnabled` always `true`

**File:** `internal/restapi/schedule_for_route_handler.go`, lines 221–223

```go
ArrivalEnabled:   true,
DepartureEnabled: true,
```

The spec defines these as `true` only when the time value is greater than 0. The implementation hardcodes `true` unconditionally. A stop scheduled at exactly midnight (GTFS time == 0) would be misrepresented as having a published arrival time.

---

### 3. Numeric timestamp not accepted for `date` parameter

**File:** `internal/restapi/schedule_for_route_handler.go`, lines 21–28

The spec says:

> A numeric value is treated as a Unix milliseconds timestamp.

`utils.ValidateDate` only accepts `yyyy-MM-dd` strings. A numeric timestamp like `1749686400000` returns HTTP 400. The utility `utils.ParseTimeParameter` already handles numeric timestamps correctly but is not used by this handler.

---

## Missing Spec Behaviours: 510 Responses

The spec defines two distinct 510 cases. The implementation collapses both into HTTP 200 with empty data.

| Spec case | Spec says | Implementation does |
|---|---|---|
| **`ServiceDateOutOfRange`** — queried date falls after all calendar dates for every trip on the route | JSON code 510, text `"ServiceDateOutOfRange"`, **no `data` field** | HTTP 200, JSON code 200, `data` present with an empty entry and empty references |
| **`NoServiceThatDay`** — date is within calendar range but no trips run on that date | JSON code 510, text `"NoServiceThatDay"`, `data` present with `routeId`, `scheduleDate`, empty arrays, and **references populated with agency and route records** | HTTP 200, JSON code 200, `data` present with an empty entry and **empty references** |

The comment in the handler ("Behavior Change Jan 2026: Previously returned 500") explains the intent to return 200 rather than 500. However the result still diverges from the spec in two ways:

- The envelope `code` is 200 instead of 510.
- The `NoServiceThatDay` response omits the agency and route references that the spec requires.

Note: The Java OBA implementation also returns HTTP 200 for 510 responses (flagged as a defect in the spec), so the HTTP status mismatch is low priority. The missing references and wrong JSON code are higher priority.

---

## Test Gaps

| Missing test | Why it matters |
|---|---|
| `references.trips[].routeId` is a combined `{agencyId}_{routeId}` string | The current bug (#1 above) would go undetected |
| Numeric `date` param (e.g. `?date=1749686400000`) | Returns HTTP 400 today; spec requires HTTP 200 |
| `arrivalEnabled` / `departureEnabled` is `false` for a zero arrival time | Hardcoded `true` would go undetected |
| Response shape for the out-of-range date case | No coverage of the empty-schedule branch's references |
| Response shape for the no-service-that-day case | No coverage that agency/route refs are included |

---

## Intentional Improvements Over the Spec

The spec documents two "suspected defects" from the Java OBA implementation. The Go implementation fixes both:

| Field | Java / spec behaviour | Go implementation |
|---|---|---|
| `stopTimes[].serviceId` | Always empty string (defect) | Correctly set to `{agencyId}_{serviceId}` |
| `stopTimes[].stopHeadsign` | Always empty string (defect) | Read from the database when available |

These are improvements, not bugs.