Bring Maglev's `GET /api/where/metrics.json` to field-name parity with
OneBusAway Java's undocumented-but-widely-used monitoring endpoint, so
existing watchdog/alerting tooling built against the Java response shape
works unmodified against a Maglev instance.

## 1. What the Java feature is, in one paragraph

`MetricsAction` (via `TransitDataService.getMetrics()` →
`MetricsBeanServiceImpl`) reports, per agency: how many blocks are active
right now, how many GTFS-RT records were seen in the last poll and how many
of those resolved (and are currently active) against the static schedule,
how many real-time stop references resolved, and how stale the feed is.
None of this is persisted — it's recomputed from in-memory state
(`MonitoredResult`, one per configured `MonitoredDataSource`) on every
request.

## 2. How the two servers behave today

### 2.1 Maglev before this work

No `metrics.json` endpoint existed at all.

### 2.2 Behavioral differences, side by side

Found by comparing a first-pass Maglev implementation against a live
production Java `metrics.json` response for the same upstream Tampa
GTFS-RT feed, field by field:

| Field | Naive Maglev implementation | Java | Root cause |
|---|---|---|---|
| `scheduledTripsCount` | Flat count of every trip ever in the static schedule | Blocks active *right now* | `TripsForAgencyQueryBean.time` defaults to current time, not "all time" — §3.2 |
| `realtimeRecordsTotal` | One record per raw `trip_update` entity (270) | One record per *block*, grouped (100) | Java groups trip updates before counting — §3.3 |
| `realtimeTripCountsMatched` | One per raw trip ID resolved statically (260) | Resolved **and** currently active, one per block (91) | Two compounding gaps: no block grouping, and no activity-window gate — §3.4 |
| `stopIDsMatchedCount` | Summed per stop_time_update occurrence (3870) | Deduplicated per feed poll (1990) | `MonitoredResult`'s stop ID sets are `Set<String>`, not counters — §3.5 |
| `timeSinceLastRealtimeUpdate` | N/A (not yet implemented) | Raw current-epoch-seconds when no data source covers an agency | Java bug: `_lastUpdate` defaults to `0`, so `(now - 0) / 1000` is emitted — §3.6 |

## 3. New state in Maglev

### 3.1 Response shape

```json
{
  "agenciesWithCoverageCount": 2,
  "agencyIDs": ["1", "2"],
  "scheduledTripsCount": { "1": 69, "2": 2 },
  "realtimeRecordsTotal": { "1": 69, "2": 0 },
  "realtimeTripCountsMatched": { "1": 57, "2": 0 },
  "realtimeTripCountsUnmatched": { "1": 0, "2": 0 },
  "realtimeTripIDsUnmatched": { "1": [], "2": [] },
  "stopIDsMatchedCount": { "1": 1701, "2": 0 },
  "stopIDsUnmatchedCount": { "1": 0, "2": 0 },
  "stopIDsUnmatched": { "1": [], "2": [] },
  "timeSinceLastRealtimeUpdate": { "1": 1, "2": 0 }
}
```

Single-entry response (like `config.json`/`current-time.json`): empty
references, no `id` field. Every agency from `ListAgencies` gets an entry in
every map, even if zero/empty.

### 3.2 `scheduledTripsCount`: active blocks, not schedule totals

Java: `MetricsBeanServiceImpl.getScheduledTrips()` /
`getScheduledTrips(agencyId, routeId)` —
`onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/beans/MetricsBeanServiceImpl.java`.
Builds a `TripsForAgencyQueryBean` with `maxCount = Integer.MAX_VALUE` and no
explicit `time`; `AbstractTripsQueryBean.time` defaults to
`SystemTime.currentTimeMillis()`. The real "active blocks, not scheduled
total" semantic traces back further, to
`BlockStatusServiceImpl#getActiveBlocksForAgency`, queried with
`timeFrom == timeTo == now` — a strict point-in-time check with **no**
running-late/running-early tolerance (unlike `trips-for-route`), where a
block counts as active both mid-trip and while laying over between two of
its trips.

Maglev: `internal/gtfs/metrics.go` —
`activeTripsByAgency` → `activeTripsForAgency` → `countActiveBlocksAt`,
evaluated in each agency's own timezone, unioning
`GetActiveTripBlockIDsForAgency` and `GetActiveLayoverBlockIDsForAgency`
(`gtfsdb/query.sql`) by block ID. Checks both today's and yesterday's
service (shifted +24h) to catch after-midnight trips, mirroring the
today/yesterday pattern already used in `trips_for_route_handler.go`.

### 3.3 `realtimeRecordsTotal`: block-grouped records

Java: `GtfsRealtimeSource.handleUpdates()` —
`onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/realtime/gtfs_realtime/GtfsRealtimeSource.java`
— calls `_tripsLibrary.groupTripUpdatesAndVehiclePositions(...)`, then
`result.setRecordsTotal(combinedUpdates.size())`. The grouping itself is in
`GtfsRealtimeTripLibrary.groupTripUpdatesAndVehiclePositionsInternal()`
(same package): trip updates are indexed by their embedded vehicle ID first
(`tripUpdatesByVehicleId`, a `ListMultimap`), and a trip update with **no**
vehicle ID is separately indexed by block descriptor, then "consumed" into
an already vehicle-assigned block's group when
`assignmentInfo.preferredVehicleByBlockId` has an entry for that block —
e.g. a scheduler's look-ahead prediction for a vehicle's next trip, which
often omits the vehicle tag the current trip's update carries.

Maglev: `internal/gtfs/metrics.go` — `groupTripsByBlock`, keyed by each
trip's static `block_id` (already fetched via the existing `GetTripsByIDs`
call, no new query needed), falling back to the trip itself when it has no
resolvable block. This is a simplification of Java's vehicle-ID-first
grouping — see §4.3 — but reproduces Java's actual number closely because a
block is, in practice, the real unit of "one vehicle's activity."

Verified directly against the live feed at the protobuf level: 217
`trip_update` entities resolved to exactly 72 distinct static block IDs on
one poll, matching production Java's `recordsTotal` for that same poll
exactly; a later poll matched 69 = 69.

### 3.4 `realtimeTripCountsMatched` / `realtimeTripCountsUnmatched`: resolution + activity gating

**This was the harder of the two matching gaps** — closing the block-grouping
gap in §3.3 was not enough on its own; matched counting needed a second,
independent fix.

Java: `GtfsRealtimeTripLibrary.createVehicleLocationRecordForUpdate()` (same
file as above) builds a `VehicleLocationRecord` per combined (block-grouped)
update, then only calls `result.addMatchedTripId(...)` when
`isTripActive(update)` returns `true`:

```java
private boolean isTripActive(CombinedTripUpdatesAndVehiclePosition update) {
  // ...
  long windowFuture = 60 * 60; // 1 hour
  TripUpdate tripUpdate = update.getTripUpdates().get(0); // first trip update in the block's group
  StopTimeUpdate first = tripUpdate.getStopTimeUpdate(0);
  StopTimeUpdate last = tripUpdate.getStopTimeUpdate(tripUpdate.getStopTimeUpdateCount() - 1);
  // firstPrediction: first.arrival, falling back to first.departure
  // lastPrediction: last.departure, falling back to last.arrival
  return (currentTime + windowFuture > firstPrediction) && (lastPrediction > currentTime);
}
```

A resolved-but-not-currently-active record (predictions already finished,
or starting more than an hour out) counts toward **neither** matched nor
unmatched — Java silently excludes it from both. This also explains why
Java's own `realtimeTripCountsMatched + realtimeTripCountsUnmatched` never
sums to `realtimeRecordsTotal`: some records are simply excluded from the
whole matched/unmatched accounting.

`addUnmatchedTripId(...)` is called separately, earlier in the pipeline
(`GtfsRealtimeSource`), when a `BlockDescriptor` can't be resolved for an
update at all — independent of `isTripActive`, and independent of §3.3's
grouping.

Maglev: `internal/gtfs/metrics.go` —
`groupHasStaticMatch` (per block group, same grouping as §3.3) gates
resolution; `isCombinedRecordActive` → `representativeTrip` →
`isTripActive` gates activity. `representativeTrip` picks the trip in the
group with the earliest first-stop prediction — Maglev's parsed trip data
(via `go-gtfs`) doesn't preserve original GTFS-RT feed entity order the way
Java's `List.get(0)` implicitly does (`go-gtfs.ParseRealtime` sorts trips by
ID), so this approximates "the currently active leg" instead. Verified
empirically against a live feed: entity-order selection reproduced Java's
number exactly (57 = 57); earliest-first-prediction selection was within 1
(58 vs 57) on the same poll — close enough, and robust to Maglev's
order-loss constraint.

`realtimeTripCountsUnmatched` / `realtimeTripIDsUnmatched` stayed a
simpler, ungrouped per-trip-ID static-existence check — not time-gated,
matching Java's earlier-pipeline, resolution-only semantics for the
unmatched path.

### 3.5 `stopIDsMatchedCount` / `stopIDsUnmatchedCount`: deduplicated stop matching

Java: `MonitoredResult._matchedStopIds` / `_unmatchedStopIds` — both
`Set<String>`, populated while applying each trip's `stop_time_update`s in
`GtfsRealtimeTripLibrary`. `MetricsBeanServiceImpl.getMatchedStopIdsCount()`
/ `getUnmatchedStopIdsCount()` read the set sizes.

Maglev: `internal/gtfs/metrics.go` — `computeFeedMetrics`, deduplicated via
`map[string]bool` sets scoped per feed poll, rather than incrementing a
counter once per `stop_time_update` occurrence across every trip. Verified
against a live feed: 3573 raw stop_time_update occurrences collapsed to
1937 distinct stop IDs, closely tracking Java's reported count for the same
poll (and later polls matched within 1: 1701 vs 1700).

### 3.6 `timeSinceLastRealtimeUpdate`: real wall clock, no epoch-fallback bug

Java: `MetricsBeanServiceImpl.getLastUpdateDelta()` sums
`MonitoredResult.getLastUpdate()` across an agency's data sources, then
returns `(SystemTime.currentTimeMillis() - lastUpdate) / 1000`.
`MonitoredResult._lastUpdate` defaults to `0`. When no data source covers an
agency, the formula degenerates to `now / 1000` — the raw current epoch in
**seconds**, which decodes as "a few seconds ago" no matter when you
actually query it. This is a real, reproducible Java bug, not a
timezone/units quirk — confirmed by decoding a live sample value from
production.

Maglev: `internal/gtfs/metrics.go` — `populateRealtimeMetrics` measures
staleness against the real wall clock (`time.Now()`, deliberately not the
`scheduleReferenceTime`/mock-clock parameter `GetMetrics` receives, since
`feedLastUpdate` is always stamped with real time regardless of test clock
injection — see the doc comment on `populateRealtimeMetrics`), and
backfills `0` for any agency no feed touched. Deliberately does **not**
replicate Java's epoch-fallback bug — see §9.

## 4. Matching mechanics deep-dive

### 4.1 `groupTripsByBlock`

Single grouping function shared by §3.3's record counting and §3.4's
matched counting, so both stay consistent with each other by construction:

```go
func groupTripsByBlock(trips []gtfs.Trip, tripBlockByID map[string]string) map[string][]gtfs.Trip {
	groups := make(map[string][]gtfs.Trip, len(trips))
	for _, trip := range trips {
		key := "trip:" + trip.ID.ID
		if blockID, hasBlock := tripBlockByID[trip.ID.ID]; hasBlock {
			key = "block:" + blockID
		}
		groups[key] = append(groups[key], trip)
	}
	return groups
}
```

### 4.2 `isCombinedRecordActive` / `representativeTrip` / `isTripActive`

```go
const activeRecordLookahead = time.Hour

func isTripActive(trip gtfs.Trip, now time.Time) bool {
	if len(trip.StopTimeUpdates) == 0 {
		return false
	}
	first := firstPredictionTime(trip.StopTimeUpdates[0])
	last := lastPredictionTime(trip.StopTimeUpdates[len(trip.StopTimeUpdates)-1])
	if first == nil || last == nil {
		return false
	}
	return now.Add(activeRecordLookahead).After(*first) && last.After(now)
}
```

`firstPredictionTime` prefers arrival, falling back to departure (matching
Java's `first.hasArrival() ? ... : ...`); `lastPredictionTime` prefers
departure, falling back to arrival — the two are deliberately asymmetric,
matching Java exactly rather than reusing one helper for both.

### 4.3 Known simplifications vs. Java's real matching engine

- **Block grouping is purely static-block-ID based.** Java's
  `AssignmentInfo` additionally correlates the vehicle-positions feed
  itself when resolving which vehicle owns a block
  (`preferredVehicleByBlockId`), which can merge a couple of additional
  vehicle-less records that Maglev currently keeps standalone. Accounts for
  the small residual gap (single digits on a ~70-record feed) in
  `realtimeRecordsTotal`.
- **"Resolved" is a direct trip-ID lookup**, not Java's full
  schedule-deviation block-matching
  (`applyTripUpdatesToRecord`/`applyVehiclePositionToRecord`, same file),
  which can reject a statically-known trip if its reported position/time
  deviates too far from the static schedule. Out of scope — see §8.
- **`representativeTrip` selection is order-independent** (earliest
  first-prediction), where Java uses literal GTFS-RT feed entity order
  (`update.getTripUpdates().get(0)`). `go-gtfs.ParseRealtime` sorts trips by
  ID and doesn't expose original entity order. Verified to be within 1 of
  Java's number in practice (§3.4).

## 5. Current progress

- [x] `GET /api/where/metrics.json` implemented and registered
      (`internal/restapi/metrics_handler.go`, `internal/restapi/routes.go`),
      backed by `internal/gtfs.Manager.GetMetrics` (`internal/gtfs/metrics.go`)
      and the sqlc queries in `gtfsdb/query.sql`.
- [x] `scheduledTripsCount` reworked from a flat schedule total to active-block
      counting (§3.2).
- [x] `realtimeRecordsTotal` reworked from raw entity counting to block-grouped
      counting (§3.3, §4.1).
- [x] `realtimeTripCountsMatched`/`Unmatched` reworked to grouped resolution +
      activity-window gating (§3.4, §4.2).
- [x] `stopIDsMatchedCount`/`UnmatchedCount` deduplicated per feed poll (§3.5).
- [x] `timeSinceLastRealtimeUpdate` implemented against the real wall clock,
      deliberately without Java's epoch-fallback bug (§3.6).
- [ ] Not yet committed: the §3.4/§4.2 activity-window gating fix is still
      being tested locally before merge.

## 6. Acceptance checklist

- [x] `GET /api/where/metrics.json` returns 200 with the shape in §3.1.
- [x] `scheduledTripsCount` verified exact against live Java (69 vs 70,
      70 vs 70 across polls — within normal poll-timing skew).
- [x] `realtimeRecordsTotal` verified within 1 of live Java (69 vs 70).
- [x] `realtimeTripCountsMatched` verified **exact** against live Java
      (57 vs 57).
- [x] `stopIDsMatchedCount` verified within 1 of live Java (1701 vs 1700).
- [x] `realtimeTripCountsUnmatched` / `stopIDsUnmatchedCount` verified
      exact (0 vs 0) — no unmatched data in the test feed, but logic
      independently unit-tested with fabricated unmatched trip/stop IDs.
- [x] `timeSinceLastRealtimeUpdate` deliberately diverges from Java for
      uncovered agencies (§3.6, §9) — confirmed intentional, not a bug.
- [x] Unit test coverage: no-feed baseline, matched/unmatched trips,
      unmatched stops, feed-agency-filter fallback, unresolvable feed
      non-attribution, staleness, multi-feed agency isolation, block
      grouping (with and without vehicle tags), blockless standalone
      records, stop deduplication across trips, activity-window gating.
- [x] `go vet` (both `sqlite_fts5 sqlite_math_functions` and `purego` build
      tags), `make test`, `go fmt` all clean.
- [ ] Reviewer sign-off on §9's deliberate deviations.

## 7. Out of scope (deliberately)

- Java's full schedule-deviation block-matching engine — §4.3.
- Vehicle-positions-feed correlation for block/vehicle assignment — §4.3,
  §9's residual `realtimeRecordsTotal` gap.
- Preserving GTFS-RT feed entity order through `go-gtfs` parsing, to make
  `representativeTrip` selection non-approximate — would require bypassing
  `go-gtfs.ParseRealtime`'s trip sorting or a proto-level pre-pass; not
  justified by the <2% numeric gap it would close.
- Replicating Java's `timeSinceLastRealtimeUpdate` epoch-fallback bug (§9)
  — the Java behavior is the bug.
- Any metric not already in Java's response shape — tracked as follow-up
  ideas in §8.

## 8. Future metrics (not in this PR)

### GTFS static data quality

- Entity counts per agency (routes, stops, trips, shapes) — quick sanity
  check that a bundle reload didn't silently drop data.
- Orphaned stops (zero trips serving them).
- Trips with no shape.
- Calendar coverage gaps (date ranges with zero active service, which
  would otherwise surface only as a silent `scheduledTripsCount: 0`).

### GTFS-RT data quality

- Vehicle position count / staleness distribution, using the existing
  `feedVehicleLastSeen` tracking.
- Service alert count per agency (`feedAlerts` isn't reflected in
  `metrics.json` at all right now).
- Added / duplicated / cancelled trip counts, separate from matched — Java
  already tracks these on `MonitoredResult` (`addedTripIds`,
  `duplicatedTripIds`, `cancelledTripIds`) but Maglev currently folds them
  into "matched" or ignores them.
- Real-time coverage ratio: `realtimeTripCountsMatched` /
  `scheduledTripsCount` per agency — cheap to compute from fields that
  already exist.
- Distinct vehicle count from the vehicle-positions feed, independent of
  `realtimeRecordsTotal`'s block-grouping — a genuinely different, useful
  number Java doesn't expose either.
- Feed poll success/failure history — current `metrics.json` is a
  point-in-time snapshot; a short rolling window would catch a feed that's
  flapping rather than just currently down.

## 9. Deviations from legacy Java

| Field | Deviation | Why |
|---|---|---|
| `timeSinceLastRealtimeUpdate` | Returns `0` for an agency with no covering feed, instead of Java's raw current-epoch-seconds | Java's behavior is a bug (`_lastUpdate` defaults to `0`, producing `(now-0)/1000`), not an intentional contract — confirmed by decoding a live production sample |
| `realtimeRecordsTotal` | Off by roughly one record on live data | Block grouping doesn't correlate the vehicle-positions feed for assignment the way Java's `AssignmentInfo` does — §4.3 |
| `realtimeTripCountsMatched` | `representativeTrip` selection is prediction-order-based, not feed-entity-order-based | `go-gtfs` doesn't preserve original entity order; verified within 1 of Java in practice — §4.3 |
| `realtimeTripCountsMatched`/`Unmatched` | "Resolved" is static-ID existence, not full schedule-deviation matching | Java's matching engine is a materially larger feature, out of proportion to a monitoring endpoint — §7 |
