# Buy-Fill: Stop Chasing Additional Stops Below a Minimum Free Cargo % — Design

## Problem

With Fill Cargo Before Returning enabled, the buy-fill pipeline in
`aiscripts/sbe_stationtrader.xml` (decision loop at lines 613-709) keeps
evaluating wanted wares — each requiring a fresh seller search, i.e. a new
stop — as long as any cargo space remains at all
(`$remainingvolume gt 0`). A ship that's already nearly full will still go
out of its way for one more small stop to top off the last sliver of
space, which isn't worth the detour.

The request: once the ship's free cargo space drops below a threshold
percentage of its total capacity, stop chasing additional stops for the
rest of that pass.

## Design

### New parameter

Add `mincargopercent` to the order's params, next to `fillcargo`:

```xml
<param name="mincargopercent" default="10" type="number" text="{8834271,117}" comment="Stop chasing additional stops once free cargo space drops below this percent of total capacity (buy-side fill-cargo only; ignored when Fill Cargo is off).">
	<input_param name="startvalue" value="10"/>
	<input_param name="min" value="0"/>
	<input_param name="max" value="95"/>
	<input_param name="step" value="1"/>
</param>
```

Range and step mirror the existing `minpercent` param. A `<patch
sinceversion="20">` block sets it to `10` when null, following this file's
existing upgrade-path convention (see `sinceversion="8"` for `fillcargo`
itself).

Localization: add `t id="117"` to `t/0001.xml`:
`Min Free Cargo % to Chase Another Stop (buy-side Fill Cargo only) - stop looking for more wares once the hold is this full or fuller`

### Mechanism

Confirmed via public GitHub code search against real X4 aiscripts —
including the unpacked vanilla scripts themselves
(`Insigar/X4-Modding:Unpacked/Vanilla/aiscripts/order.trade.single.xml`,
which uses `this.ship.cargo.capacity.all`) and a mod using the same
ware-bracketed form this file already relies on
(`nf_a/aiscripts/move.plunder.taxi.xml`:
`this.ship.cargo.{ware.fuelcells}.max`) — the ware-bracketed capacity
attribute is `.max`, the direct counterpart to the `.free` and `.count`
suffixes this file already uses on `this.ship.cargo.{$ware}`. No fallback
attribute is needed.

In the buy-fill decision phase, immediately after `$remainingvolume` is
seeded (currently line 606, inside each category's — production or Build
Storage — pass):

```xml
<set_value name="$remainingvolume" exact="this.ship.cargo.{$seedware}.free * $seedware.volume"/>
<set_value name="$cargocapacity" exact="this.ship.cargo.{$seedware}.max"/>
<set_value name="$mincargovolume" exact="$cargocapacity*$mincargopercent/100"/>
```

The decision loop's guard condition (currently line 614):

```xml
<do_if value="(($startingorders+($decidedwares.count*2)) le 4) and ($remainingvolume gt 0) and ($remainingbudget gt 0)">
```

becomes:

```xml
<do_if value="(($startingorders+($decidedwares.count*2)) le 4) and ($remainingvolume ge $mincargovolume) and ($remainingbudget gt 0)">
```

This is the only functional change to the loop. Because the same
condition guards every iteration — including the first — a pass that
starts already at or above the threshold-full mark (e.g. leftover cargo
from an interrupted prior run) skips buy-fill entirely for that pass
rather than chasing even one more stop, per the agreed design. `balanced`
mode, the order-budget guard, and the price/amount computation inside the
loop are all unaffected — this only changes when the loop stops iterating,
not what it does per ware.

The existing pass-start debug line (currently line 610) is extended to
also report `$cargocapacity` and `$mincargovolume`, matching how it
already reports `$remainingvolume` and `$remainingbudget`. No new
"skipped due to threshold" debug line is added — consistent with this
loop's existing convention of not logging the order-budget/volume/budget
exhaustion cases individually (unlike classic mode, which does log its
skip case).

### Scope

- **Buy-side fill-cargo only**, per the agreed design — the sell-fill
  pipeline (lines 294-415, `$sellremainingvolume`) is untouched.
- Applies uniformly to both categories the buy-fill loop already services
  per pass (a station's own production buy offers, and its Build Storage
  construction demand, per `buildstoragefirst`) — both derive
  `$cargocapacity`/`$mincargovolume` from the same per-category
  `$remainingvolume` seed, so no special-casing is needed between them.
- `mincargopercent` is only meaningful when `fillcargo` is true; no
  runtime check is added to hide or disable the param when `fillcargo` is
  off (matching how `buildstoragefirst` and `manageconstruction` are
  already always-visible params that are simply inert unless their
  companion setting applies).

### Out of scope

- Classic mode (both directions) — already processes one ware per pass by
  design, so "chasing additional stops" doesn't apply to it.
- Capping the *amount* bought of the ware currently being evaluated so it
  never dips the hold below the threshold — this design only gates
  *whether another ware/stop gets considered*, not how much of an
  already-accepted ware gets bought. A single large ware could still take
  the hold from above-threshold to well below it in one stop; that stop
  itself is not reduced or split by this feature.
- Sell-fill pipeline.

## Versioning

Bump the aiscript `version` attribute from 19 to 20, with a
`<patch sinceversion="20">` block defaulting `$mincargopercent` to `10`
when null.
