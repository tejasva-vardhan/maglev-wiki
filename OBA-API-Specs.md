# OBA API Endpoint Specifications

Cockburn-style use case specs generated from the live Java implementation.
Each page covers accepted parameters, response structure, error conditions,
and suspected defects.

For the authoritative Maglev 1.0 endpoint list, see [[Maglev-1.0-Endpoint-Scope]].

## Agencies

| Endpoint | Page |
|----------|------|
| `GET /api/where/agencies-with-coverage.json` | [[agencies-with-coverage]] |
| `GET /api/where/agency/{id}.json` | [[agency]] |

## Routes

| Endpoint | Page |
|----------|------|
| `GET /api/where/route/{id}.json` | [[route]] |
| `GET /api/where/routes-for-agency/{id}.json` | [[routes-for-agency]] |
| `GET /api/where/route-ids-for-agency/{id}.json` | [[route-ids-for-agency]], [[routes-for-agency]] |
| `GET /api/where/routes-for-location.json` | [[routes-for-location]] |

## Stops

| Endpoint | Page |
|----------|------|
| `GET /api/where/stop/{id}.json` | [[stop]] |
| `GET /api/where/stops-for-agency/{id}.json` | [[stops-for-agency]] |
| `GET /api/where/stop-ids-for-agency/{id}.json` | [[stop-ids-for-agency]], [[stops-for-agency]] |
| `GET /api/where/stops-for-location.json` | [[stops-for-location]] |
| `GET /api/where/stops-for-route/{id}.json` | [[stops-for-route]] |

## Trips

| Endpoint | Page |
|----------|------|
| `GET /api/where/trip/{id}.json` | [[trip]] |
| `GET /api/where/trip-details/{id}.json` | [[trip-details]] |
| `GET /api/where/trips-for-route/{id}.json` | [[trips-for-route]] |
| `GET /api/where/trips-for-location.json` | [[trips-for-location]] |

## Schedules & Arrivals

| Endpoint | Page |
|----------|------|
| `GET /api/where/schedule-for-stop/{id}.json` | [[schedule-for-stop]] |
| `GET /api/where/schedule-for-route/{id}.json` | [[schedule-for-route]] |
| `GET /api/where/arrivals-and-departures-for-stop/{id}.json` | [[arrivals-and-departures-for-stop]] |
| `GET /api/where/arrival-and-departure-for-stop/{stopId}.json` | [[arrival-and-departure-for-stop]] |
| `GET /api/where/trip-for-vehicle/{id}.json` | [[trip-for-vehicle]] |

## Vehicles

| Endpoint | Page |
|----------|------|
| `GET /api/where/vehicles-for-agency/{id}.json` | [[vehicles-for-agency]] |

## Search

| Endpoint | Page |
|----------|------|
| `GET /api/where/search/route.json` | [[search-route]] |
| `GET /api/where/search/stop.json` | [[search-stop]] |

## Utilities

| Endpoint | Page |
|----------|------|
| `GET /api/where/config.json` | [[config]] |
| `GET /api/where/current-time.json` | [[current-time]] |
| `GET /api/where/shape/{id}.json` | [[shape]] |

## On-Demand/GTFS-Flex

General Spec: [GTFS-Flex Support](/OneBusAway/maglev/wiki/GTFS-Flex%20Support)

## Shared Algorithms and Response Rules

Behavior shared across multiple endpoint pages, documented once and linked from each.

| Topic | Page |
|-------|------|
| Boolean query flags: omitted defaults, empty values, and strict validation | [[Boolean-Query-Parameters]] |
| Stop `direction` field inference from shape geometry | [[Stop-Direction-Calculation]] |
| Service-alert situation IDs and `references.situations` inclusion | [[Situation-References]] |
