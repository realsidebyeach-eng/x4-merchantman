# Storage Clamp Money-Entanglement Regression Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop the version-16 `clamp_trade_amount` storage checks from silently gating trader ships on their home station's account balance, since home deliveries always execute at `price="0"` and must never depend on the station's money.

**Architecture:** `aiscripts/sbe_stationtrader.xml`'s three `clamp_trade_amount` calls (added in the version-16 storage-capacity-checks fix) each clamp against both the buyer's storage *and* its ability to afford the trade at the trade offer's real configured price — but the home-delivery leg they feed into always executes at `price="0"`, so that money check is both wrong and the direct cause of ships going stationary until their home station's account happened to have funds. Remove all three `clamp_trade_amount` calls and replace each with a plain `$homestation.cargo.{ware}.free` term folded into the existing `min()` expression — the same property this file already uses for the ship's own cargo hold (`this.ship.cargo.{$ware}.free`), and confirmed to work identically on a station object by TaterTrade (`extensions/ws_2082610969/aiscripts/tatertrade.xml:992`), a mature, currently-installed reference mod. This checks storage only, never money.

**Tech Stack:** X4: Foundations `aiscript` XML. Verification is `xmllint --noout` (this repo's standard gate). No schema validation needed this time (unlike the fix being corrected) — this change only removes an action and adds a plain arithmetic term to an existing `min()` expression, using a property (`.cargo.{ware}.free`) already proven twice over: once by this file's own pre-existing ship-side usage, once by TaterTrade's station-side usage.

## Global Constraints

- Every task ends with `xmllint --noout aiscripts/sbe_stationtrader.xml` passing.
- Bump the aiscript `version` attribute (currently 16, so 17) — this project bumps version on every functional change.
- Do not touch the stray-cargo-to-home fallback (`$strayhandled`/`$strayhomeamount`, version 16's second fix) — it never called `clamp_trade_amount` and has no part of this regression.
- Do not touch the Selling pipeline, the stray-cargo-to-external-buyer fallback, the price-floor logic, or the wallet-budget fix (version 15) — all out of scope.
- No new param, no player-facing toggle.
- `$deliverclamped` keeps its existing name and role (used as the `create_trade_order` amount) — only how it's computed changes.
- This is X4: Foundations aiscript XML — no unit-test framework exists; xmllint is the verification gate.

---

## Task 1: Replace all three clamp_trade_amount calls with money-blind storage checks

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml`

**Interfaces:** none — this task only changes how `$amount` and `$deliverclamped` are computed; both variables keep their existing names and downstream usage.

- [ ] **Step 1: Bump the aiscript version**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="16">
```

Replace with:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="17">
```

- [ ] **Step 2: Fix the fill-cargo buy pipeline's amount check**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
										<set_value name="$amount" exact="[$wareunitcap,$bestseller.amount,$wantedoffer.amount,$remainingbudget/$bestprice].min"/>
										<!-- $wantedoffer.amount is a demand target, not live storage room - clamp against the home's actual current capacity so the ship never buys more than it can deliver. -->
										<clamp_trade_amount trade="$wantedoffer" amount="$amount" buyer="$homestation" result="$amount"/>
```

Replace with:

```xml
										<!-- $wantedoffer.amount is a demand target, not live storage room. $homestation.cargo.{$ware}.free is the station's real current free space for this ware - storage only, never money, since the home-delivery leg below always executes at price="0" and must never be limited by the station's account balance. -->
										<set_value name="$amount" exact="[$wareunitcap,$bestseller.amount,$wantedoffer.amount,$remainingbudget/$bestprice,$homestation.cargo.{$ware}.free].min"/>
```

- [ ] **Step 3: Fix the fill-cargo delivery loop's re-check**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
							<do_all exact="$deliverwares.count" counter="$d">
								<!-- Re-check live storage room right before delivering: an earlier ware in this same batch may have already used up shared capacity. -->
								<clamp_trade_amount trade="$deliverwares.{$d}" amount="$deliveramounts.{$d}" buyer="$homestation" result="$deliverclamped"/>
								<create_trade_order name="$deliverwares.{$d}" object="this.object" tradeoffer="$deliverwares.{$d}" amount="$deliverclamped" price="0" immediate="false"/>
```

Replace with:

```xml
							<do_all exact="$deliverwares.count" counter="$d">
								<!-- Re-check live storage room right before delivering: an earlier ware in this same batch may have already used up shared capacity. Storage only, never money - this leg always executes at price="0". -->
								<set_value name="$deliverclamped" exact="[$deliveramounts.{$d},$homestation.cargo.{$deliverwares.{$d}.ware}.free].min"/>
								<create_trade_order name="$deliverwares.{$d}" object="this.object" tradeoffer="$deliverwares.{$d}" amount="$deliverclamped" price="0" immediate="false"/>
```

- [ ] **Step 4: Fix the classic buy pipeline's amount check**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
										<set_value name="$amount" exact="[$cargocap,$bestseller.amount,$wantedoffer.amount,$spendablemoney/(2*$bestprice)].min"/>
										<!-- $wantedoffer.amount is a demand target, not live storage room - clamp against the home's actual current capacity so the ship never buys more than it can deliver. -->
										<clamp_trade_amount trade="$wantedoffer" amount="$amount" buyer="$homestation" result="$amount"/>
```

Replace with:

```xml
										<!-- $wantedoffer.amount is a demand target, not live storage room. $homestation.cargo.{$ware}.free is the station's real current free space for this ware - storage only, never money, since the home-delivery leg below always executes at price="0" and must never be limited by the station's account balance. -->
										<set_value name="$amount" exact="[$cargocap,$bestseller.amount,$wantedoffer.amount,$spendablemoney/(2*$bestprice),$homestation.cargo.{$ware}.free].min"/>
```

- [ ] **Step 5: Validate XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 6: Verify the fix**

```bash
grep -c 'clamp_trade_amount' aiscripts/sbe_stationtrader.xml
grep -c 'homestation.cargo' aiscripts/sbe_stationtrader.xml
```
Expected: the first command shows `0` — no `clamp_trade_amount` calls remain anywhere in the file. The second command shows `5` — a `$homestation.cargo.{...}.free` reference in each of the 3 fixed `set_value` lines, plus 2 of the 3 explanatory comments above them also mention it in prose (the fill-cargo and classic buy-pipeline comments; the delivery-loop comment does not restate the property path).

- [ ] **Step 7: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml
git commit -m "fix: stop trader buy/deliver checks from gating on the home station's account balance"
```

---

## Task 2: In-game verification (manual — required before considering this fix done)

This mod has no automated test harness for aiscript behavior. Confidence in
this specific change is high (the replacement property is proven both by
this file's own pre-existing ship-side usage and by TaterTrade's
station-side usage), but this corrects a regression that was only caught by
manual in-game testing in the first place — manual verification remains the
real gate.

**Files:** none (manual QA against the installed extension).

- [ ] **Step 1: Deploy the updated mod**

```bash
rsync -a --delete --exclude '.git' --exclude '.claude' \
  ~/Projects/X4StationTrader/ \
  "$HOME/.local/share/Steam/steamapps/common/X4 Foundations/extensions/sbe_station_trader/"
```

- [ ] **Step 2: Reproduce the original stuck scenario**

1. Find (or set up) a trader ship whose home station's account balance is
   very low or zero, while the station still has an active buy offer for
   some ware.
2. With **Enable Debug Log** on, let the ship run a buy pass and confirm it
   creates a buy order and a home-delivery order as normal — it should no
   longer sit stationary regardless of the station's account balance.

- [ ] **Step 3: Confirm storage clamping still works**

Reproduce the original LKT-678-style scenario (home station's warehouse
for a ware is nearly full) and confirm the ship still doesn't overbuy
relative to real storage room — i.e. confirm this fix didn't regress the
version-16 storage-capacity fix's own original purpose, only its
money-entanglement side effect.

- [ ] **Step 4: Confirm the account balance is irrelevant**

With a ship mid-buy-cycle, deliberately reduce the home station's account
to 0 (or confirm it's already near 0) and verify the ship continues buying
and delivering wares for that station without interruption, as long as
storage room and player money remain available.
