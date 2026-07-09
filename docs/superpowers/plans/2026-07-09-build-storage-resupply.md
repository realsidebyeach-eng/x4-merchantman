# Build Storage Resupply Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the Station Trader ship order also resupply a home station's Build Storage (construction-phase demand), controlled by two new params, alongside its existing production-ware resupply.

**Architecture:** The per-home-station processing block in `aiscripts/sbe_stationtrader.xml` currently queries `find_buy_offer buyer="$homestation"` once per station per pass. We wrap that block in an additional loop over 1–2 "category buyers" (the station itself, and — if present and enabled — `$homestation.buildstorage`), ordered by a new `buildstoragefirst` toggle, so each category runs the existing Priority/Balanced/Fill-Cargo pipeline independently and in full before the next category is considered. This is the only way to guarantee "Build Storage first" actually means cargo/budget gets spent there before production wares touch what's left — a shared list re-sorted by the existing Priority/Balanced logic would not preserve that guarantee. Price-ceiling lookups (`find_sell_offer seller=`) stay pinned to the real station, not the buildstorage sub-object, since Build Storage doesn't sell wares. Debug log and Logbook output gets a "(Build Storage)" tag on the construction category so troubleshooting can tell the two apart.

**Tech Stack:** X4: Foundations `aiscript` XML (custom Egosoft schema, no general-purpose runtime/test framework). Verification is `xmllint --noout` for structural validity plus in-game testing with the mod's own `Enable Debug Log` param — there is no automated test harness for aiscript logic itself.

## Global Constraints

- No `var`/no dynamic typing concerns apply (N/A — this is declarative XML, not a general-purpose language), but **every new param must follow the existing style**: `default`, `type`, `text="{8834271,N}"`, and a `comment=` attribute, exactly like `balanced`/`fillcargo` in the current file.
- **Never bump `content.xml`'s `version` attribute** — established repo pattern leaves it at `1` across all prior commits; only the aiscript's own internal `version` attribute (used by `<patch sinceversion="N">`) gets bumped.
- New text ids must not collide with existing ones (currently used: 101–109, 111, 112, 300, 500, 501 — id 110 was intentionally retired, do not reuse it).
- Preserve the existing 3-tabs-per-nesting-level indentation convention **only at edit boundaries** (opening/closing tags you add). Do **not** re-indent the ~300 lines of unchanged body content this plan wraps in a new loop — re-indenting the whole block for cosmetic consistency would turn every task into a full-file rewrite for zero functional benefit and much higher risk of a transcription error breaking the mod. XML validity does not depend on indentation depth.
- Every task ends with `xmllint --noout aiscripts/sbe_stationtrader.xml` (and `t/0001.xml` where touched) passing — this is the closest equivalent to a compile check this domain has.

---

## Task 1: Add the two new order parameters, their text strings, and the version/patch bump

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml:4` (version bump)
- Modify: `aiscripts/sbe_stationtrader.xml:20-21` (insert two new `<param>` entries)
- Modify: `aiscripts/sbe_stationtrader.xml:89-93` (insert new `<patch sinceversion="11">` block after the existing `sinceversion="8"` one)
- Modify: `t/0001.xml` (insert two new `<t>` entries)

**Interfaces:**
- Produces: order params `$manageconstruction` (bool, default `true`) and `$buildstoragefirst` (bool, default `true`), readable by name from anywhere in the script — consumed by Task 2.
- Produces: text ids `{8834271,113}` and `{8834271,114}` for the two new param labels.

- [ ] **Step 1: Bump the aiscript version**

In `aiscripts/sbe_stationtrader.xml`, change line 4:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="10">
```

to:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="11">
```

- [ ] **Step 2: Add the two new params**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
			<param name="fillcargo" default="true" type="bool" text="{8834271,112}" comment="Fill the cargo hold across all wanted wares before returning home to deliver."/>
			<!-- How many jumps from a home station's own sector we are willing to search for a cheap seller. -->
			<param name="maxjumps" default="3" type="number" text="{8834271,104}" comment="Max jump range from home.">
```

Replace with:

```xml
			<param name="fillcargo" default="true" type="bool" text="{8834271,112}" comment="Fill the cargo hold across all wanted wares before returning home to deliver."/>
			<!-- true: also resupply a home station's Build Storage (construction-phase demand for a station under construction or having a module added) alongside its normal production buy offers. -->
			<param name="manageconstruction" default="true" type="bool" text="{8834271,113}" comment="Also resupply Build Storage (construction wares) for stations under construction/expansion."/>
			<!-- true: service Build Storage's wanted wares before the station's own production-wanted wares each pass, when both have demand this pass. false: production wares first. -->
			<param name="buildstoragefirst" default="true" type="bool" text="{8834271,114}" comment="Service Build Storage before production wares when both need supplies this pass."/>
			<!-- How many jumps from a home station's own sector we are willing to search for a cheap seller. -->
			<param name="maxjumps" default="3" type="number" text="{8834271,104}" comment="Max jump range from home.">
```

- [ ] **Step 3: Add the version-11 patch block**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
		<patch sinceversion="8">
			<do_if value="$fillcargo == null">
				<set_value name="$fillcargo" exact="true"/>
			</do_if>
		</patch>
```

Replace with:

```xml
		<patch sinceversion="8">
			<do_if value="$fillcargo == null">
				<set_value name="$fillcargo" exact="true"/>
			</do_if>
		</patch>
		<patch sinceversion="11">
			<do_if value="$manageconstruction == null">
				<set_value name="$manageconstruction" exact="true"/>
			</do_if>
			<do_if value="$buildstoragefirst == null">
				<set_value name="$buildstoragefirst" exact="true"/>
			</do_if>
		</patch>
```

- [ ] **Step 4: Add the two new text strings**

In `t/0001.xml`, find:

```xml
		<t id="112">Fill Cargo Before Returning (buy every wanted ware first, deliver all at once; off = classic immediate buy-then-deliver per ware)</t>
		<t id="104">Max Jump Range (sectors from home)</t>
```

Replace with:

```xml
		<t id="112">Fill Cargo Before Returning (buy every wanted ware first, deliver all at once; off = classic immediate buy-then-deliver per ware)</t>
		<t id="113">Also Resupply Build Storage (buy construction wares for stations under construction or being expanded)</t>
		<t id="114">Service Build Storage First (before production wares, when both need supplies this pass)</t>
		<t id="104">Max Jump Range (sectors from home)</t>
```

- [ ] **Step 5: Validate XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && xmllint --noout t/0001.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 6: Verify the new params and text ids are present and nothing else changed**

Run:
```bash
grep -n 'manageconstruction\|buildstoragefirst' aiscripts/sbe_stationtrader.xml
grep -n '"113"\|"114"' t/0001.xml
```
Expected: the param/patch lines from Steps 2–3 and the two `<t id="113">`/`<t id="114">` lines from Step 4, and nothing unrelated.

- [ ] **Step 7: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml t/0001.xml
git commit -m "feat: add Also Resupply Build Storage and Build Storage First params"
```

---

## Task 2: Wrap per-station processing in a category loop (station + its Build Storage)

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml:209-217` (insert category-list construction + open new `do_all` loop)
- Modify: `aiscripts/sbe_stationtrader.xml:216` (now shifted — the `find_buy_offer buyer=` line; change `$homestation` to `$querybuyer`)
- Modify: `aiscripts/sbe_stationtrader.xml:511-515` (close the new `do_all` loop + cleanup)

**Interfaces:**
- Consumes: `$manageconstruction`, `$buildstoragefirst` from Task 1.
- Produces: per-category-iteration variables `$querybuyer` (the buy-offer buyer object to query — either `$homestation` or `$homestation.buildstorage`), `$categorylabel` (string, `''` or `' (Build Storage)'`), `$isbuildstorage` (bool) — all consumed by Task 3.
- Everything between the old lines 218–510 keeps its existing variable names (`$wantedoffers`, `$homebuyoffers`, `$processoffers`, etc.) untouched in this task; only the one `buyer=` attribute changes.

- [ ] **Step 1: Open the category loop and build the buyer list**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
			<!-- Process every assigned home station in turn. -->
			<do_all exact="$home.count" counter="$h2">
				<set_value name="$homestation" exact="$home.{$h2}"/>
				<do_if value="not ($homestation == null)">

					<!-- What does this station currently want to buy? If a Ware Priority List was given, restrict to that; otherwise auto-manage everything the station wants. -->
					<create_list name="$wantedoffers"/>
					<find_buy_offer buyer="$homestation" result="$homebuyoffers" multiple="true"/>
```

Replace with:

```xml
			<!-- Process every assigned home station in turn. -->
			<do_all exact="$home.count" counter="$h2">
				<set_value name="$homestation" exact="$home.{$h2}"/>
				<do_if value="not ($homestation == null)">

					<!-- Build the ordered list of buyers to service this pass: the station's own production buy offers, and - if present and enabled - its Build Storage construction demand. Order is controlled by Build Storage First. Each category runs the full existing priority/balanced/fill-cargo pipeline independently, so "first" actually means first claim on cargo space and budget. -->
					<create_list name="$categorybuyers"/>
					<create_list name="$categorylabels"/>
					<create_list name="$categoryflags"/>
					<set_value name="$stationbuildstorage" exact="$homestation.buildstorage"/>
					<set_value name="$hasbuildstorage" exact="$manageconstruction and not ($stationbuildstorage == null)"/>
					<do_if value="$hasbuildstorage and $buildstoragefirst">
						<append_to_list name="$categorybuyers" exact="$stationbuildstorage"/>
						<append_to_list name="$categorylabels" exact="' (Build Storage)'"/>
						<append_to_list name="$categoryflags" exact="true"/>
					</do_if>
					<append_to_list name="$categorybuyers" exact="$homestation"/>
					<append_to_list name="$categorylabels" exact="''"/>
					<append_to_list name="$categoryflags" exact="false"/>
					<do_if value="$hasbuildstorage and not $buildstoragefirst">
						<append_to_list name="$categorybuyers" exact="$stationbuildstorage"/>
						<append_to_list name="$categorylabels" exact="' (Build Storage)'"/>
						<append_to_list name="$categoryflags" exact="true"/>
					</do_if>

					<do_all exact="$categorybuyers.count" counter="$cat">
						<set_value name="$querybuyer" exact="$categorybuyers.{$cat}"/>
						<set_value name="$categorylabel" exact="$categorylabels.{$cat}"/>
						<set_value name="$isbuildstorage" exact="$categoryflags.{$cat}"/>

						<!-- What does this buyer currently want to buy? If a Ware Priority List was given, restrict to that; otherwise auto-manage everything it wants. -->
						<create_list name="$wantedoffers"/>
						<find_buy_offer buyer="$querybuyer" result="$homebuyoffers" multiple="true"/>
```

- [ ] **Step 2: Validate XML after Step 1**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: an error — the file is now unbalanced because the closing tags for this new `do_all` haven't been added yet. This is expected at this point; continue to Step 3 before validating again.

- [ ] **Step 3: Close the category loop and clean up**

In `aiscripts/sbe_stationtrader.xml`, find (this is the original tail of the per-station block, now the tail of the per-category body):

```xml
						<remove_value name="$processoffers"/>
					</do_if>
					<remove_value name="$homebuyoffers"/>
					<remove_value name="$wantedoffers"/>
				</do_if>
			</do_all>
```

Replace with:

```xml
						<remove_value name="$processoffers"/>
					</do_if>
					<remove_value name="$homebuyoffers"/>
					<remove_value name="$wantedoffers"/>
					</do_all>
					<remove_value name="$categorybuyers"/>
					<remove_value name="$categorylabels"/>
					<remove_value name="$categoryflags"/>
					<remove_value name="$stationbuildstorage"/>
				</do_if>
			</do_all>
```

- [ ] **Step 4: Validate XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors. If it still fails, the tag balance is wrong — re-check that Step 1 opened exactly one new `<do_all>` and Step 3 closed exactly one, and that no other `<do_all>`/`</do_all>` pair in the file was accidentally touched.

- [ ] **Step 5: Verify the price-reference lookups were NOT changed**

Build Storage doesn't sell wares, so the percent-cap price-reference lookup must keep pointing at the real station, not `$querybuyer`. Confirm both instances are untouched:

```bash
grep -n 'find_sell_offer seller="\$homestation"' aiscripts/sbe_stationtrader.xml
```
Expected: 2 matches (one in the Fill Cargo path, one in the classic path) — same count as before this task.

- [ ] **Step 6: Verify only the intended `buyer=` attribute changed**

```bash
grep -n 'buyer="\$homestation"\|buyer="\$querybuyer"' aiscripts/sbe_stationtrader.xml
```
Expected: exactly one `buyer="$querybuyer"` match (the `find_buy_offer` from Step 1), and zero `buyer="$homestation"` matches — the stray-cargo section earlier in the file uses `buyer="this.ship"` and other distinct queries, not `buyer="$homestation"`, so this should not affect it. If a `buyer="$homestation"` match unexpectedly remains, find it and confirm whether it's inside the stray-cargo block (line ~143-207, out of scope for this task — leave it) before treating it as an error.

- [ ] **Step 7: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml
git commit -m "feat: query Build Storage buy offers alongside station production offers"
```

---

## Task 3: Tag debug log lines and Logbook entries with the Build Storage label

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml` (4 `debug_to_file` lines, 2 `write_to_logbook` lines)
- Modify: `t/0001.xml` (1 new `<t>` entry)

**Interfaces:**
- Consumes: `$categorylabel` and `$isbuildstorage` from Task 2.
- Produces: text id `{8834271,301}` — the Build Storage variant of the delivery Logbook message.

- [ ] **Step 1: Add the Build Storage Logbook text variant**

In `t/0001.xml`, find:

```xml
		<t id="300">Bought %1 %2 from %3 at %4cr/unit (%5cr total) to restock %6.</t>
```

Replace with:

```xml
		<t id="300">Bought %1 %2 from %3 at %4cr/unit (%5cr total) to restock %6.</t>
		<t id="301">Bought %1 %2 from %3 at %4cr/unit (%5cr total) to fill Build Storage at %6.</t>
```

- [ ] **Step 2: Tag the four debug_to_file lines that name the station**

In `aiscripts/sbe_stationtrader.xml`, find each of these four lines (they are not adjacent — find and replace each independently) and change `$homestation.knownname+':'` to `$homestation.knownname+$categorylabel+':'` in each (note the exact text differs after the colon in each line — only the `$homestation.knownname+'` prefix changes):

Line A — find:
```xml
							<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+': '+$homebuyoffers.count+' total buy offers, '+$wantedoffers.count+' match the ware priority list.'" output="false" append="true" chance="100"/>
```
Replace with:
```xml
							<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+$categorylabel+': '+$homebuyoffers.count+' total buy offers, '+$wantedoffers.count+' match the ware priority list.'" output="false" append="true" chance="100"/>
```

Line B — find:
```xml
							<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+': '+$homebuyoffers.count+' total buy offers, auto-managing all of them (no ware priority list set).'" output="false" append="true" chance="100"/>
```
Replace with:
```xml
							<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+$categorylabel+': '+$homebuyoffers.count+' total buy offers, auto-managing all of them (no ware priority list set).'" output="false" append="true" chance="100"/>
```

Line C — find:
```xml
							<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+': fill pass starting with '+$remainingvolume+'m3 free cargo, '+$remainingbudget+'cr spendable, '+$processoffers.count+' wanted ware(s).'" output="false" append="true" chance="100"/>
```
Replace with:
```xml
							<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+$categorylabel+': fill pass starting with '+$remainingvolume+'m3 free cargo, '+$remainingbudget+'cr spendable, '+$processoffers.count+' wanted ware(s).'" output="false" append="true" chance="100"/>
```

Line D — find:
```xml
							<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+': fill pass done, delivering '+$deliverwares.count+' ware type(s) home in one trip ('+$remainingvolume+'m3 still free).'" output="false" append="true" chance="100"/>
```
Replace with:
```xml
							<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+$categorylabel+': fill pass done, delivering '+$deliverwares.count+' ware type(s) home in one trip ('+$remainingvolume+'m3 still free).'" output="false" append="true" chance="100"/>
```

- [ ] **Step 3: Validate XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 4: Switch the Fill Cargo path's Logbook entry between text id 300 and 301**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
								<do_if value="$enablelogbook">
									<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$deliveramounts.{$d}*$deliverprices.{$d}" text="{8834271,300}.[$deliveramounts.{$d},$deliverwares.{$d}.ware,$deliversellers.{$d},$deliverprices.{$d},$deliveramounts.{$d}*$deliverprices.{$d},$homestation.knownname]"/>
								</do_if>
```

Replace with:

```xml
								<do_if value="$enablelogbook">
									<do_if value="$isbuildstorage">
										<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$deliveramounts.{$d}*$deliverprices.{$d}" text="{8834271,301}.[$deliveramounts.{$d},$deliverwares.{$d}.ware,$deliversellers.{$d},$deliverprices.{$d},$deliveramounts.{$d}*$deliverprices.{$d},$homestation.knownname]"/>
									</do_if>
									<do_else>
										<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$deliveramounts.{$d}*$deliverprices.{$d}" text="{8834271,300}.[$deliveramounts.{$d},$deliverwares.{$d}.ware,$deliversellers.{$d},$deliverprices.{$d},$deliveramounts.{$d}*$deliverprices.{$d},$homestation.knownname]"/>
									</do_else>
								</do_if>
```

- [ ] **Step 5: Switch the classic path's Logbook entry between text id 300 and 301**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
											<do_if value="$enablelogbook">
												<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$amount*$bestprice" text="{8834271,300}.[$amount,$ware,$sellername,$bestprice,$amount*$bestprice,$homestation.knownname]"/>
											</do_if>
```

Replace with:

```xml
											<do_if value="$enablelogbook">
												<do_if value="$isbuildstorage">
													<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$amount*$bestprice" text="{8834271,301}.[$amount,$ware,$sellername,$bestprice,$amount*$bestprice,$homestation.knownname]"/>
												</do_if>
												<do_else>
													<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$amount*$bestprice" text="{8834271,300}.[$amount,$ware,$sellername,$bestprice,$amount*$bestprice,$homestation.knownname]"/>
												</do_else>
											</do_if>
```

- [ ] **Step 6: Validate XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && xmllint --noout t/0001.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 7: Verify tag counts balance and both text ids are wired up**

```bash
grep -c '<do_if' aiscripts/sbe_stationtrader.xml
grep -c '</do_if>' aiscripts/sbe_stationtrader.xml
grep -n '{8834271,301}' aiscripts/sbe_stationtrader.xml
grep -n '"301"' t/0001.xml
```
Expected: the two `do_if`/`</do_if>` counts are equal to each other; two matches for `{8834271,301}` (Fill Cargo path + classic path); one match for `<t id="301">`.

- [ ] **Step 8: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml t/0001.xml
git commit -m "feat: distinguish Build Storage deliveries in debug log and Logbook"
```

---

## Task 4: Update documentation

**Files:**
- Modify: `README.md` (remove the now-stale known-limitation bullet, add a capability bullet)
- Modify: `docs/CONFIGURATION.md` (parameter table rows, new explainer section, troubleshooting entry)

**Interfaces:**
- None (documentation only).

- [ ] **Step 1: Remove the stale known-limitation bullet from README.md**

In `README.md`, find:

```markdown
- **Absolute Max Price is a single flat value for every ware in the list**,
  not an independently configurable per-ware table — X4's native ship-order
  parameter UI doesn't support a "table of ware → number" widget. Use
  **Price Cap Mode = on** (percent below home price) for genuine per-ware
  differentiation instead.
- Station build-storage restocking (a separate buyer object some stations
  have) isn't included; only the station's own regular buy offers are read.
```

Replace with:

```markdown
- **Absolute Max Price is a single flat value for every ware in the list**,
  not an independently configurable per-ware table — X4's native ship-order
  parameter UI doesn't support a "table of ware → number" widget. Use
  **Price Cap Mode = on** (percent below home price) for genuine per-ware
  differentiation instead.
```

- [ ] **Step 2: Add a capability bullet to the "What it does" list in README.md**

In `README.md`, find:

```markdown
8. If the ship ends up idle holding cargo it can't otherwise account for
   — e.g. after complying with a pirate "drop cargo" demand mid-delivery —
   it tries to deliver it to a home station that wants it, or failing
   that, sell it to the best accessible buyer in range, rather than sit
   there holding it indefinitely.
```

Replace with:

```markdown
8. If the ship ends up idle holding cargo it can't otherwise account for
   — e.g. after complying with a pirate "drop cargo" demand mid-delivery —
   it tries to deliver it to a home station that wants it, or failing
   that, sell it to the best accessible buyer in range, rather than sit
   there holding it indefinitely.
9. It also resupplies a home station's Build Storage — construction-phase
   demand for a station under construction or having a module added —
   alongside its normal production buy offers, controlled by **Also
   Resupply Build Storage** and **Build Storage First**.
```

- [ ] **Step 3: Add the two new params to the CONFIGURATION.md parameter table**

In `docs/CONFIGURATION.md`, find:

```markdown
| **Fill Cargo Before Returning** | on/off | on | On = buy from every wanted ware first, tracking remaining cargo space and credits as it goes, then deliver everything home in one trip once the hold is full or nothing more is available/affordable. Off = classic mode, buy and immediately deliver one ware at a time. See below. |
| **Max Jump Range** | 0–10 | 3 | How many jumps from a home station's own sector to search for a seller. 0 = the home station's own sector only. |
```

Replace with:

```markdown
| **Fill Cargo Before Returning** | on/off | on | On = buy from every wanted ware first, tracking remaining cargo space and credits as it goes, then deliver everything home in one trip once the hold is full or nothing more is available/affordable. Off = classic mode, buy and immediately deliver one ware at a time. See below. |
| **Also Resupply Build Storage** | on/off | on | On = also buy construction wares a home station's Build Storage currently wants (station under construction or having a module added), same as normal production wares. Off = Build Storage is ignored entirely. See [Build Storage](#build-storage-construction-wares) below. |
| **Build Storage First** | on/off | on | Used only when Also Resupply Build Storage is on. On = Build Storage's wanted wares are fully serviced before the station's own production-wanted wares each pass. Off = production wares first. |
| **Max Jump Range** | 0–10 | 3 | How many jumps from a home station's own sector to search for a seller. 0 = the home station's own sector only. |
```

- [ ] **Step 4: Add the Build Storage explainer section**

In `docs/CONFIGURATION.md`, find:

```markdown
## Price cap: percentage vs. absolute — which to use
```

Replace with:

```markdown
## Build Storage (construction wares)

A station under construction — or an existing station with a module
being added — has a separate **Build Storage** resource pool distinct
from its normal production buy offers (in-game, this shows up as a
separate "trade with build storage" option when right-clicking the
station). With **Also Resupply Build Storage** on (the default), the ship
treats Build Storage's wanted wares the same way it treats the station's
own production wares — same Ware Priority List filter, same price-cap
rules, same Priority/Balanced/Fill-Cargo behavior — just queried from the
Build Storage object instead of the finished station.

**Build Storage First** (default on) decides which side gets first claim
on cargo space and credit budget when both the station and its Build
Storage want supplies in the same pass: each category is serviced fully
before the next is considered, so a stalled construction isn't left
waiting behind routine production restocking (or vice versa, if you turn
this off).

Once a station finishes building (or finishes its current module
addition), it no longer has a Build Storage object, so the ship simply
stops finding anything in that category and continues servicing normal
production wares as before — no reconfiguration needed.

## Price cap: percentage vs. absolute — which to use
```

- [ ] **Step 5: Add a troubleshooting entry**

In `docs/CONFIGURATION.md`, find:

```markdown
- **A deal you can see on the map isn't being taken, and none of the above
  explains it**: turn on Enable Debug Log and read the trace (see below) —
  it tells you exactly how many sectors were searched, how many sellers
  were found for that ware, and either the seller it picked or the
  cheapest price it saw that still came in above the ceiling.
```

Replace with:

```markdown
- **A deal you can see on the map isn't being taken, and none of the above
  explains it**: turn on Enable Debug Log and read the trace (see below) —
  it tells you exactly how many sectors were searched, how many sellers
  were found for that ware, and either the seller it picked or the
  cheapest price it saw that still came in above the ceiling.
- **Ship isn't buying construction wares for a station under
  construction**: confirm **Also Resupply Build Storage** is on, and that
  the station is actually showing a "trade with build storage" option
  in-game right now (a station between construction phases can briefly
  have no active Build Storage). If it's genuinely under construction and
  still not being serviced, turn on Enable Debug Log — lines tagged
  `(Build Storage)` show what was found for that category each pass.
```

- [ ] **Step 6: Commit**

```bash
git add README.md docs/CONFIGURATION.md
git commit -m "docs: document Build Storage resupply behavior and params"
```

---

## Task 5: In-game verification (manual — required before considering this feature done)

This mod has no automated test harness for aiscript behavior; the only
way to confirm the Build Storage query actually works is to run it inside
X4 itself. The design's flagged risk is the exact property used to reach
a station's Build Storage from script (`$homestation.buildstorage`,
introduced in Task 2 Step 1) — everything else in this plan does not
depend on that guess being right.

**Files:** none (manual QA against the installed extension).

- [ ] **Step 1: Deploy the updated mod**

```bash
rsync -a --delete --exclude '.git' \
  ~/Projects/X4StationTrader/ \
  "$HOME/.local/share/Steam/steamapps/common/X4 Foundations/extensions/sbe_station_trader/"
```

- [ ] **Step 2: Set up a test scenario in-game**

1. Load a save (or start a new one) where you either already have a
   station under construction, or place a new station / queue a module
   addition on an existing owned station so it has an active Build
   Storage.
2. Assign the Station Trader order to a ship, with that station as its
   Home Station.
3. Set **Enable Debug Log** to on. Leave **Also Resupply Build Storage**
   and **Build Storage First** at their defaults (both on).

- [ ] **Step 3: Read the debug trace**

Debug output writes to
`~/.config/EgoSoft/X4/<save-id>/StationTrader/<ship idcode>.txt` (native
Linux build). Let the ship run a few passes, then check for lines
containing `(Build Storage)`:

```bash
tail -f "$HOME/.config/EgoSoft/X4"/*/StationTrader/*.txt | grep --line-buffered 'Build Storage'
```

Expected: lines like `<Station Name> (Build Storage): N total buy
offers, ...` appear, and if a seller is found, a corresponding purchase
line — confirming `$homestation.buildstorage` resolved to a real object
and its buy offers were found.

- [ ] **Step 4: If no `(Build Storage)` lines ever appear**

This means the property name guessed in Task 2 Step 1 didn't resolve
(likely returning `null`, so `$hasbuildstorage` is always false and the
category is never added). To fix:

1. Confirm the station genuinely has an active Build Storage right now —
   right-click it in-game and check whether a "trade with build storage"
   / "exchange wares with buildstorage" option is present. If it isn't,
   the station has finished building and there's nothing to find (not a
   bug).
2. If the option is present but no `(Build Storage)` debug lines ever
   appear, the property name is wrong. Search the X4 scripting community
   (forum.egosoft.com Scripting & Modding subforum, or an extracted copy
   of a vanilla script that references build storage) for the correct
   property/query, then update the single line in
   `aiscripts/sbe_stationtrader.xml` from Task 2 Step 1:
   ```xml
   <set_value name="$stationbuildstorage" exact="$homestation.buildstorage"/>
   ```
   replacing `$homestation.buildstorage` with the confirmed correct
   expression. No other line in the plan needs to change — every
   downstream step only depends on `$stationbuildstorage` holding the
   right object or `null`, not on how it was obtained.
3. Re-run Steps 1–3 to confirm the fix.

- [ ] **Step 5: Confirm ordering behavior**

With a station that has both an active Build Storage *and* active
production buy offers at the same time (or simulate by watching over
several passes), confirm from the debug trace that entries for the
Build Storage category are fully processed (purchases queued or
"no seller under ceiling" logged for every wanted construction ware)
before the production category's entries begin, matching
**Build Storage First = on**. Toggle it off and confirm the order
flips.

- [ ] **Step 6: Confirm the Logbook tag**

With **Enable Logbook Entries** on, complete at least one Build Storage
delivery and confirm its Logbook entry (Menu → Logbook) reads "...to fill
Build Storage at `<station>`." rather than "...to restock `<station>`.".
