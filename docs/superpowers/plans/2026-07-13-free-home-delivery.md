# Free Home-Station Delivery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop ships getting permanently stuck holding cargo when a home station's own account can't afford to pay for a delivery, by making every "deliver cargo already in the ship's hold to a home station" trade order free (`price="0"`).

**Architecture:** Three `create_trade_order` calls in `aiscripts/sbe_stationtrader.xml` currently let the trade offer's own price apply (the home station pays the ship at its buy-offer price): the buy pipeline's Fill-Cargo batch delivery, the buy pipeline's Classic-mode delivery, and the stray-cargo-to-home-station fallback. Adding `price="0"` to each removes the redundant second payment — the wares were already paid for once, when the ship bought them externally (or however stray cargo was acquired) — without touching amount, offer selection, or any other logic.

**Tech Stack:** X4: Foundations `aiscript` XML. Verification is `xmllint --noout` plus schema validation against the local `aiscripts.xsd` cache (`/home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/aiscripts.xsd`) — both already confirmed to pass for this exact change before this plan was written. In-game testing with **Enable Debug Log** confirms the functional fix.

## Global Constraints

- Every task ends with `xmllint --noout aiscripts/sbe_stationtrader.xml` passing.
- Bump the aiscript `version` attribute (currently 12, so 13) — this project bumps version on every functional change, even ones with no new param (precedent: the version-9 Logbook fix had no new param either).
- No new param, no toggle, no ownership check — this is an unconditional, always-on fix per the design.
- Do not touch the sell pipeline's pick-up leg (`$sellwantedoffer`, lines ~363/478) or export leg (`$sellbestbuyer`/`$sellexportbuyers`) — those are out of scope (see design doc's "Out of scope" section).

---

## Task 1: Set price="0" on the three home-delivery trade orders, bump version, update docs

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml` (version bump + 3 `create_trade_order` edits)
- Modify: `docs/CONFIGURATION.md` (new short explainer section)

**Interfaces:**
- None — this is a self-contained attribute change with no new variables, params, or text ids.

- [ ] **Step 1: Bump the aiscript version**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="12">
```

Replace with:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="13">
```

- [ ] **Step 2: Zero the stray-cargo-to-home-station delivery**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
											<create_trade_order name="$strayhomeoffer" object="this.object" tradeoffer="$strayhomeoffer" amount="[$strayamount,$strayhomeoffer.amount].min" immediate="false"/>
```

Replace with:

```xml
											<create_trade_order name="$strayhomeoffer" object="this.object" tradeoffer="$strayhomeoffer" amount="[$strayamount,$strayhomeoffer.amount].min" price="0" immediate="false"/>
```

- [ ] **Step 3: Zero the Fill-Cargo mode's batch delivery leg**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
							<create_trade_order name="$deliverwares.{$d}" object="this.object" tradeoffer="$deliverwares.{$d}" amount="$deliveramounts.{$d}" immediate="false"/>
```

Replace with:

```xml
							<create_trade_order name="$deliverwares.{$d}" object="this.object" tradeoffer="$deliverwares.{$d}" amount="$deliveramounts.{$d}" price="0" immediate="false"/>
```

- [ ] **Step 4: Zero the Classic mode's delivery leg**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
										<create_trade_order name="$wantedoffer" object="this.object" tradeoffer="$wantedoffer" amount="$amount" immediate="false"/>
```

Replace with:

```xml
										<create_trade_order name="$wantedoffer" object="this.object" tradeoffer="$wantedoffer" amount="$amount" price="0" immediate="false"/>
```

- [ ] **Step 5: Validate XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 6: Verify exactly three price="0" attributes were added, in the right places**

```bash
grep -n 'price="0"' aiscripts/sbe_stationtrader.xml
```
Expected: exactly 3 matches — one on the `$strayhomeoffer` line, one on the `$deliverwares.{$d}` line, one on the `$wantedoffer` line. Confirm no other `create_trade_order` call (the buy leg from `$bestseller`, the sell pipeline's `$sellwantedoffer`/`$sellbestbuyer`/`$sellexportbuyers` calls) was touched — those must NOT have `price="0"`.

- [ ] **Step 7: Add the docs explainer section**

In `docs/CONFIGURATION.md`, find:

```markdown
## Handling pirates and other interruptions

If a ship complies with a pirate "drop cargo" demand (or is otherwise
interrupted mid-delivery) it can end up idle, holding cargo that no longer
matches any of its trade orders. Every pass, if the ship currently has zero
pending trade orders and is still holding cargo, it tries to get rid of it
rather than sit there forever: first by delivering to a home station that
currently wants that ware, and if none do, by selling to the best-paying
accessible buyer within Max Jump Range (same docking/blacklist access
checks as normal buying). This only ever runs when no trade order is
already in progress, so it can't conflict with or double-sell cargo that's
legitimately mid-delivery.

## Troubleshooting
```

Replace with:

```markdown
## Handling pirates and other interruptions

If a ship complies with a pirate "drop cargo" demand (or is otherwise
interrupted mid-delivery) it can end up idle, holding cargo that no longer
matches any of its trade orders. Every pass, if the ship currently has zero
pending trade orders and is still holding cargo, it tries to get rid of it
rather than sit there forever: first by delivering to a home station that
currently wants that ware, and if none do, by selling to the best-paying
accessible buyer within Max Jump Range (same docking/blacklist access
checks as normal buying). This only ever runs when no trade order is
already in progress, so it can't conflict with or double-sell cargo that's
legitimately mid-delivery.

## Deliveries to your own home station are always free

Every trade order that delivers cargo already in the ship's hold to one of
its home stations — the normal restock delivery, the Build Storage
delivery, and the stray-cargo fallback above — is priced at 0cr. The wares
were already paid for once, when the ship bought them from an external
seller (or however stray cargo was acquired); charging the home station's
own account a second time for the same goods would be redundant, and if
that station's own account happened to be low on funds, it would block the
delivery entirely and leave the ship stuck holding cargo it could never
unload. This has no effect on the Logbook — completed-delivery entries
always show the real acquisition price (what was paid to the external
seller), never the home-delivery price.

## Troubleshooting
```

- [ ] **Step 8: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml docs/CONFIGURATION.md
git commit -m "fix: stop ships getting stuck when a home station can't afford a delivery"
```

---

## Task 2: In-game verification (manual — required before considering this fix done)

This mod has no automated test harness for aiscript behavior. The fix
itself carries no unverified-syntax risk — `price` is a documented,
schema-valid attribute of `create_trade_order` already confirmed via
`xmllint --noout --schema` before this plan was written — so this task is
purely a functional smoke test.

**Files:** none (manual QA against the installed extension).

- [ ] **Step 1: Deploy the updated mod**

```bash
rsync -a --delete --exclude '.git' --exclude '.claude' \
  ~/Projects/X4StationTrader/ \
  "$HOME/.local/share/Steam/steamapps/common/X4 Foundations/extensions/sbe_station_trader/"
```

- [ ] **Step 2: Reproduce the original bug's scenario, then confirm it's fixed**

1. Load a save with an owned station whose own account is deliberately low
   (or wait for one to naturally run low), with an assigned ship carrying
   cargo destined for it.
2. Set **Enable Debug Log** on and let the ship run a pass.
3. Confirm the delivery trade order actually completes (cargo leaves the
   ship's hold, station's stock increases) even though the station's own
   account balance is low or zero — this is the core regression test.
4. Confirm the station's own account balance is unaffected by the
   delivery (it shouldn't decrease at all for that transaction).

- [ ] **Step 3: Confirm normal deliveries still work when funds aren't an issue**

With a well-funded station, confirm deliveries still complete normally and
the Logbook entry still shows the real acquisition price (not 0cr) — e.g.
"Bought 500 Ore from Seller at 12cr/unit (6000cr total) to restock
Station." should be unchanged from before this fix.

- [ ] **Step 4: Confirm the sell pipeline is unaffected**

With **Enable Selling** on, confirm the pick-up leg (ship buying the
station's own surplus) and export leg (selling to an external buyer) still
show normal, non-zero prices in the debug trace and Logbook — this fix
must not have touched those legs.
