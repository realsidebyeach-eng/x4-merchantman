# Buy-Fill Min Cargo % Stop Condition Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `mincargopercent` parameter so the buy-side fill-cargo pipeline stops chasing additional wares/stops once the ship's free cargo space (relative to its total capacity for the wanted ware's cargo class) drops below that percent, instead of continuing to search for another partner all the way down to a literally full hold.

**Architecture:** `aiscripts/sbe_stationtrader.xml`'s buy-fill decision loop (inside `$categorybuyers`, servicing both a station's production buy offers and its Build Storage) currently continues to the next wanted ware — each requiring a fresh seller search — as long as `$remainingvolume gt 0`. This plan computes a `$mincargovolume` threshold once per category pass from a new `$mincargopercent` param and the ware-bracketed `.max` capacity attribute, then swaps the loop's `$remainingvolume gt 0` guard for `$remainingvolume ge $mincargovolume`. Because that guard fires on every iteration including the first, a pass that starts already below the threshold skips buy-fill entirely for that pass. Sell-fill and classic mode are untouched.

**Tech Stack:** X4: Foundations `aiscript` XML. Verification is `xmllint --noout` (this repo's standard gate) plus schema validation against the local `aiscripts.xsd` cache (`/home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/aiscripts.xsd`). The `.cargo.{$ware}.max` attribute used here was confirmed via public GitHub code search against real X4 aiscripts (including unpacked vanilla scripts using `.cargo.capacity.all`, and a mod using `.cargo.{$ware}.max` in the exact bracketed form this file already relies on for `.free`/`.count`) before this plan was written — no fallback attribute is needed, but in-game debug-log verification is still the final gate per this project's standard practice, since this domain has no automated test harness for aiscript *behavior* (only structural/schema validity).

## Global Constraints

- Every task ends with `xmllint --noout aiscripts/sbe_stationtrader.xml` passing (and `xmllint --noout t/0001.xml` where that file is touched).
- Bump the aiscript `version` attribute from 19 to 20, with a matching `<patch sinceversion="20">` upgrade block defaulting `$mincargopercent` to `10` when null — this project bumps version on every functional change and provides an upgrade patch for existing saves.
- Do not bump `content.xml`'s `version` — it has stayed at `1` across this project's entire history.
- Applies to the **buy-side fill-cargo pipeline only**. Do not touch the sell-fill pipeline (`$sellremainingvolume`, lines ~294-415) or classic mode (either direction).
- `mincargopercent` gates *whether another ware/stop gets considered*, not how much of an already-accepted ware gets bought — do not add logic that caps or splits a single ware's amount to avoid dipping below the threshold.
- No fallback capacity attribute — use `this.ship.cargo.{$seedware}.max` as designed; do not add a `this.ship.cargo.capacity` fallback path (rejected in the design in favor of the confirmed bracketed form).

---

## Task 1: Add the `mincargopercent` parameter, its localization text, and the version-upgrade patch

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml:4` (version attribute)
- Modify: `aiscripts/sbe_stationtrader.xml:24` (insert new param after `fillcargo`)
- Modify: `aiscripts/sbe_stationtrader.xml:110-117` (insert new patch block after the `sinceversion="12"` block)
- Modify: `t/0001.xml` (insert new text id `117`)

**Interfaces:**
- Produces: order param `$mincargopercent` (number, 0-95, default 10), consumed by Task 2's loop-condition change.

- [ ] **Step 1: Bump the aiscript version**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="19">
```

Replace with:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="20">
```

- [ ] **Step 2: Add the new param after `fillcargo`**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
			<!-- true: buy from every wanted ware until the hold is full (or nothing more is available/affordable), then deliver everything home in one go. false: classic immediate buy-then-deliver, one ware at a time. -->
			<param name="fillcargo" default="true" type="bool" text="{8834271,112}" comment="Fill the cargo hold across all wanted wares before returning home to deliver."/>
			<!-- true: also resupply a home station's Build Storage (construction-phase demand for a station under construction or having a module added) alongside its normal production buy offers. -->
			<param name="manageconstruction" default="true" type="bool" text="{8834271,113}" comment="Also resupply Build Storage (construction wares) for stations under construction/expansion."/>
```

Replace with:

```xml
			<!-- true: buy from every wanted ware until the hold is full (or nothing more is available/affordable), then deliver everything home in one go. false: classic immediate buy-then-deliver, one ware at a time. -->
			<param name="fillcargo" default="true" type="bool" text="{8834271,112}" comment="Fill the cargo hold across all wanted wares before returning home to deliver."/>
			<!-- Buy-side fill-cargo only: once free cargo space (relative to total capacity for the wanted ware's cargo class) drops below this percent, stop searching for another wanted ware to fill the rest of the hold - deliver/return with what's already queued instead of chasing one more stop. Ignored when fillcargo is off. -->
			<param name="mincargopercent" default="10" type="number" text="{8834271,117}" comment="Stop chasing additional stops once free cargo space drops below this percent of total capacity (buy-side fill-cargo only; ignored when Fill Cargo is off).">
				<input_param name="startvalue" value="10"/>
				<input_param name="min" value="0"/>
				<input_param name="max" value="95"/>
				<input_param name="step" value="1"/>
			</param>
			<!-- true: also resupply a home station's Build Storage (construction-phase demand for a station under construction or having a module added) alongside its normal production buy offers. -->
			<param name="manageconstruction" default="true" type="bool" text="{8834271,113}" comment="Also resupply Build Storage (construction wares) for stations under construction/expansion."/>
```

- [ ] **Step 3: Add the version-20 upgrade patch**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
		<patch sinceversion="12">
			<do_if value="$enablebuying == null">
				<set_value name="$enablebuying" exact="true"/>
			</do_if>
			<do_if value="$enableselling == null">
				<set_value name="$enableselling" exact="true"/>
			</do_if>
		</patch>
		<attention min="unknown">
```

Replace with:

```xml
		<patch sinceversion="12">
			<do_if value="$enablebuying == null">
				<set_value name="$enablebuying" exact="true"/>
			</do_if>
			<do_if value="$enableselling == null">
				<set_value name="$enableselling" exact="true"/>
			</do_if>
		</patch>
		<patch sinceversion="20">
			<do_if value="$mincargopercent == null">
				<set_value name="$mincargopercent" exact="10"/>
			</do_if>
		</patch>
		<attention min="unknown">
```

- [ ] **Step 4: Add the localization text**

In `t/0001.xml`, find:

```xml
		<t id="112">Fill Cargo Before Returning (buy every wanted ware first, deliver all at once; off = classic immediate buy-then-deliver per ware)</t>
		<t id="113">Also Resupply Build Storage (buy construction wares for stations under construction or being expanded)</t>
```

Replace with:

```xml
		<t id="112">Fill Cargo Before Returning (buy every wanted ware first, deliver all at once; off = classic immediate buy-then-deliver per ware)</t>
		<t id="117">Min Free Cargo % to Chase Another Stop (buy-side Fill Cargo only) - stop looking for more wares once the hold is this full or fuller</t>
		<t id="113">Also Resupply Build Storage (buy construction wares for stations under construction or being expanded)</t>
```

- [ ] **Step 5: Validate both files**

Run:

```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && xmllint --noout t/0001.xml && echo VALID
```

Expected: `VALID`. Also run schema validation:

```bash
xmllint --noout --schema /home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/aiscripts.xsd aiscripts/sbe_stationtrader.xml
```

Expected: no errors (the last line printed is the filename followed by `validates`).

- [ ] **Step 6: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml t/0001.xml
git commit -m "feat: add Min Free Cargo % to Chase Another Stop parameter"
```

---

## Task 2: Wire the loop-condition change in the buy-fill decision phase

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml:605-614` (per current numbering before Task 1's insertions shift line numbers — search by content, not line number)

**Interfaces:**
- Consumes: `$mincargopercent` (param, produced by Task 1), `$seedware` and `$remainingvolume` (already-existing locals in this loop).
- Produces: `$cargocapacity`, `$mincargovolume` (new locals, scoped to this per-category pass, not consumed outside this task).

- [ ] **Step 1: Add the capacity/threshold computation and update the pass-start debug line**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
							<set_value name="$seedware" exact="$processoffers.{1}.ware"/>
							<set_value name="$remainingvolume" exact="this.ship.cargo.{$seedware}.free * $seedware.volume"/>
							<set_value name="$remainingbudget" exact="player.money/100"/>
							<set_value name="$startingorders" exact="this.ship.tradeorders.count"/>
							<do_if value="$debugtrace">
								<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+$categorylabel+': fill pass starting with '+$remainingvolume+'m3 free cargo, '+$remainingbudget+'cr spendable, '+$processoffers.count+' wanted ware(s).'" output="false" append="true" chance="100"/>
							</do_if>

							<do_all exact="$processoffers.count" counter="$w">
								<do_if value="(($startingorders+($decidedwares.count*2)) le 4) and ($remainingvolume gt 0) and ($remainingbudget gt 0)">
```

Replace with:

```xml
							<set_value name="$seedware" exact="$processoffers.{1}.ware"/>
							<set_value name="$remainingvolume" exact="this.ship.cargo.{$seedware}.free * $seedware.volume"/>
							<set_value name="$cargocapacity" exact="this.ship.cargo.{$seedware}.max"/>
							<set_value name="$mincargovolume" exact="$cargocapacity*$mincargopercent/100"/>
							<set_value name="$remainingbudget" exact="player.money/100"/>
							<set_value name="$startingorders" exact="this.ship.tradeorders.count"/>
							<do_if value="$debugtrace">
								<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+$categorylabel+': fill pass starting with '+$remainingvolume+'m3 free cargo (min '+$mincargovolume+'m3 of '+$cargocapacity+'m3 total to keep chasing stops), '+$remainingbudget+'cr spendable, '+$processoffers.count+' wanted ware(s).'" output="false" append="true" chance="100"/>
							</do_if>

							<do_all exact="$processoffers.count" counter="$w">
								<do_if value="(($startingorders+($decidedwares.count*2)) le 4) and ($remainingvolume ge $mincargovolume) and ($remainingbudget gt 0)">
```

Note: this exact block appears only once in the file (the sell-fill pipeline's equivalent block uses `$sellremainingvolume`/`$sellprocessoffers`/`$sw` naming, not `$remainingvolume`/`$processoffers`/`$w`) — a plain string match is sufficient, no line-number anchoring needed.

- [ ] **Step 2: Validate**

Run:

```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && xmllint --noout --schema /home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/aiscripts.xsd aiscripts/sbe_stationtrader.xml && echo VALID
```

Expected: `VALID` with no schema errors above it.

- [ ] **Step 3: Confirm the sell-fill pipeline is untouched**

Run:

```bash
grep -n "sellremainingvolume" aiscripts/sbe_stationtrader.xml
```

Expected: every match still reads `$sellremainingvolume gt 0` (or its declaration/decrement) — none replaced with `ge`, and no `$sellmincargovolume`/`$sellcargocapacity` variables introduced anywhere in the file.

- [ ] **Step 4: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml
git commit -m "feat: stop buy-fill from chasing additional stops below Min Free Cargo %"
```

---

## Task 3: Document the new parameter in CONFIGURATION.md

**Files:**
- Modify: `docs/CONFIGURATION.md`

**Interfaces:** none (documentation only).

- [ ] **Step 1: Add the parameter reference table row**

In `docs/CONFIGURATION.md`, find:

```markdown
| **Fill Cargo Before Returning** | on/off | on | On = buy and/or sell (per whichever roles are enabled) from every wanted/sellable ware first, tracking remaining cargo space (and credits, for buying) as it goes, then deliver/export everything home or to buyers in one trip once the hold is full or nothing more is available/affordable. Off = classic mode, buy or sell and immediately deliver/export one ware at a time. See below. |
| **Also Resupply Build Storage** | on/off | on | On = also buy construction wares a home station's Build Storage currently wants (station under construction or having a module added), same as normal production wares. Off = Build Storage is ignored entirely. See [Build Storage](#build-storage-construction-wares) below. |
```

Replace with:

```markdown
| **Fill Cargo Before Returning** | on/off | on | On = buy and/or sell (per whichever roles are enabled) from every wanted/sellable ware first, tracking remaining cargo space (and credits, for buying) as it goes, then deliver/export everything home or to buyers in one trip once the hold is full or nothing more is available/affordable. Off = classic mode, buy or sell and immediately deliver/export one ware at a time. See below. |
| **Min Free Cargo % to Chase Another Stop** | 0–95 | 10 | Buy-side Fill Cargo Before Returning only. Once free cargo space drops below this percent of the ship's total capacity, buy-fill stops searching for another wanted ware/seller and delivers with whatever's already queued instead of chasing one more stop. Ignored when Fill Cargo Before Returning is off, and has no equivalent on the selling side. See below. |
| **Also Resupply Build Storage** | on/off | on | On = also buy construction wares a home station's Build Storage currently wants (station under construction or having a module added), same as normal production wares. Off = Build Storage is ignored entirely. See [Build Storage](#build-storage-construction-wares) below. |
```

- [ ] **Step 2: Add the explanatory paragraph in the Fill Cargo section**

In `docs/CONFIGURATION.md`, find:

```markdown
hold physically fills up. This means fewer separate round trips overall —
the ship gathers or exports several wares in one outing instead of
shuttling back and forth for each ware individually.

**Classic mode** (Fill Cargo Before Returning = off): the original
```

Replace with:

```markdown
hold physically fills up. This means fewer separate round trips overall —
the ship gathers or exports several wares in one outing instead of
shuttling back and forth for each ware individually.

On the buying side only, **Min Free Cargo % to Chase Another Stop** adds
an early-exit condition: once free cargo space drops below this percent
of the ship's total capacity, buy-fill stops considering further wanted
wares for the rest of that pass, even if more demand and cargo space
technically remain. This avoids sending the ship on a whole extra
detour just to top off the last sliver of a nearly-full hold. It has no
effect in classic mode (which only ever considers one ware per pass
anyway) and no equivalent on the selling side.

**Classic mode** (Fill Cargo Before Returning = off): the original
```

- [ ] **Step 3: Commit**

```bash
git add docs/CONFIGURATION.md
git commit -m "docs: document Min Free Cargo % to Chase Another Stop"
```

---

## Task 4: In-game verification (manual — required before considering this feature done)

This mod has no automated test harness for aiscript *behavior* — Tasks 1-2
already covered structural/schema validity. The capacity attribute
(`this.ship.cargo.{$ware}.max`) was corroborated by public GitHub code
search across independent X4 aiscripts (including vanilla) before this
plan was written, but has no prior confirmed usage inside *this specific
file*, so this task is the first real-engine confirmation that it
resolves to a sane number here, not just a syntax-risk mitigation.

**Files:** none (manual QA against the installed extension).

- [ ] **Step 1: Deploy the updated mod**

```bash
rsync -a --delete --exclude '.git' --exclude '.claude' \
  ~/Projects/X4StationTrader/ \
  "$HOME/.local/share/Steam/steamapps/common/X4 Foundations/extensions/sbe_station_trader/"
```

- [ ] **Step 2: Set up a ship to observe a buy-fill pass with debug logging on**

1. Assign a ship to a home station with several wanted wares (or, if
   testing against Build Storage, a station under construction/expansion)
   using **Enable Buying** on, **Fill Cargo Before Returning** on, and
   **Enable Debug Log** on.
2. Leave **Min Free Cargo % to Chase Another Stop** at its default (10) for
   the first check.

- [ ] **Step 3: Confirm the pass-start debug line reports sane numbers**

Find the line starting with `<station>: fill pass starting with` in
`Documents/Egosoft/X4/.../StationTrader/<ship idcode>.txt`.

Expected: it now reads something like `... fill pass starting with 1200m3
free cargo (min 300m3 of 3000m3 total to keep chasing stops), 45000cr
spendable, 4 wanted ware(s).` — `$cargocapacity` (the `...m3 total` figure)
should roughly match the ship's known cargo hold size (check the ship's
info panel in-game), and `$mincargovolume` (the `min ...m3` figure) should
be `$cargocapacity * 0.10`. If `$cargocapacity` instead prints as `0`,
`null`, or a value wildly inconsistent with the ship's actual hold size,
the `.max` attribute assumption was wrong for this cargo-class context —
stop and report back rather than proceeding to Step 4.

- [ ] **Step 4: Confirm the stop condition actually triggers**

1. Lower **Min Free Cargo % to Chase Another Stop** to a high value (e.g.
   80) on a ship whose home station has several wanted wares spread across
   sellers that would normally each get visited.
2. Let it run a buy-fill pass.

Expected: the debug log's `fill pass done deciding, committing N ware
type(s)` line shows fewer wares decided than it would with a lower/default
threshold (compare against a prior pass, or against the same station with
the parameter reset to 10) — because the loop now exits once simulated
free space would drop under 80% of capacity, rather than continuing until
the hold is nearly full. Confirm no error/exception appears in the log
around the new `set_value` calls.

- [ ] **Step 5: Confirm a pass starting already below threshold skips buy-fill**

1. With **Min Free Cargo % to Chase Another Stop** still at a high value
   (e.g. 80), let the ship carry partial stray cargo into a new pass (or
   simply observe a pass where the hold is already more than 20% full at
   pass start).

Expected: the debug log shows the pass-start line printing
`$remainingvolume` already below `$mincargovolume`, and no `fill pass done
deciding` line follows for that category that pass — buy-fill was skipped
entirely, per the agreed "gate from the first ware onward" design.
