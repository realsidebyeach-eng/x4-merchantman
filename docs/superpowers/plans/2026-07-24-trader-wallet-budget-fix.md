# Trader Wallet Budget Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop trader ships from silently drawing purchase funds from their home station's own account when that station has a commander-subordinate relationship with its own funded account — purchases must always draw from the player wallet only, matching this project's own documented version 6 fix.

**Architecture:** `aiscripts/sbe_stationtrader.xml` has two independent buy pipelines (fill-cargo mode and classic one-ware-at-a-time mode), each of which sets a budget variable to `player.money/100` and then unconditionally overrides it with `this.ship.commander.money/100` whenever `this.ship.commander.hasownaccount and this.ship.commander.money gt 0`. Delete both override blocks so the budget variable is always `player.money/100`.

**Tech Stack:** X4: Foundations `aiscript` XML. Verification is `xmllint --noout` — already confirmed to pass for this exact change before this plan was written.

## Global Constraints

- Every task ends with `xmllint --noout aiscripts/sbe_stationtrader.xml` passing.
- Bump the aiscript `version` attribute (currently 14, so 15) — this project bumps version on every functional change.
- Only the two `this.ship.commander` budget-override blocks are touched — do not touch the price-floor selling logic, the stray-cargo fallback, or the home-delivery `price="0"` logic, none of which are related to this bug.
- No new param, no player-facing toggle — this is an unconditional removal of dead/regressed code, not a new feature.
- This is X4: Foundations aiscript XML — no unit-test framework exists; xmllint well-formedness is the verification gate.

---

## Task 1: Remove the commander-account budget override from both buy pipelines, bump version

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml` (version bump + two block deletions)

**Interfaces:** none — this task removes code, it doesn't introduce any new variable, function, or interface other tasks depend on.

- [ ] **Step 1: Bump the aiscript version**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="14">
```

Replace with:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="15">
```

- [ ] **Step 2: Remove the override in the fill-cargo buy pipeline**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
							<set_value name="$remainingbudget" exact="player.money/100"/>
							<do_if value="this.ship.commander">
								<do_if value="this.ship.commander.hasownaccount and this.ship.commander.money gt 0">
									<set_value name="$remainingbudget" exact="this.ship.commander.money/100"/>
								</do_if>
							</do_if>
							<do_if value="$debugtrace">
```

Replace with:

```xml
							<set_value name="$remainingbudget" exact="player.money/100"/>
							<do_if value="$debugtrace">
```

- [ ] **Step 3: Remove the override in the classic buy pipeline**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
										<set_value name="$spendablemoney" exact="player.money/100"/>
										<do_if value="this.ship.commander">
											<do_if value="this.ship.commander.hasownaccount and this.ship.commander.money gt 0">
												<set_value name="$spendablemoney" exact="this.ship.commander.money/100"/>
											</do_if>
										</do_if>

										<set_value name="$cargocap" exact="this.ship.cargo.{$ware}.free"/>
```

Replace with:

```xml
										<set_value name="$spendablemoney" exact="player.money/100"/>

										<set_value name="$cargocap" exact="this.ship.cargo.{$ware}.free"/>
```

- [ ] **Step 4: Validate XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 5: Verify the fix — no `this.ship.commander` references remain**

```bash
grep -n 'this.ship.commander' aiscripts/sbe_stationtrader.xml
```
Expected: no output (zero matches) — the entire `this.ship.commander` budget-override pattern is gone from the file.

- [ ] **Step 6: Verify both budget variables still exist and are unconditional**

```bash
grep -n '\$remainingbudget" exact="player.money/100"\|\$spendablemoney" exact="player.money/100"' aiscripts/sbe_stationtrader.xml
```
Expected: exactly 2 matches — one per buy pipeline, each now the only assignment to that variable's initial value.

- [ ] **Step 7: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml
git commit -m "fix: always fund trader purchases from the player wallet, never the home station's own account"
```

---

## Task 2: In-game verification (manual — required before considering this fix done)

This mod has no automated test harness for aiscript behavior. The fix
itself carries no unverified-syntax risk (`set_value`/`do_if` deletion
follows patterns already proven throughout this file) — this task is a
functional smoke test, not a syntax-risk mitigation.

**Files:** none (manual QA against the installed extension).

- [ ] **Step 1: Deploy the updated mod**

```bash
rsync -a --delete --exclude '.git' --exclude '.claude' \
  ~/Projects/X4StationTrader/ \
  "$HOME/.local/share/Steam/steamapps/common/X4 Foundations/extensions/sbe_station_trader/"
```

- [ ] **Step 2: Reproduce the original scenario**

1. Find (or set up) a ship whose home station has its own funded account
   (`hasownaccount` true, `money gt 0`) — e.g. a station the ship is a
   subordinate of for an unrelated reason (defense/logistics group).
2. Set the player wallet to a known, distinct value from the station's
   account balance (so it's unambiguous which one funded a purchase) —
   easiest via the in-game cheat menu (`Ctrl+Shift+Alt` in vanilla control
   scheme unless remapped) or simply noting both balances before and after.
3. With **Enable Debug Log** on, let the ship run a buy pass and check the
   resulting `'... spendable ...'` debug log line.

- [ ] **Step 3: Confirm the spendable amount always matches the player wallet**

Expected: the logged `cr spendable` amount matches `player.money/100`
before the pass, regardless of the home station's own account balance —
and after a purchase completes, the player's credits (not the station's)
decrease by the purchase cost.
