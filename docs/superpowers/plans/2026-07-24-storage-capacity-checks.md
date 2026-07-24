# Storage Capacity Checks Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop trader ships from buying/holding more of a ware than their home station can currently physically store, and stop the stray-cargo-to-home fallback from permanently stranding cargo when the home's live buy capacity is 0.

**Architecture:** `aiscripts/sbe_stationtrader.xml` has three places that decide a buy/deliver amount using only `$wantedoffer.amount` (a demand-target figure, not real storage capacity) as the cap: the fill-cargo buy pipeline's per-ware buy decision, its batched delivery loop, and the classic one-ware-at-a-time buy pipeline. Insert `clamp_trade_amount` (an X4 script action never previously used in this file) at each point, clamping against `$homestation`'s actual storage. Separately, the stray-cargo-to-home fallback marks itself "handled" even when the resulting order amount is 0, which prevents the (correct, floor-free) external-buyer fallback from ever running on genuinely stuck cargo — fix the guard to require amount > 0, and fix its debug log to report the real clamped amount instead of the ship's full held quantity.

**Tech Stack:** X4: Foundations `aiscript` XML. Verification is `xmllint --noout` plus schema validation against the local `aiscripts.xsd` cache (`/home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/aiscripts.xsd`) — both already confirmed to pass for `clamp_trade_amount` usage before this plan was written (the element and its `trade`/`amount`/`buyer`/`result` attributes are defined in `common.xsd`, confirmed via direct schema inspection).

## Global Constraints

- Every task ends with `xmllint --noout aiscripts/sbe_stationtrader.xml` passing.
- Also run `xmllint --noout --schema /home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/aiscripts.xsd aiscripts/sbe_stationtrader.xml` for Task 1 (the task introducing `clamp_trade_amount`, a never-before-used element in this file) to confirm the new element and its attributes validate against the real X4 schema, not just well-formedness.
- Bump the aiscript `version` attribute (currently 15, so 16) — this project bumps version on every functional change.
- Do not touch the Selling pipeline, the stray-cargo-to-external-buyer fallback, the price-floor logic, or the wallet-budget fix (version 15) — all are out of scope and already correct for what they do.
- No new param, no player-facing toggle.
- No change to the Logbook `write_to_logbook` money/amount text in the fill-cargo delivery loop — those describe the already-executed purchase, not the delivery, and must keep using `$deliveramounts.{$d}` (the original bought amount), not any clamped delivery amount.
- This is X4: Foundations aiscript XML — no unit-test framework exists; xmllint (plus schema validation for Task 1) is the verification gate.

---

## Task 1: Add clamp_trade_amount checks to both buy pipelines

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml`

**Interfaces:**
- Produces: `$amount` (fill-cargo and classic pipelines) and a new
  `$deliverclamped` variable (fill-cargo delivery loop) — both scoped
  locally to their existing blocks, not consumed by Task 2.

- [ ] **Step 1: Bump the aiscript version**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="15">
```

Replace with:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="16">
```

- [ ] **Step 2: Clamp the fill-cargo pipeline's buy amount against home storage**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
										<set_value name="$amount" exact="[$wareunitcap,$bestseller.amount,$wantedoffer.amount,$remainingbudget/$bestprice].min"/>
										<do_if value="$debugtrace">
											<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$ware+': picked '+$sellername+' at '+$bestprice+'cr/unit, amount='+$amount+' (wareunitcap='+$wareunitcap+', sellerhas='+$bestseller.amount+', homewants='+$wantedoffer.amount+', afford='+($remainingbudget/$bestprice)+')'" output="false" append="true" chance="100"/>
										</do_if>

										<do_if value="$amount gt 0">
											<!-- Buy leg now; the matching sell/deliver leg is queued after the whole hold is filled. -->
											<create_trade_order name="$bestseller" object="this.object" tradeoffer="$bestseller" amount="$amount" immediate="false"/>
											<append_to_list name="$deliverwares" exact="$wantedoffer"/>
											<append_to_list name="$deliveramounts" exact="$amount"/>
```

Replace with:

```xml
										<set_value name="$amount" exact="[$wareunitcap,$bestseller.amount,$wantedoffer.amount,$remainingbudget/$bestprice].min"/>
										<!-- $wantedoffer.amount is a demand target, not live storage room - clamp against the home's actual current capacity so the ship never buys more than it can deliver. -->
										<clamp_trade_amount trade="$wantedoffer" amount="$amount" buyer="$homestation" result="$amount"/>
										<do_if value="$debugtrace">
											<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$ware+': picked '+$sellername+' at '+$bestprice+'cr/unit, amount='+$amount+' (wareunitcap='+$wareunitcap+', sellerhas='+$bestseller.amount+', homewants='+$wantedoffer.amount+', afford='+($remainingbudget/$bestprice)+')'" output="false" append="true" chance="100"/>
										</do_if>

										<do_if value="$amount gt 0">
											<!-- Buy leg now; the matching sell/deliver leg is queued after the whole hold is filled. -->
											<create_trade_order name="$bestseller" object="this.object" tradeoffer="$bestseller" amount="$amount" immediate="false"/>
											<append_to_list name="$deliverwares" exact="$wantedoffer"/>
											<append_to_list name="$deliveramounts" exact="$amount"/>
```

- [ ] **Step 3: Clamp each batched delivery order right before it's created**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
							<do_all exact="$deliverwares.count" counter="$d">
								<create_trade_order name="$deliverwares.{$d}" object="this.object" tradeoffer="$deliverwares.{$d}" amount="$deliveramounts.{$d}" price="0" immediate="false"/>
								<do_if value="$enablelogbook">
```

Replace with:

```xml
							<do_all exact="$deliverwares.count" counter="$d">
								<!-- Re-check live storage room right before delivering: an earlier ware in this same batch may have already used up shared capacity. -->
								<clamp_trade_amount trade="$deliverwares.{$d}" amount="$deliveramounts.{$d}" buyer="$homestation" result="$deliverclamped"/>
								<create_trade_order name="$deliverwares.{$d}" object="this.object" tradeoffer="$deliverwares.{$d}" amount="$deliverclamped" price="0" immediate="false"/>
								<do_if value="$enablelogbook">
```

- [ ] **Step 4: Clamp the classic pipeline's buy amount against home storage**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
										<set_value name="$amount" exact="[$cargocap,$bestseller.amount,$wantedoffer.amount,$spendablemoney/(2*$bestprice)].min"/>
										<do_if value="$debugtrace">
											<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$ware+': amount='+$amount+' (cargocap='+$cargocap+', sellerhas='+$bestseller.amount+', homewants='+$wantedoffer.amount+', afford='+($spendablemoney/(2*$bestprice))+')'" output="false" append="true" chance="100"/>
										</do_if>

										<do_if value="$amount gt 0">
											<!-- Buy leg: purchase from the cheapest accessible seller. -->
											<create_trade_order name="$bestseller" object="this.object" tradeoffer="$bestseller" amount="$amount" immediate="false"/>
											<!-- Sell leg: deliver straight into the home station's own buy offer. -->
											<create_trade_order name="$wantedoffer" object="this.object" tradeoffer="$wantedoffer" amount="$amount" price="0" immediate="false"/>
```

Replace with:

```xml
										<set_value name="$amount" exact="[$cargocap,$bestseller.amount,$wantedoffer.amount,$spendablemoney/(2*$bestprice)].min"/>
										<!-- $wantedoffer.amount is a demand target, not live storage room - clamp against the home's actual current capacity so the ship never buys more than it can deliver. -->
										<clamp_trade_amount trade="$wantedoffer" amount="$amount" buyer="$homestation" result="$amount"/>
										<do_if value="$debugtrace">
											<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$ware+': amount='+$amount+' (cargocap='+$cargocap+', sellerhas='+$bestseller.amount+', homewants='+$wantedoffer.amount+', afford='+($spendablemoney/(2*$bestprice))+')'" output="false" append="true" chance="100"/>
										</do_if>

										<do_if value="$amount gt 0">
											<!-- Buy leg: purchase from the cheapest accessible seller. -->
											<create_trade_order name="$bestseller" object="this.object" tradeoffer="$bestseller" amount="$amount" immediate="false"/>
											<!-- Sell leg: deliver straight into the home station's own buy offer. -->
											<create_trade_order name="$wantedoffer" object="this.object" tradeoffer="$wantedoffer" amount="$amount" price="0" immediate="false"/>
```

- [ ] **Step 5: Validate XML well-formedness**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 6: Validate against the real X4 aiscript schema**

Run:
```bash
xmllint --noout --schema /home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/aiscripts.xsd aiscripts/sbe_stationtrader.xml && echo SCHEMA_VALID
```
Expected: `SCHEMA_VALID` printed, no errors. This is the first use of
`clamp_trade_amount` in this file — schema validation (not just
well-formedness) confirms the element and its `trade`/`amount`/`buyer`/
`result` attributes are used correctly per X4's actual aiscript engine.

- [ ] **Step 7: Verify placement — exactly 3 clamp_trade_amount calls, none touching Logbook amounts**

```bash
grep -n 'clamp_trade_amount' aiscripts/sbe_stationtrader.xml
grep -c 'deliveramounts.{\$d}\*\$deliverprices' aiscripts/sbe_stationtrader.xml
```
Expected: the first command shows exactly 3 matches (fill-cargo buy,
fill-cargo delivery, classic buy). The second command shows `2` — both
Logbook `write_to_logbook` money calculations still reference
`$deliveramounts.{$d}` (the original bought amount), unchanged by this task.

- [ ] **Step 8: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml
git commit -m "fix: clamp trader buy/deliver amounts to the home station's actual storage capacity"
```

---

## Task 2: Fix the stray-cargo-to-home fallback's phantom-handled bug

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml`

**Interfaces:**
- Produces: `$strayhomeamount` — a local variable scoped to this one
  `do_if` block, not consumed elsewhere.

- [ ] **Step 1: Fix the $strayhandled guard and debug log**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
										<find_buy_offer tradepartner="this.ship" buyer="$strayhome" wares="$strayware" result="$strayhomeoffer"/>
										<do_if value="$strayhomeoffer.available">
											<create_trade_order name="$strayhomeoffer" object="this.object" tradeoffer="$strayhomeoffer" amount="[$strayamount,$strayhomeoffer.amount].min" price="0" immediate="false"/>
											<set_value name="$strayhandled" exact="true"/>
											<do_if value="$debugtrace">
												<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'Stray cargo: delivering '+$strayamount+' '+$strayware+' to home station '+$strayhome.knownname+'.'" output="false" append="true" chance="100"/>
											</do_if>
										</do_if>
										<remove_value name="$strayhomeoffer"/>
```

Replace with:

```xml
										<find_buy_offer tradepartner="this.ship" buyer="$strayhome" wares="$strayware" result="$strayhomeoffer"/>
										<do_if value="$strayhomeoffer.available">
											<!-- $strayhomeoffer.amount can be 0 (home currently has no live storage room) even while the offer itself stays active - only count this as handled when a real amount will actually move. -->
											<set_value name="$strayhomeamount" exact="[$strayamount,$strayhomeoffer.amount].min"/>
											<do_if value="$strayhomeamount gt 0">
												<create_trade_order name="$strayhomeoffer" object="this.object" tradeoffer="$strayhomeoffer" amount="$strayhomeamount" price="0" immediate="false"/>
												<set_value name="$strayhandled" exact="true"/>
												<do_if value="$debugtrace">
													<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'Stray cargo: delivering '+$strayhomeamount+' '+$strayware+' to home station '+$strayhome.knownname+'.'" output="false" append="true" chance="100"/>
												</do_if>
											</do_if>
										</do_if>
										<remove_value name="$strayhomeoffer"/>
```

- [ ] **Step 2: Validate XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 3: Verify the fix**

```bash
grep -n 'strayhomeamount' aiscripts/sbe_stationtrader.xml
```
Expected: 3 matches — the `set_value`, the `do_if value="$strayhomeamount gt 0"`, and the debug line — all inside the stray-cargo-to-home block (within a few lines of `$strayhomeoffer.available`).

- [ ] **Step 4: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml
git commit -m "fix: stop stray-cargo home-delivery from marking itself handled on a zero-amount order"
```

---

## Task 3: In-game verification (manual — required before considering this fix done)

This mod has no automated test harness for aiscript behavior. Unlike the
prior two fixes, this change introduces an X4 script action
(`clamp_trade_amount`) never previously used in this file — schema
validation (Task 1, Step 6) confirms correct usage syntactically, but not
runtime behavior. This manual verification is more load-bearing than for
prior fixes and should not be skipped.

**Files:** none (manual QA against the installed extension).

- [ ] **Step 1: Deploy the updated mod**

```bash
rsync -a --delete --exclude '.git' --exclude '.claude' \
  ~/Projects/X4StationTrader/ \
  "$HOME/.local/share/Steam/steamapps/common/X4 Foundations/extensions/sbe_station_trader/"
```

- [ ] **Step 2: Reproduce the original scenario**

1. Find (or wait for) a ship whose home station wants to buy a ware
   (`Trade Station Mk III: N total buy offers`) while that same ware's
   warehouse room is constrained (e.g. the station also has an unsold
   surplus of the same ware it can't clear externally).
2. With **Enable Debug Log** on, let the ship run a buy pass and check the
   resulting `'... amount=... (wareunitcap=..., sellerhas=..., homewants=...,
   afford=...)'` debug line.

- [ ] **Step 3: Confirm the buy amount now respects real storage room**

Expected: the ship no longer buys (and gets stuck holding) more of a ware
than the home station can actually accept — the bought amount should track
what actually gets delivered, not the raw `homewants` target.

- [ ] **Step 4: Confirm stray cargo no longer gets permanently stuck**

If a ship ends up with stray cargo the home station currently has no room
for (buy offer `.amount` at or near 0), confirm the debug log now shows it
falling through to `'Stray cargo: no home station wanted ..., selling ...
to ... instead.'` (the external-buyer path) rather than repeating an
unchanging `'Stray cargo: delivering N ... to home station ...'` line pass
after pass with no real progress.
