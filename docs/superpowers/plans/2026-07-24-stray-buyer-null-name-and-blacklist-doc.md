# Stray-Buyer Null-Name Fix + Blacklist Travel Limitation Doc Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the stray-cargo-to-external-buyer debug log always showing "selling ... to null" instead of the real buyer's name, and document (without any code change) that this mod cannot control whether a ship's automated travel routes through a blacklisted sector.

**Architecture:** `aiscripts/sbe_stationtrader.xml`'s stray-cargo fallback reads `$beststraybuyer.buyer.knownname` in a debug line *after* `create_trade_order` has already consumed that reference, returning null. Capture the name into a new `$beststraybuyername` variable immediately before the `create_trade_order` call — the same pattern already used at 4 other capture sites in this file (`$sellbuyername`, `$sellername`) — and use that variable in the debug text instead. Separately, add a README.md "Known limitations" bullet — no code change, since investigation confirmed no X4 script mechanism exists for an order script to influence ship travel routing.

**Tech Stack:** X4: Foundations `aiscript` XML. Verification is `xmllint --noout` plus schema validation against the local `aiscripts.xsd` cache (`/home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/aiscripts.xsd`) — both already confirmed to pass for this exact change before this plan was written.

## Global Constraints

- Every task ends with `xmllint --noout aiscripts/sbe_stationtrader.xml` passing.
- Bump the aiscript `version` attribute (currently 13, so 14) — this project bumps version on every functional change, even ones with no new param (precedent: the version-9 Logbook fix, the version-13 free-home-delivery fix).
- Only the stray-cargo-to-external-buyer path (`$beststraybuyer`) is touched — do not touch the stray-cargo-to-home path (`$strayhomeoffer`), which already has its own `price="0"` from the prior fix and does not have this bug (it never reads a name after order creation).
- No new param, no ownership check — the null-name fix is unconditional; the doc addition is prose only.
- This is X4: Foundations aiscript XML — no unit-test framework exists; xmllint well-formedness (and schema validation, since this domain's schema is available locally) is the verification gate.

---

## Task 1: Fix the "selling to null" debug log, bump version, document the blacklist travel limitation

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml` (version bump + capture-name fix)
- Modify: `README.md` (new Known Limitations bullet)

**Interfaces:**
- Produces: `$beststraybuyername` — a local variable scoped to this one `do_if` block, not consumed by any other task or later code.

- [ ] **Step 1: Bump the aiscript version**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="13">
```

Replace with:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="14">
```

- [ ] **Step 2: Capture the buyer's name before create_trade_order consumes it**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
								<do_if value="not ($beststraybuyer == 0)">
									<create_trade_order name="$beststraybuyer" object="this.object" tradeoffer="$beststraybuyer" amount="[$strayamount,$beststraybuyer.amount].min" immediate="false"/>
									<do_if value="$debugtrace">
										<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'Stray cargo: no home station wanted '+$strayware+', selling '+$strayamount+' to '+$beststraybuyer.buyer.knownname+' at '+$beststrayprice+'cr/unit instead.'" output="false" append="true" chance="100"/>
									</do_if>
								</do_if>
```

Replace with:

```xml
								<do_if value="not ($beststraybuyer == 0)">
									<!-- Capture the buyer's name now, before create_trade_order consumes $beststraybuyer below (reading $beststraybuyer.buyer afterwards returns null). -->
									<set_value name="$beststraybuyername" exact="$beststraybuyer.buyer.knownname"/>
									<create_trade_order name="$beststraybuyer" object="this.object" tradeoffer="$beststraybuyer" amount="[$strayamount,$beststraybuyer.amount].min" immediate="false"/>
									<do_if value="$debugtrace">
										<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'Stray cargo: no home station wanted '+$strayware+', selling '+$strayamount+' to '+$beststraybuyername+' at '+$beststrayprice+'cr/unit instead.'" output="false" append="true" chance="100"/>
									</do_if>
								</do_if>
```

- [ ] **Step 3: Validate XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 4: Verify the fix and that nothing else changed**

```bash
grep -n 'beststraybuyername' aiscripts/sbe_stationtrader.xml
grep -c 'beststraybuyer\.buyer\.knownname' aiscripts/sbe_stationtrader.xml
```
Expected: the first command shows exactly 2 matches (the `set_value` and the `debug_to_file` use) both inside the stray-cargo block around line 216-223; the second command shows `0` — no remaining raw `$beststraybuyer.buyer.knownname` reads anywhere in the file.

- [ ] **Step 5: Add the Known Limitations bullet to README.md**

In `README.md`, find:

```markdown
- **Absolute Max Price is a single flat value for every ware in the list**,
  not an independently configurable per-ware table — X4's native ship-order
  parameter UI doesn't support a "table of ware → number" widget. Use
  **Price Cap Mode = on** (percent below home price) for genuine per-ware
  differentiation instead.

## Files
```

Replace with:

```markdown
- **Absolute Max Price is a single flat value for every ware in the list**,
  not an independently configurable per-ware table — X4's native ship-order
  parameter UI doesn't support a "table of ware → number" widget. Use
  **Price Cap Mode = on** (percent below home price) for genuine per-ware
  differentiation instead.
- **A ship's automated travel can still pass through a blacklisted sector**
  as a waypoint en route to a legitimate, non-blacklisted trade
  destination. This mod never issues travel/movement commands itself —
  `create_trade_order` hands the actual flying to X4's standard trade-order
  AI, the same mechanism every trade ship in the game uses. There is no
  route-safety/avoidance action exposed anywhere in the X4 script schema
  for an order script to use, so this mod has no way to influence the
  flight path taken to reach a chosen destination — only which destination
  gets chosen in the first place (which does correctly avoid blacklisted
  sectors). Lowering **Max Jump Range** reduces how far a ship will reach
  for a deal, which is the only available mitigation.

## Files
```

- [ ] **Step 6: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml README.md
git commit -m "fix: stop stray-cargo Logbook/debug line showing null buyer name, document sector-blacklist travel limitation"
```

---

## Task 2: In-game verification (manual — required before considering this fix done)

This mod has no automated test harness for aiscript behavior. The fix
itself carries no unverified-syntax risk (`set_value`/`debug_to_file` are
already proven working throughout this file) — this task is a functional
smoke test of the log output, not a syntax-risk mitigation.

**Files:** none (manual QA against the installed extension).

- [ ] **Step 1: Deploy the updated mod**

```bash
rsync -a --delete --exclude '.git' --exclude '.claude' \
  ~/Projects/X4StationTrader/ \
  "$HOME/.local/share/Steam/steamapps/common/X4 Foundations/extensions/sbe_station_trader/"
```

- [ ] **Step 2: Reproduce the stray-cargo-to-external-buyer scenario**

1. With **Enable Debug Log** on for a ship, let it end up with cargo its
   home station doesn't want (e.g. after a forced pirate "drop cargo"
   demand, or by removing a ware from its Ware Priority List while it's
   holding stock of that ware).
2. Let it run a pass and find the resulting debug log line starting with
   `Stray cargo: no home station wanted`.

- [ ] **Step 3: Confirm the buyer name is real, not null**

Expected: the line now reads something like `Stray cargo: no home station
wanted Hull Parts, selling 571 to <Real Station Name> at 299cr/unit
instead.` — a real station/object name, never the literal word `null`.
