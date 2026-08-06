# Stop Direction Calculation

## Summary

`direction` is an optional field on stop objects across the OBA API — a compass value
(`"N"`, `"SW"`, etc.) indicating which way a vehicle is travelling when it serves the
stop. GTFS's `stop_direction` extension supplies this directly for feeds that publish
it, but most feeds don't. When it's absent, both legacy Java and Maglev fall back to
**inferring** a direction from the geometry of the shapes that pass through the stop,
using the same statistical algorithm. This page documents that shared algorithm once;
individual endpoint pages link back here instead of each re-describing it.

## Legacy algorithm (Java)

Java computes stop direction once, **offline, during GTFS bundle build** — not
per-request — and stores the result in the narrative provider for later lookup.
`GenerateNarrativesTask.computeStopDirection`
([lines 407–473](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation-builder/src/main/java/org/onebusaway/transit_data_federation/bundle/tasks/GenerateNarrativesTask.java#L407-L473)):

1. **GTFS extension first.** If the feed's `stop_direction` value translates to a
   compass direction, use it directly and stop.
2. **Gather shape orientations.** Otherwise, collect the bearing of every shape
   segment the stop sits on, via
   [`DistanceTraveledShapePointIndex`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation/src/main/java/org/onebusaway/transit_data_federation/impl/shapes/DistanceTraveledShapePointIndex.java),
   which locates the shape segment bracketing the stop using `shape_dist_traveled`
   (falling back to geographic distance matching when a feed omits
   `shape_dist_traveled` — confirmed as Java-matching behavior by the introducing
   Maglev commit message, though not independently re-traced to a Java line citation
   here).
3. **Vector-average the orientations.** Convert each bearing to a unit vector
   (`cos`/`sin`) and average the x and y components separately, then take
   `atan2(yMean, xMean)` as the circular mean bearing.
4. **Reject high-variance stops.** Compute the sample standard deviation of the x and
   y components. If either exceeds `0.7`
   ([`_stopDirectionStandardDeviationThreshold`](https://github.com/OneBusAway/onebusaway-application-modules/blob/095bf1ac3aeb7009b9f78e7f1cb4c65bd38b988a/onebusaway-transit-data-federation-builder/src/main/java/org/onebusaway/transit_data_federation/bundle/tasks/GenerateNarrativesTask.java#L95)),
   the stop sits at too ambiguous/multi-directional a junction to assign a direction
   confidently, and the field is left absent rather than guessed.
5. **Median, not mean, for the final angle.** Normalize each orientation relative to
   the circular mean, sort them, and take the median as the final bearing — this
   resists outlier shape points that the mean alone would be skewed by.
6. Convert the final angle to one of the eight compass directions (N/NE/E/SE/S/SW/W/NW).

## Maglev implementation

`internal/gtfs/advanced_direction_calculator.go`'s `AdvancedDirectionCalculator` is a
faithful port of the algorithm above — same two-step GTFS-then-shapes order, same
vector-averaging circular mean, same `0.7` standard-deviation rejection threshold (the
literal same constant, not a coincidence), same median-of-normalized-angles final step.
The introducing commit (`d03c9e0`, *"feat: implement OneBusAway stop direction
calculation algorithm"*) states this directly and cites `GenerateNarrativesTask.java`
as its source.

Maglev also mirrors Java's "precompute once, not per-request" design: `DirectionPrecomputer`
batch-processes all stops at startup after GTFS data loads, so `CalculateStopDirection`
is an O(1) cache lookup at request time for the common case, with the same shape-based
computation available lazily for cache misses.

Every endpoint that returns a stop object (in `data.entry`, `data.list[]`, or
`data.references.stops[]`) populates `direction` through this same shared
`AdvancedDirectionCalculator`, via the common `buildStopModel` helper in
`internal/restapi/reference_utils.go` or a direct call. As of this writing that's:
[[stop]], [[stops-for-agency]], [[search-stop]], [[stops-for-location]],
[[stops-for-route]], [[arrival-and-departure-for-stop]],
[[arrivals-and-departures-for-stop]], and [[trips-for-location]].

## Implementation Decisions

**Rejects near-zero mean vectors using an epsilon comparison, rather than Java's exact `== 0.0` check (deviates from legacy, in effect only).**

Java's undefined-case guard for directly-opposing shape orientations is a literal
equality check:

```java
if (yMu == 0.0 && xMu == 0.0)
  return null;
```

In practice, a floating-point mean of `cos`/`sin` values is vanishingly unlikely to
land on *exactly* `0.0`, so this guard rarely fires even when the true orientation is
genuinely undefined (e.g. a stop sitting exactly between two opposite-facing shape
segments) — the mean will be some tiny non-zero value that survives the check and
produces an arbitrary, low-confidence direction instead of the intended "leave it
absent" outcome.

Maglev's port
([`advanced_direction_calculator.go:252-256`](https://github.com/OneBusAway/maglev/blob/main/internal/gtfs/advanced_direction_calculator.go#L252-L256))
uses an epsilon comparison instead:

```go
if math.Abs(xMu) < 1e-6 && math.Abs(yMu) < 1e-6 {
    return "", nil
}
```

This makes the guard actually catch the near-zero cases it was written for, at the cost
of no longer being a literal translation of Java's condition. The practical effect is
that a small number of stops at genuinely ambiguous junctions, which legacy Java would
assign an arbitrary/low-confidence direction to due to the guard's ineffectiveness, get
`direction: ""` from Maglev instead.

If this proves too aggressive in practice (rejecting stops a human would consider to
have an obvious direction), the fallback is to loosen or remove the epsilon check and
revert to matching Java's literal (largely inert) condition.
