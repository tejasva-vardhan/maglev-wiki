# Maglev 1.0 Endpoint Scope

Maglev 1.0 intentionally implements a bounded set of OneBusAway endpoints. The scope is the OpenAPI-documented OBA API plus explicitly approved Maglev extensions and GTFS Realtime protobuf feeds.

Maglev 1.0 is **not** intended to reproduce every endpoint that exists in the legacy Java OneBusAway server.

## Contributor guidance

- If an endpoint is not listed on this page, do not implement it for Maglev 1.0 just because it exists in the legacy server.
- The legacy server contains undocumented, experimental, or internal endpoints. Those endpoints are out of scope by default.
- New endpoints require an explicit scope decision and should be documented/speced before implementation.

## In-scope OBA REST JSON endpoints

### Agencies

- `/api/where/agencies-with-coverage.json`
- `/api/where/agency/{id}.json`

### Routes

- `/api/where/route/{id}.json`
- `/api/where/route-ids-for-agency/{id}.json`
- `/api/where/routes-for-agency/{id}.json`
- `/api/where/routes-for-location.json`

### Stops

- `/api/where/stop/{id}.json`
- `/api/where/stop-ids-for-agency/{id}.json`
- `/api/where/stops-for-agency/{id}.json`
- `/api/where/stops-for-location.json`
- `/api/where/stops-for-route/{id}.json`

### Schedules and arrivals

- `/api/where/arrival-and-departure-for-stop/{id}.json`
- `/api/where/arrivals-and-departures-for-location.json`
- `/api/where/arrivals-and-departures-for-stop/{id}.json`
- `/api/where/schedule-for-route/{id}.json`
- `/api/where/schedule-for-stop/{id}.json`
- `/api/where/trip-for-vehicle/{id}.json`

### Trips

- `/api/where/trip/{id}.json`
- `/api/where/trip-details/{id}.json`
- `/api/where/trips-for-location.json`
- `/api/where/trips-for-route/{id}.json`

### Vehicles

- `/api/where/vehicles-for-agency/{id}.json`

### Search

- `/api/where/search/route.json`
- `/api/where/search/stop.json`

### Utilities

- `/api/where/config.json`
- `/api/where/current-time.json`
- `/api/where/shape/{id}.json`

### Problem reports

- `/api/where/report-problem-with-stop/{id}.json`
- `/api/where/report-problem-with-trip/{id}.json`

## In-scope Maglev extensions

These endpoints are intentionally added by Maglev and are in scope for 1.0:

- `/api/where/problem-reports-for-stop/{id}.json`
- `/api/where/problem-reports-for-trip/{id}.json`
- `/api/v2/metadata.json`

## In-scope GTFS Realtime protobuf endpoints

These are separate from the OBA REST JSON API but are in scope for Maglev 1.0:

- `/api/gtfs_realtime/vehicle-positions-for-agency/{id}.pb`
- `/api/gtfs_realtime/alerts-for-agency/{id}.pb`
- `/api/gtfs_realtime/trip-updates-for-agency/{id}.pb`

## Out-of-scope legacy endpoints

Any legacy Java endpoint not listed above is out of scope for Maglev 1.0 unless a later scope decision explicitly adds it. This includes undocumented legacy endpoints discovered by code inspection.
