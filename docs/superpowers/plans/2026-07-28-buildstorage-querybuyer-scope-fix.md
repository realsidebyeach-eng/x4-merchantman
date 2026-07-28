# Build Storage Storage-Check Wrong-Object Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop the version-17 storage-capacity check from querying the wrong object during Build Storage passes, which currently clamps every Build Storage buy amount to 0 and leaves ships assigned to under-construction stations permanently idle.

**Architecture:** `aiscripts/sbe_stationtrader.xml` services two possible buyer categories per pass in a shared loop (`$categorybuyers`: the base station for production, `$stationbuildstorage` = `$homestation.buildstorage` for construction demand), tracked via `$querybuyer` — set once per category iteration and used for the actual offer search (`find_buy_offer buyer="$querybuyer"`). The version-17 fix hardcoded `$homestation.cargo.{ware}.free` in three places inside this loop, which is correct only when `$querybuyer == $homestation` (the production category) and wrong for Build Storage (`$querybuyer == $stationbuildstorage`, a different object). This plan swaps all three `$homestation.cargo` references to `$querybuyer.cargo` — the variable that's actually correct for whichever category is currently processing, and which is already in scope at every edit point.

**Tech Stack:** X4: Foundations `aiscript` XML. Verification is `xmllint --noout` (this repo's standard gate). This is a variable-reference correction using a variable (`$querybuyer`) already proven in scope and already used one property away (`find_buy_offer buyer="$querybuyer"`) — no new schema surface introduced.

## Global Constraints

- Every task ends with `xmllint --noout aiscripts/sbe_stationtrader.xml` passing.
- Bump the aiscript `version` attribute (currently 17, so 18) — this project bumps version on every functional change.
- Do not touch `$stationbuildstorage`'s definition, the category-buyer list construction, or any other part of the dual-category loop — only the storage-check object reference changes.
- Do not touch the stray-cargo-to-home fallback, the Selling pipeline, the stray-cargo-to-external-buyer fallback, the price-floor logic, or the wallet-budget fix.
- No new param, no player-facing toggle.
- This is X4: Foundations aiscript XML — no unit-test framework exists; xmllint is the verification gate.

---

## Task 1: Swap $homestation.cargo for $querybuyer.cargo at all three storage-check sites

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml`

**Interfaces:** none — this task only changes which variable the existing `$amount`/`$deliverclamped` computations read from; both variables keep their existing names and downstream usage.

- [ ] **Step 1: Bump the aiscript version**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="17">
```

Replace with:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="18">
```

- [ ] **Step 2: Fix the fill-cargo buy pipeline's storage check**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
										<!-- $wantedoffer.amount is a demand target, not live storage room. $homestation.cargo.{$ware}.free is the station's real current free space for this ware - storage only, never money, since the home-delivery leg below always executes at price="0" and must never be limited by the station's account balance. -->
										<set_value name="$amount" exact="[$wareunitcap,$bestseller.amount,$wantedoffer.amount,$remainingbudget/$bestprice,$homestation.cargo.{$ware}.free].min"/>
```

Replace with:

```xml
										<!-- $wantedoffer.amount is a demand target, not live storage room. $querybuyer.cargo.{$ware}.free is the actual buyer's (base station, or its Build Storage) real current free space for this ware - storage only, never money, since the home-delivery leg below always executes at price="0" and must never be limited by the station's account balance. -->
										<set_value name="$amount" exact="[$wareunitcap,$bestseller.amount,$wantedoffer.amount,$remainingbudget/$bestprice,$querybuyer.cargo.{$ware}.free].min"/>
```

- [ ] **Step 3: Fix the fill-cargo delivery loop's storage re-check**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
								<!-- Re-check live storage room right before delivering: an earlier ware in this same batch may have already used up shared capacity. Storage only, never money - this leg always executes at price="0". -->
								<set_value name="$deliverclamped" exact="[$deliveramounts.{$d},$homestation.cargo.{$deliverwares.{$d}.ware}.free].min"/>
```

Replace with:

```xml
								<!-- Re-check live storage room right before delivering: an earlier ware in this same batch may have already used up shared capacity. Storage only, never money - this leg always executes at price="0". -->
								<set_value name="$deliverclamped" exact="[$deliveramounts.{$d},$querybuyer.cargo.{$deliverwares.{$d}.ware}.free].min"/>
```

- [ ] **Step 4: Fix the classic buy pipeline's storage check**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
										<!-- $wantedoffer.amount is a demand target, not live storage room. $homestation.cargo.{$ware}.free is the station's real current free space for this ware - storage only, never money, since the home-delivery leg below always executes at price="0" and must never be limited by the station's account balance. -->
										<set_value name="$amount" exact="[$cargocap,$bestseller.amount,$wantedoffer.amount,$spendablemoney/(2*$bestprice),$homestation.cargo.{$ware}.free].min"/>
```

Replace with:

```xml
										<!-- $wantedoffer.amount is a demand target, not live storage room. $querybuyer.cargo.{$ware}.free is the actual buyer's (base station, or its Build Storage) real current free space for this ware - storage only, never money, since the home-delivery leg below always executes at price="0" and must never be limited by the station's account balance. -->
										<set_value name="$amount" exact="[$cargocap,$bestseller.amount,$wantedoffer.amount,$spendablemoney/(2*$bestprice),$querybuyer.cargo.{$ware}.free].min"/>
```

- [ ] **Step 5: Validate XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 6: Verify the fix**

```bash
grep -c 'homestation.cargo' aiscripts/sbe_stationtrader.xml
grep -c 'querybuyer.cargo' aiscripts/sbe_stationtrader.xml
```
Expected: the first command shows `0` — no `$homestation.cargo` references remain anywhere in the file. The second command shows `5` — a `$querybuyer.cargo.{...}.free` reference in each of the 3 fixed `set_value` lines, plus 2 of the 3 explanatory comments also mention it in prose (the fill-cargo and classic buy-pipeline comments; the delivery-loop comment does not restate the property path) — same pattern as the version-17 fix's own verification, just with the variable renamed.

- [ ] **Step 7: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml
git commit -m "fix: check the actual per-category buyer's storage, not always the base station, so Build Storage isn't permanently clamped to zero"
```

---

## Task 2: In-game verification (manual — required before considering this fix done)

This mod has no automated test harness for aiscript behavior. This fix
directly targets the exact scenario that surfaced the bug in live testing
(ship KHT-578, assigned to a station under construction) — re-testing that
same scenario is the most direct verification available.

**Files:** none (manual QA against the installed extension).

- [ ] **Step 1: Deploy the updated mod**

```bash
rsync -a --delete --exclude '.git' --exclude '.claude' \
  ~/Projects/X4StationTrader/ \
  "$HOME/.local/share/Steam/steamapps/common/X4 Foundations/extensions/sbe_station_trader/"
```

- [ ] **Step 2: Reproduce the original stuck scenario**

1. Find a ship assigned to a station still under construction (Build
   Storage active, `manageconstruction` enabled).
2. With **Enable Debug Log** on, let the ship run a Build Storage buy pass
   and check the resulting `'... amount=... (wareunitcap=..., sellerhas=...,
   homewants=..., afford=...)'` debug lines.

- [ ] **Step 3: Confirm Build Storage amounts are no longer clamped to zero**

Expected: at least some wares now show `amount` greater than 0 (assuming
the build storage genuinely has room and the other caps — cargo, seller
stock, money — aren't independently limiting), and the ship starts buying
and delivering for construction again.

- [ ] **Step 4: Confirm the production category's storage check still works**

Reproduce (or recall from prior QA) the original LKT-678-style scenario
for the regular production category and confirm it still correctly clamps
against the station's real storage — i.e. confirm this fix didn't
regress the case it was already working for.
