# Buy/Sell Trading Roles Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the Station Trader ship order two independent, toggleable roles — buying (existing restock behavior) and selling (new: export a home station's production surplus to the best-paying accessible buyer) — controlled by two new params, `enablebuying` and `enableselling`.

**Architecture:** The per-home-station block in `aiscripts/sbe_stationtrader.xml` currently always runs the buy-category pipeline (production buy offers + Build Storage). We wrap that existing pipeline in a `do_if value="$enablebuying"` guard, and insert a new sell pipeline — a full mirror of the buy pipeline's structure (query wanted offers, filter by Ware Priority List, Priority/Balanced ordering, Fill-Cargo-Before-Returning vs. classic mode, price bound, search accessible counterparties, place the trade-order pair) but reversed in direction — guarded by `do_if value="$enableselling"`, running before the buy pipeline per home station. A shared-library extraction (this design's original idea, to avoid duplicating ~275 lines) was checked against this machine's local `aiscripts.xsd` schema and confirmed **not supported** — the schema's only reusable-action mechanism (`<include_interrupt_actions>` / `interrupt_library`) is scoped to the `<interrupts>` block, not the main `<attention><actions>` body where this pipeline lives. So the sell pipeline is hand-written inline, following exactly the same proven pattern the (already-merged) Build Storage feature used for its own second concern.

**Tech Stack:** X4: Foundations `aiscript` XML (custom Egosoft schema, no general-purpose runtime/test framework). Verification is `xmllint --noout` for structural validity plus in-game testing with the mod's own `Enable Debug Log` param — there is no automated test harness for aiscript logic itself.

## Global Constraints

- Every new param must follow the existing style: `default`, `type`, `text="{8834271,N}"`, and a `comment=` attribute, exactly like `enablebuying`'s siblings (`manageconstruction`, `fillcargo`).
- **Never bump `content.xml`'s `version` attribute** — established repo pattern leaves it at `1` across all prior commits; only the aiscript's own internal `version` attribute (used by `<patch sinceversion="N">`) gets bumped.
- New text ids must not collide with existing ones (currently used: 101–109, 111–114, 300–301, 500–501 — id 110 was intentionally retired, do not reuse it). This plan adds 115, 116, and 302.
- Preserve the existing indentation convention **only at edit boundaries** (opening/closing tags you add). Do **not** re-indent unrelated unchanged lines for cosmetic consistency — XML validity does not depend on indentation depth, and re-indenting the whole file for every task multiplies transcription risk for zero functional benefit.
- Every task ends with `xmllint --noout aiscripts/sbe_stationtrader.xml` (and `t/0001.xml` where touched) passing — this is the closest equivalent to a compile check this domain has.
- No shared library / `<include_actions>` — confirmed unsupported by the local `aiscripts.xsd` schema (see Architecture above). Do not attempt this even as a "cleaner" alternative mid-implementation.

---

## Task 1: Add the two new order parameters, their text strings, version/patch bump, and description updates

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml:3` (top-of-file behavior comment)
- Modify: `aiscripts/sbe_stationtrader.xml:4` (version bump)
- Modify: `aiscripts/sbe_stationtrader.xml:6-16` (insert two new `<param>` entries after `home`, before `wareprio`)
- Modify: `aiscripts/sbe_stationtrader.xml:98-105` (insert new `<patch sinceversion="12">` block after the existing `sinceversion="11"` one)
- Modify: `t/0001.xml` (insert two new `<t>` entries near id 101, update id 501)

**Interfaces:**
- Produces: order params `$enablebuying` (bool, default `true`) and `$enableselling` (bool, default `true`), readable by name from anywhere in the script — consumed by Task 2 (`$enablebuying`) and Task 3 (`$enableselling`).
- Produces: text ids `{8834271,115}` and `{8834271,116}` for the two new param labels.

- [ ] **Step 1: Update the top-of-file behavior comment**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
<!-- Restocks one or more assigned home stations by buying the wares they currently want from the cheapest accessible seller within a configurable jump range. -->
```

Replace with:

```xml
<!-- Restocks one or more assigned home stations: buys the wares they currently want from the cheapest accessible seller, and/or sells their production surplus to the best-paying accessible buyer, within a configurable jump range. -->
```

- [ ] **Step 2: Bump the aiscript version**

In `aiscripts/sbe_stationtrader.xml`, change:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="11">
```

to:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="12">
```

- [ ] **Step 3: Add the two new params**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
			<!-- One or more stations this ship restocks. -->
			<param name="home" default="[]" type="list" text="{8834271,101}" comment="Home station(s) to restock.">
				<input_param name="type" value="'object'"/>
				<input_param name="class" value="[class.station]"/>
			</param>
			<!-- The wares this ship is allowed to manage, in priority order (top = highest priority). Leave empty to auto-manage every ware the station currently wants to buy. -->
			<param name="wareprio" default="[]" type="list" text="{8834271,102}" comment="Wares to manage, top to bottom priority. Leave empty to auto-manage everything the station wants.">
```

Replace with:

```xml
			<!-- One or more stations this ship restocks. -->
			<param name="home" default="[]" type="list" text="{8834271,101}" comment="Home station(s) to restock.">
				<input_param name="type" value="'object'"/>
				<input_param name="class" value="[class.station]"/>
			</param>
			<!-- true: buy wares this station currently wants from the cheapest accessible external seller and deliver them home. -->
			<param name="enablebuying" default="true" type="bool" text="{8834271,115}" comment="Buy wares the station wants and deliver them home."/>
			<!-- true: pick up wares this station currently has for sale (its production surplus) and sell them to the best-paying accessible external buyer. -->
			<param name="enableselling" default="true" type="bool" text="{8834271,116}" comment="Sell the station's surplus wares to the best accessible buyer."/>
			<!-- The wares this ship is allowed to manage, in priority order (top = highest priority). Leave empty to auto-manage every ware the station currently wants to buy. -->
			<param name="wareprio" default="[]" type="list" text="{8834271,102}" comment="Wares to manage, top to bottom priority. Leave empty to auto-manage everything the station wants.">
```

- [ ] **Step 4: Add the version-12 patch block**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
	<patch sinceversion="11">
		<do_if value="$manageconstruction == null">
			<set_value name="$manageconstruction" exact="true"/>
		</do_if>
		<do_if value="$buildstoragefirst == null">
			<set_value name="$buildstoragefirst" exact="true"/>
		</do_if>
	</patch>
```

Replace with:

```xml
	<patch sinceversion="11">
		<do_if value="$manageconstruction == null">
			<set_value name="$manageconstruction" exact="true"/>
		</do_if>
		<do_if value="$buildstoragefirst == null">
			<set_value name="$buildstoragefirst" exact="true"/>
		</do_if>
	</patch>
	<patch sinceversion="12">
		<do_if value="$enablebuying == null">
			<set_value name="$enablebuying" exact="true"/>
		</do_if>
		<do_if value="$enableselling == null">
			<set_value name="$enableselling" exact="true"/>
		</do_if>
	</patch>
```

- [ ] **Step 5: Validate the aiscript XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 6: Add the two new param text strings and update the order description**

In `t/0001.xml`, find:

```xml
		<t id="101">Home Station(s)</t>
		<t id="102">Ware Priority List (top = highest priority)</t>
```

Replace with:

```xml
		<t id="101">Home Station(s)</t>
		<t id="115">Enable Buying (buy wares the station wants and deliver them home)</t>
		<t id="116">Enable Selling (sell the station's surplus wares to the best accessible buyer)</t>
		<t id="102">Ware Priority List (top = highest priority)</t>
```

In `t/0001.xml`, find:

```xml
		<t id="501">Restocks the assigned home station(s) by buying their wanted wares from the cheapest accessible seller within range.</t>
```

Replace with:

```xml
		<t id="501">Restocks the assigned home station(s): buys their wanted wares from the cheapest accessible seller and/or sells their surplus wares to the best-paying accessible buyer, within range.</t>
```

- [ ] **Step 7: Validate the text file**

Run:
```bash
xmllint --noout t/0001.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 8: Verify the new params and text ids are present**

Run:
```bash
grep -n 'enablebuying\|enableselling' aiscripts/sbe_stationtrader.xml
grep -n '"115"\|"116"\|"501"' t/0001.xml
```
Expected: the param/patch lines from Steps 3–4 in the first command; the two new `<t id="115">`/`<t id="116">` lines and the updated `<t id="501">` line in the second.

- [ ] **Step 9: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml t/0001.xml
git commit -m "feat: add Enable Buying and Enable Selling params"
```

---

## Task 2: Gate the existing buy pipeline behind Enable Buying

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml` (wrap the existing per-station buy-category block in a new `do_if`)

**Interfaces:**
- Consumes: `$enablebuying` from Task 1.
- Produces: nothing new — purely a structural gate around existing, unmodified logic.

- [ ] **Step 1: Open the gate before the buy-category block**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
				<do_if value="not ($homestation == null)">

					<!-- Build the ordered list of buyers to service this pass: the station's own production buy offers, and - if present and enabled - its Build Storage construction demand. Order is controlled by Build Storage First. Each category runs the full existing priority/balanced/fill-cargo pipeline independently, so "first" actually means first claim on cargo space and budget. -->
					<create_list name="$categorybuyers"/>
```

Replace with:

```xml
				<do_if value="not ($homestation == null)">

					<do_if value="$enablebuying">
					<!-- Build the ordered list of buyers to service this pass: the station's own production buy offers, and - if present and enabled - its Build Storage construction demand. Order is controlled by Build Storage First. Each category runs the full existing priority/balanced/fill-cargo pipeline independently, so "first" actually means first claim on cargo space and budget. -->
					<create_list name="$categorybuyers"/>
```

- [ ] **Step 2: Validate XML after Step 1**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: an error — the file is now unbalanced because the closing tag for this new `do_if` hasn't been added yet. This is expected at this point; continue to Step 3 before validating again.

- [ ] **Step 3: Close the gate after the buy-category block**

In `aiscripts/sbe_stationtrader.xml`, find (this is the original tail of the buy-category loop):

```xml
					<remove_value name="$categorybuyers"/>
					<remove_value name="$categorylabels"/>
					<remove_value name="$categoryflags"/>
					<remove_value name="$stationbuildstorage"/>
				</do_if>
			</do_all>
```

Replace with:

```xml
					<remove_value name="$categorybuyers"/>
					<remove_value name="$categorylabels"/>
					<remove_value name="$categoryflags"/>
					<remove_value name="$stationbuildstorage"/>
					</do_if>
				</do_if>
			</do_all>
```

- [ ] **Step 4: Validate XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors. If it still fails, the tag balance is wrong — confirm Step 1 opened exactly one new `<do_if>` and Step 3 closed exactly one, and that no other `<do_if>`/`</do_if>` pair in the file was accidentally touched.

- [ ] **Step 5: Verify tag counts balance**

```bash
grep -c '<do_if' aiscripts/sbe_stationtrader.xml
grep -c '</do_if>' aiscripts/sbe_stationtrader.xml
```
Expected: the two counts are equal to each other.

- [ ] **Step 6: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml
git commit -m "feat: gate the buy pipeline behind Enable Buying"
```

---

## Task 3: Add the sell pipeline (Enable Selling)

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml` (insert the full sell pipeline immediately before the now-gated buy pipeline)
- Modify: `t/0001.xml` (insert one new `<t>` entry for the export Logbook message)

**Interfaces:**
- Consumes: `$enableselling`, `$wareprio`, `$balanced`, `$fillcargo`, `$debugtrace`, `$enablelogbook`, `$searchspaces`, `$blacklistgroup`, `$homestation`, `$scantick`, `$scantickrate` — all already in scope from earlier in the same per-home-station pass.
- Produces: text id `{8834271,302}` — the export/sell Logbook message. Produces no variables consumed by later tasks (this is the last piece of pipeline logic).
- Uses `find_sell_offer seller="$homestation"` to read the station's own surplus (mirrors the existing `find_sell_offer seller="$homestation"` calls already used elsewhere in the buy pipeline for price-reference lookups) and `find_buy_offer tradepartner="this.ship" ... match_buyer` to find external buyers (mirrors the exact query shape the stray-cargo dump logic already uses for the same purpose).

- [ ] **Step 1: Insert the sell pipeline**

In `aiscripts/sbe_stationtrader.xml`, find (this is the state after Task 2 — the blank line and the now-gated buy pipeline's opening):

```xml
				<do_if value="not ($homestation == null)">

					<do_if value="$enablebuying">
```

Replace with:

```xml
				<do_if value="not ($homestation == null)">

					<do_if value="$enableselling">
						<!-- What is this station currently selling (its production surplus)? If a Ware Priority List was given, restrict exports to that; otherwise auto-manage everything it currently has for sale. -->
						<create_list name="$sellwantedoffers"/>
						<find_sell_offer seller="$homestation" result="$homesselloffers" multiple="true"/>
						<do_if value="$wareprio.count gt 0">
							<do_all exact="$homesselloffers.count" counter="$so">
								<set_value name="$sellcandidate" exact="$homesselloffers.{$so}"/>
								<do_if value="$wareprio.indexof.{$sellcandidate.ware}">
									<append_to_list name="$sellwantedoffers" exact="$sellcandidate"/>
								</do_if>
							</do_all>
						</do_if>
						<do_else>
							<append_list_elements name="$sellwantedoffers" other="$homesselloffers"/>
						</do_else>
						<do_if value="$debugtrace">
							<do_if value="$wareprio.count gt 0">
								<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+' (Selling)'+': '+$homesselloffers.count+' total sell offers, '+$sellwantedoffers.count+' match the ware priority list.'" output="false" append="true" chance="100"/>
							</do_if>
							<do_else>
								<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+' (Selling)'+': '+$homesselloffers.count+' total sell offers, auto-managing all of them (no ware priority list set).'" output="false" append="true" chance="100"/>
							</do_else>
						</do_if>

						<do_if value="$sellwantedoffers.count gt 0">

							<do_if value="not $balanced">
								<do_if value="$wareprio.count gt 0">
									<!-- Priority mode with an explicit list: re-order the sell offers to match the player's top-to-bottom ware list. -->
									<create_list name="$sellprocessoffers"/>
									<do_all exact="$wareprio.count" counter="$sp3">
										<set_value name="$sellpriow" exact="$wareprio.{$sp3}"/>
										<do_all exact="$sellwantedoffers.count" counter="$so2">
											<do_if value="$sellwantedoffers.{$so2}.ware == $sellpriow">
												<append_to_list name="$sellprocessoffers" exact="$sellwantedoffers.{$so2}"/>
											</do_if>
										</do_all>
									</do_all>
								</do_if>
								<do_else>
									<!-- Priority mode, auto-detect: no player-specified order to follow, so process the most oversupplied ware (highest stocklevel) first. -->
									<set_value name="$sellprocessoffers" exact="$sellwantedoffers"/>
									<sort_list list="$sellprocessoffers" sortbyvalue="loop.element.stocklevel" sortdescending="true"/>
								</do_else>
							</do_if>
							<do_else>
								<!-- Balanced mode: keep the station's own order; each ware below gets an equal cargo-share cap instead of first-come-first-served. -->
								<set_value name="$sellprocessoffers" exact="$sellwantedoffers"/>
							</do_else>

							<do_if value="$fillcargo">
								<!-- Fill-cargo mode for selling: pick up every sellable ware from home first (tracking a running cargo-space budget across the whole pass - no credit budget needed, we're earning rather than spending), then export everything to its buyer in one batch at the end. -->
								<create_list name="$sellexportwares"/>
								<create_list name="$sellexportamounts"/>
								<create_list name="$sellexportbuyers"/>
								<create_list name="$sellexportbuyernames"/>
								<create_list name="$sellexportprices"/>
								<set_value name="$sellseedware" exact="$sellprocessoffers.{1}.ware"/>
								<set_value name="$sellremainingvolume" exact="this.ship.cargo.{$sellseedware}.free * $sellseedware.volume"/>
								<do_if value="$debugtrace">
									<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+' (Selling)'+': fill pass starting with '+$sellremainingvolume+'m3 free cargo, '+$sellprocessoffers.count+' sellable ware(s).'" output="false" append="true" chance="100"/>
								</do_if>

								<do_all exact="$sellprocessoffers.count" counter="$sw">
									<do_if value="(this.ship.tradeorders.count lt 6) and ($sellremainingvolume gt 0)">
										<set_value name="$sellwantedoffer" exact="$sellprocessoffers.{$sw}"/>
										<set_value name="$sellware" exact="$sellwantedoffer.ware"/>

										<!-- Price floor: never sell below what the home station itself is currently asking. -->
										<set_value name="$sellpricefloor" exact="$sellwantedoffer.unitprice/100"/>

										<!-- Search every sector in range for accessible external buyers of this ware. -->
										<create_list name="$sellfoundbuyers"/>
										<do_all exact="$searchspaces.count" counter="$ssp">
											<find_buy_offer tradepartner="this.ship" space="$searchspaces.{$ssp}" result="$sellbuyers" wares="$sellware" multiple="true">
												<match_buyer tradesknownto="this.owner">
													<match_relation_to object="this.ship" relation="dock" comparison="ge"/>
													<match_use_blacklist group="$blacklistgroup" type="blacklisttype.objectactivity" object="this.ship"/>
												</match_buyer>
											</find_buy_offer>
											<do_all exact="$sellbuyers.count" counter="$sbb">
												<append_to_list name="$sellfoundbuyers" exact="$sellbuyers.{$sbb}"/>
											</do_all>
											<set_value name="$scantick" exact="$scantick+1"/>
											<do_if value="$scantick gt $scantickrate">
												<set_value name="$scantick" exact="0"/>
												<wait exact="1ms"/>
											</do_if>
										</do_all>

										<!-- Pick the highest-paying offer at or above the price floor, excluding the home station buying from itself. -->
										<set_value name="$sellbestbuyer" exact="0"/>
										<set_value name="$sellbestprice" exact="$sellpricefloor"/>
										<set_value name="$sellpriciestseen" exact="0"/>
										<do_all exact="$sellfoundbuyers.count" counter="$sfb">
											<set_value name="$sellcandbuyer" exact="$sellfoundbuyers.{$sfb}"/>
											<do_if value="not ($sellcandbuyer.buyer == $homestation)">
												<set_value name="$sellcandprice" exact="$sellcandbuyer.unitprice/100"/>
												<do_if value="$debugtrace and (($sellpriciestseen == 0) or ($sellcandprice gt $sellpriciestseen))">
													<set_value name="$sellpriciestseen" exact="$sellcandprice"/>
												</do_if>
												<do_if value="$sellcandprice ge $sellbestprice">
													<set_value name="$sellbestbuyer" exact="$sellcandbuyer"/>
													<set_value name="$sellbestprice" exact="$sellcandprice"/>
												</do_if>
											</do_if>
										</do_all>

										<do_if value="not ($sellbestbuyer == 0)">
											<!-- Capture the buyer's name now, before create_trade_order consumes $sellbestbuyer below (reading $sellbestbuyer.buyer afterwards returns null). -->
											<set_value name="$sellbuyername" exact="$sellbestbuyer.buyer.knownname"/>
											<set_value name="$sellwareunitcap" exact="$sellremainingvolume/$sellware.volume"/>
											<do_if value="$balanced">
												<!-- Fair-share cap across whatever wares are still left to consider this pass. -->
												<set_value name="$sellremainingcount" exact="$sellprocessoffers.count-$sw+1"/>
												<set_value name="$sellwareunitcap" exact="[$sellwareunitcap/$sellremainingcount,1].max"/>
											</do_if>

											<set_value name="$sellamount" exact="[$sellwareunitcap,$sellbestbuyer.amount,$sellwantedoffer.amount].min"/>
											<do_if value="$debugtrace">
												<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$sellware+': picked buyer '+$sellbuyername+' at '+$sellbestprice+'cr/unit, amount='+$sellamount+' (wareunitcap='+$sellwareunitcap+', buyerwants='+$sellbestbuyer.amount+', homehas='+$sellwantedoffer.amount+')'" output="false" append="true" chance="100"/>
											</do_if>

											<do_if value="$sellamount gt 0">
												<!-- Pick-up leg now; the matching export/sell leg is queued after the whole hold is filled. -->
												<create_trade_order name="$sellwantedoffer" object="this.object" tradeoffer="$sellwantedoffer" amount="$sellamount" immediate="false"/>
												<append_to_list name="$sellexportwares" exact="$sellware"/>
												<append_to_list name="$sellexportamounts" exact="$sellamount"/>
												<append_to_list name="$sellexportbuyers" exact="$sellbestbuyer"/>
												<append_to_list name="$sellexportbuyernames" exact="$sellbuyername"/>
												<append_to_list name="$sellexportprices" exact="$sellbestprice"/>
												<set_value name="$sellremainingvolume" exact="$sellremainingvolume-($sellamount*$sellware.volume)"/>
											</do_if>
										</do_if>
										<do_elseif value="$debugtrace">
											<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$sellware+': no buyer at or above floor. Priciest seen was '+$sellpriciestseen+'cr/unit (floor was '+$sellpricefloor+').'" output="false" append="true" chance="100"/>
										</do_elseif>
										<remove_value name="$sellfoundbuyers"/>
									</do_if>

									<set_value name="$scantick" exact="$scantick+1"/>
									<do_if value="$scantick gt $scantickrate">
										<set_value name="$scantick" exact="0"/>
										<wait exact="1ms"/>
									</do_if>
								</do_all>

								<!-- Hold is as full as it's going to get this pass (or we ran out of sellable wares) - export everything to its buyer in one batch. -->
								<do_if value="$debugtrace">
									<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+' (Selling)'+': fill pass done, exporting '+$sellexportwares.count+' ware type(s) to their buyers ('+$sellremainingvolume+'m3 still free).'" output="false" append="true" chance="100"/>
								</do_if>
								<do_all exact="$sellexportwares.count" counter="$sd">
									<create_trade_order name="$sellexportbuyers.{$sd}" object="this.object" tradeoffer="$sellexportbuyers.{$sd}" amount="$sellexportamounts.{$sd}" immediate="false"/>
									<do_if value="$enablelogbook">
										<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$sellexportamounts.{$sd}*$sellexportprices.{$sd}" text="{8834271,302}.[$sellexportamounts.{$sd},$sellexportwares.{$sd},$sellexportbuyernames.{$sd},$sellexportprices.{$sd},$sellexportamounts.{$sd}*$sellexportprices.{$sd},$homestation.knownname]"/>
									</do_if>
								</do_all>
								<remove_value name="$sellexportwares"/>
								<remove_value name="$sellexportamounts"/>
								<remove_value name="$sellexportbuyers"/>
								<remove_value name="$sellexportbuyernames"/>
								<remove_value name="$sellexportprices"/>
							</do_if>
							<do_else>
								<!-- Classic mode: pick up and immediately sell one ware at a time. -->
								<do_all exact="$sellprocessoffers.count" counter="$sw2">
									<do_if value="this.ship.tradeorders.count lt 6">
										<set_value name="$sellwantedoffer" exact="$sellprocessoffers.{$sw2}"/>
										<set_value name="$sellware" exact="$sellwantedoffer.ware"/>

										<!-- Price floor: never sell below what the home station itself is currently asking. -->
										<set_value name="$sellpricefloor" exact="$sellwantedoffer.unitprice/100"/>
										<do_if value="$debugtrace">
											<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$sellware+': pricefloor='+$sellpricefloor" output="false" append="true" chance="100"/>
										</do_if>

										<!-- Search every sector in range for accessible external buyers of this ware. -->
										<create_list name="$sellfoundbuyers"/>
										<do_all exact="$searchspaces.count" counter="$ssp2">
											<find_buy_offer tradepartner="this.ship" space="$searchspaces.{$ssp2}" result="$sellbuyers2" wares="$sellware" multiple="true">
												<match_buyer tradesknownto="this.owner">
													<match_relation_to object="this.ship" relation="dock" comparison="ge"/>
													<match_use_blacklist group="$blacklistgroup" type="blacklisttype.objectactivity" object="this.ship"/>
												</match_buyer>
											</find_buy_offer>
											<do_all exact="$sellbuyers2.count" counter="$sbb2">
												<append_to_list name="$sellfoundbuyers" exact="$sellbuyers2.{$sbb2}"/>
											</do_all>
											<set_value name="$scantick" exact="$scantick+1"/>
											<do_if value="$scantick gt $scantickrate">
												<set_value name="$scantick" exact="0"/>
												<wait exact="1ms"/>
											</do_if>
										</do_all>
										<do_if value="$debugtrace">
											<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$sellware+': '+$sellfoundbuyers.count+' accessible buyer(s) found across '+$searchspaces.count+' sector(s).'" output="false" append="true" chance="100"/>
										</do_if>

										<!-- Pick the highest-paying offer at or above the price floor, excluding the home station buying from itself. -->
										<set_value name="$sellbestbuyer" exact="0"/>
										<set_value name="$sellbestprice" exact="$sellpricefloor"/>
										<set_value name="$sellpriciestseen" exact="0"/>
										<do_all exact="$sellfoundbuyers.count" counter="$sfb2">
											<set_value name="$sellcandbuyer" exact="$sellfoundbuyers.{$sfb2}"/>
											<do_if value="not ($sellcandbuyer.buyer == $homestation)">
												<set_value name="$sellcandprice" exact="$sellcandbuyer.unitprice/100"/>
												<do_if value="$debugtrace and (($sellpriciestseen == 0) or ($sellcandprice gt $sellpriciestseen))">
													<set_value name="$sellpriciestseen" exact="$sellcandprice"/>
												</do_if>
												<do_if value="$sellcandprice ge $sellbestprice">
													<set_value name="$sellbestbuyer" exact="$sellcandbuyer"/>
													<set_value name="$sellbestprice" exact="$sellcandprice"/>
												</do_if>
											</do_if>
										</do_all>
										<do_if value="$debugtrace">
											<do_if value="not ($sellbestbuyer == 0)">
												<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$sellware+': picked buyer '+$sellbestbuyer.buyer.knownname+' at '+$sellbestprice+'cr/unit.'" output="false" append="true" chance="100"/>
											</do_if>
											<do_else>
												<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$sellware+': no buyer at or above floor. Priciest seen was '+$sellpriciestseen+'cr/unit (floor was '+$sellpricefloor+').'" output="false" append="true" chance="100"/>
											</do_else>
										</do_if>

										<do_if value="not ($sellbestbuyer == 0)">
											<!-- Capture the buyer's name now, before create_trade_order consumes $sellbestbuyer below (reading $sellbestbuyer.buyer afterwards returns null). -->
											<set_value name="$sellbuyername" exact="$sellbestbuyer.buyer.knownname"/>
											<set_value name="$sellcargocap" exact="this.ship.cargo.{$sellware}.free"/>
											<do_if value="$balanced">
												<!-- Fair-share cap: don't let one ware use the whole hold this pass. -->
												<set_value name="$sellcargocap" exact="[$sellcargocap/$sellprocessoffers.count,1].max"/>
											</do_if>

											<set_value name="$sellamount" exact="[$sellcargocap,$sellbestbuyer.amount,$sellwantedoffer.amount].min"/>
											<do_if value="$debugtrace">
												<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$sellware+': amount='+$sellamount+' (cargocap='+$sellcargocap+', buyerwants='+$sellbestbuyer.amount+', homehas='+$sellwantedoffer.amount+')'" output="false" append="true" chance="100"/>
											</do_if>

											<do_if value="$sellamount gt 0">
												<!-- Pick-up leg: buy straight out of the home station's own sell offer. -->
												<create_trade_order name="$sellwantedoffer" object="this.object" tradeoffer="$sellwantedoffer" amount="$sellamount" immediate="false"/>
												<!-- Export leg: sell to the best accessible buyer. -->
												<create_trade_order name="$sellbestbuyer" object="this.object" tradeoffer="$sellbestbuyer" amount="$sellamount" immediate="false"/>
												<do_if value="$enablelogbook">
													<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$sellamount*$sellbestprice" text="{8834271,302}.[$sellamount,$sellware,$sellbuyername,$sellbestprice,$sellamount*$sellbestprice,$homestation.knownname]"/>
												</do_if>
											</do_if>
										</do_if>
										<remove_value name="$sellfoundbuyers"/>
									</do_if>
									<do_else>
										<do_if value="$debugtrace">
											<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  Skipping remaining sellable wares this pass - already have '+this.ship.tradeorders.count+' queued trade orders.'" output="false" append="true" chance="100"/>
										</do_if>
									</do_else>

									<set_value name="$scantick" exact="$scantick+1"/>
									<do_if value="$scantick gt $scantickrate">
										<set_value name="$scantick" exact="0"/>
										<wait exact="1ms"/>
									</do_if>
								</do_all>
							</do_else>
							<remove_value name="$sellprocessoffers"/>
						</do_if>
						<remove_value name="$homesselloffers"/>
						<remove_value name="$sellwantedoffers"/>
					</do_if>

					<do_if value="$enablebuying">
```

- [ ] **Step 2: Validate XML**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors. If unbalanced, check that every `do_if`/`do_all` opened inside the new block above has a matching close — the block is self-contained (opens and closes `$enableselling`'s `do_if` entirely within Step 1's replacement).

- [ ] **Step 3: Add the export Logbook text**

In `t/0001.xml`, find:

```xml
		<t id="301">Bought %1 %2 from %3 at %4cr/unit (%5cr total) to fill Build Storage at %6.</t>
```

Replace with:

```xml
		<t id="301">Bought %1 %2 from %3 at %4cr/unit (%5cr total) to fill Build Storage at %6.</t>
		<t id="302">Sold %1 %2 to %3 at %4cr/unit (%5cr total), exported from %6.</t>
```

- [ ] **Step 4: Validate the text file**

Run:
```bash
xmllint --noout t/0001.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [ ] **Step 5: Verify tag counts balance and the new text id is wired up**

```bash
grep -c '<do_if' aiscripts/sbe_stationtrader.xml
grep -c '</do_if>' aiscripts/sbe_stationtrader.xml
grep -c '<do_all' aiscripts/sbe_stationtrader.xml
grep -c '</do_all>' aiscripts/sbe_stationtrader.xml
grep -n '{8834271,302}' aiscripts/sbe_stationtrader.xml
grep -n '"302"' t/0001.xml
```
Expected: `do_if`/`</do_if>` counts equal each other; `do_all`/`</do_all>` counts equal each other; two matches for `{8834271,302}` (Fill Cargo path + classic path); one match for `<t id="302">`.

- [ ] **Step 6: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml t/0001.xml
git commit -m "feat: add sell pipeline for Enable Selling"
```

---

## Task 4: Update documentation

**Files:**
- Modify: `README.md` (top description, "What it does" list)
- Modify: `docs/CONFIGURATION.md` (parameter table rows, new explainer section, Fill Cargo section update, troubleshooting entry)

**Interfaces:**
- None (documentation only).

- [ ] **Step 1: Update the top description in README.md**

In `README.md`, find:

```markdown
A ship order that turns any trade ship into a dedicated restocker for one or
more home stations: it watches what the station currently wants to buy,
shops for the cheapest accessible seller within a configurable jump range,
and keeps ferrying goods in — respecting a priority or balanced buying
strategy and a price ceiling you control.
```

Replace with:

```markdown
A ship order that turns any trade ship into a dedicated logistics runner for
one or more home stations: it can buy the wares a station currently wants
from the cheapest accessible seller, sell the station's production surplus
to the best-paying accessible buyer, or both — respecting a priority or
balanced strategy and a price ceiling (buying) or floor (selling) you
control. Buying and selling are independently toggleable per ship via
**Enable Buying** and **Enable Selling** (both on by default).
```

- [ ] **Step 2: Rewrite the "What it does" list in README.md**

In `README.md`, find:

```markdown
## What it does

1. Assign the ship to one or more home stations.
2. It auto-detects what each station currently wants to buy — no manual
   ware list required, though you can optionally provide one to restrict
   or reorder what it manages.
3. It searches every accessible station within a configurable jump range for
   the cheapest seller of each wanted ware.
4. It buys in either strict top-to-bottom priority order or a balanced
   round-robin that keeps the wares roughly even.
5. By default it fills its cargo hold across every wanted ware before
   heading home, only returning short of a full hold when wares genuinely
   aren't available — one gathering trip instead of a round trip per ware
   (toggle this off for the classic immediate buy-then-deliver behavior).
6. It never buys above a price ceiling you set — either a flat max price, or
   a percentage below the home station's own price for that ware.
7. It respects your Empire > Blacklist entries — blacklisted sectors and
   factions/objects are excluded from the search, on top of the normal
   docking/relation and "offer known to your faction" checks.
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

Replace with:

```markdown
## What it does

1. Assign the ship to one or more home stations, and choose which roles are
   active for it — **Enable Buying**, **Enable Selling**, or both (both on
   by default).
2. With Enable Buying on, it auto-detects what each station currently wants
   to buy — no manual ware list required, though you can optionally provide
   one to restrict or reorder what it manages.
3. With Enable Selling on, it auto-detects what each station currently has
   for sale (its production surplus) — using the same Ware Priority List to
   restrict or reorder what it manages.
4. It searches every accessible station within a configurable jump range:
   for buying, the cheapest seller of each wanted ware; for selling, the
   best-paying buyer of each sellable ware.
5. It works in either strict top-to-bottom priority order or a balanced
   round-robin that keeps the wares roughly even — the same setting governs
   both roles.
6. By default it fills its cargo hold across every wanted/sellable ware
   before heading home or to a buyer, only returning short of a full hold
   when wares genuinely aren't available — one gathering (or exporting)
   trip instead of a round trip per ware (toggle this off for the classic
   immediate buy-then-deliver behavior).
7. When buying, it never pays above a price ceiling you set — either a flat
   max price, or a percentage below the home station's own price for that
   ware. When selling, it never accepts less than the home station's own
   current asking price for that ware — no separate config needed.
8. If both roles are on for the same ship, selling always runs before
   buying each pass, freeing up cargo space and generating income before
   the buy pass spends either.
9. It respects your Empire > Blacklist entries — blacklisted sectors and
   factions/objects are excluded from the search, on top of the normal
   docking/relation and "offer known to your faction" checks.
10. If the ship ends up idle holding cargo it can't otherwise account for
    — e.g. after complying with a pirate "drop cargo" demand mid-delivery —
    it tries to deliver it to a home station that wants it, or failing
    that, sell it to the best accessible buyer in range, rather than sit
    there holding it indefinitely.
11. It also resupplies a home station's Build Storage — construction-phase
    demand for a station under construction or having a module added —
    alongside its normal production buy offers, controlled by **Also
    Resupply Build Storage** and **Build Storage First**. Build Storage is
    only ever a buying concern; it never sells.
```

- [ ] **Step 3: Validate README rendering isn't broken (spot check)**

```bash
grep -c '^[0-9]\+\.' README.md
```
Expected: at least 11 (the renumbered list) — a sanity check that no list item was accidentally dropped or duplicated.

- [ ] **Step 4: Add Enable Buying / Enable Selling to the CONFIGURATION.md parameter table**

In `docs/CONFIGURATION.md`, find:

```markdown
| **Home Station(s)** | station list | empty | One or more stations this ship restocks. Pick multiple to have one ship service several stations in rotation. |
| **Ware Priority List** | ware list | empty | The wares this ship is allowed to manage, and (in Priority mode) the order to buy them in. **Leave empty to auto-manage every ware the station currently wants to buy** — see below. If you list specific wares, only those are ever bought, even if the station is short on something else. |
```

Replace with:

```markdown
| **Home Station(s)** | station list | empty | One or more stations this ship restocks. Pick multiple to have one ship service several stations in rotation. |
| **Enable Buying** | on/off | on | On = buy wares the station wants and deliver them home. See [Buying vs Selling](#buying-vs-selling-roles) below. |
| **Enable Selling** | on/off | on | On = sell the station's surplus wares (its active sell offers) to the best accessible buyer. See [Buying vs Selling](#buying-vs-selling-roles) below. |
| **Ware Priority List** | ware list | empty | The wares this ship is allowed to manage, and (in Priority mode) the order to handle them in — applies to both buying and selling. **Leave empty to auto-manage every ware the station currently wants to buy and/or sell** — see below. If you list specific wares, only those are ever bought/sold, even if the station has demand or surplus elsewhere. |
```

- [ ] **Step 5: Add the "Buying vs Selling roles" explainer section**

In `docs/CONFIGURATION.md`, find:

```markdown
## Auto-detect vs. an explicit ware list
```

Replace with:

```markdown
## Buying vs. Selling roles

**Enable Buying** and **Enable Selling** are independent on/off switches
(both on by default) — a ship can be buy-only, sell-only, or both. When
both are on for the same ship, **selling always runs before buying** in
every pass, for every home station: it's a fixed order, not configurable,
because freeing up cargo space and generating income before the buy pass
spends either is the sensible default. This has no effect if a station only
ever has demand in one direction.

Selling mirrors buying's mechanics exactly — same Ware Priority List, same
Balanced/Priority mode, same Fill Cargo Before Returning behavior — just in
reverse: it reads the station's own active **sell** offers instead of its
buy offers, searches for the best-paying accessible buyer instead of the
cheapest accessible seller, and picks up the ware from the home station
instead of delivering to it.

The one place selling is deliberately simpler than buying: there's no
Price Cap Mode / Min Discount % equivalent for selling. The floor is always
exactly the home station's own current sell-offer price for that ware — the
ship never accepts less than the station itself is already asking. If no
accessible buyer meets that floor, the ware is left for a future pass, same
as a buy-side ware with no seller under the ceiling.

Build Storage (see below) is a buying-only concept — it represents
construction demand, not surplus to sell, so **Enable Selling** never
touches it.

## Auto-detect vs. an explicit ware list
```

- [ ] **Step 6: Update the Fill Cargo Before Returning section to mention both roles**

In `docs/CONFIGURATION.md`, find:

```markdown
## Fill Cargo Before Returning vs. classic mode

**Fill Cargo Before Returning** (default, on): each pass, the ship buys from
every wanted ware it can find a valid deal for — tracking a running "how
much cargo space is left" and "how many credits are left" as it goes — and
only queues the delivery-to-home trade orders at the very end, once the
hold is as full as it's going to get this pass. It only "returns short"
(delivers less than a full hold) when there genuinely isn't enough
available: no more wares have a seller under the price ceiling, the credit
budget runs out, or the hold physically fills up. This means fewer separate
round trips overall — the ship gathers several wares in one outing before
heading home to unload, instead of shuttling back and forth for each ware
individually.

**Classic mode** (Fill Cargo Before Returning = off): the original
behavior — for each wanted ware, buy it and immediately queue the delivery
to home before considering the next ware. More round trips, but the home
station starts receiving partial deliveries sooner rather than waiting for
a full hold.
```

Replace with:

```markdown
## Fill Cargo Before Returning vs. classic mode

**Fill Cargo Before Returning** (default, on): each pass, the ship buys (or,
with Enable Selling on, picks up for export) from every wanted/sellable
ware it can find a valid deal for — tracking a running "how much cargo
space is left" (and, for buying, "how many credits are left") as it goes —
and only queues the delivery/export trade orders at the very end, once the
hold is as full as it's going to get this pass. It only "returns short"
(delivers/exports less than a full hold) when there genuinely isn't enough
available: no more wares have a seller under the price ceiling (or buyer
over the price floor), the credit budget runs out (buying only), or the
hold physically fills up. This means fewer separate round trips overall —
the ship gathers or exports several wares in one outing instead of
shuttling back and forth for each ware individually.

**Classic mode** (Fill Cargo Before Returning = off): the original
behavior — for each wanted/sellable ware, buy or sell it and immediately
queue the delivery/export before considering the next ware. More round
trips, but the home station's demand or surplus starts being worked sooner
rather than waiting for a full hold.
```

- [ ] **Step 7: Add a troubleshooting entry for selling**

In `docs/CONFIGURATION.md`, find:

```markdown
- **Ship isn't buying construction wares for a station under
  construction**: confirm **Also Resupply Build Storage** is on, and that
  the station is actually showing a "trade with build storage" option
  in-game right now (a station between construction phases can briefly
  have no active Build Storage). If it's genuinely under construction and
  still not being serviced, turn on Enable Debug Log — lines tagged
  `(Build Storage)` show what was found for that category each pass.
```

Replace with:

```markdown
- **Ship isn't buying construction wares for a station under
  construction**: confirm **Also Resupply Build Storage** is on, and that
  the station is actually showing a "trade with build storage" option
  in-game right now (a station between construction phases can briefly
  have no active Build Storage). If it's genuinely under construction and
  still not being serviced, turn on Enable Debug Log — lines tagged
  `(Build Storage)` show what was found for that category each pass.
- **Ship never exports anything**: confirm **Enable Selling** is on, and
  that the home station actually has an active sell offer for at least one
  ware right now (check the station's own Trade menu in-game — "selling"
  wares are shown there). If you populated Ware Priority List, confirm at
  least one of those wares is among what the station is currently selling.
  Also check Max Jump Range reaches a buyer your faction can dock at, and
  remember the price floor is always the home station's own current asking
  price — a buyer offering less than that is correctly skipped, not a bug.
  Turn on Enable Debug Log and look for lines tagged `(Selling)`.
```

- [ ] **Step 8: Commit**

```bash
git add README.md docs/CONFIGURATION.md
git commit -m "docs: document Enable Buying / Enable Selling roles"
```

---

## Task 5: In-game verification (manual — required before considering this feature done)

This mod has no automated test harness for aiscript behavior; the only way
to confirm the sell pipeline actually works is to run it inside X4 itself.
Unlike the Build Storage feature, this plan carries no unverified-syntax
risk (the shared-library idea was ruled out against the schema before any
code was written, and everything else — `find_sell_offer`,
`find_buy_offer`, `create_trade_order` — is already proven working
elsewhere in this same file). This task is a functional smoke test, not a
syntax-risk mitigation.

**Files:** none (manual QA against the installed extension).

- [ ] **Step 1: Deploy the updated mod**

```bash
rsync -a --delete --exclude '.git' \
  ~/Projects/X4StationTrader/ \
  "$HOME/.local/share/Steam/steamapps/common/X4 Foundations/extensions/sbe_station_trader/"
```

- [ ] **Step 2: Set up a test scenario in-game**

1. Load a save with an owned station that currently has both an active buy
   offer for at least one ware and an active sell offer for at least one
   (different) ware — most production stations qualify (e.g. a Refinery
   buying Ore/Silicon while selling refined metals).
2. Assign the Station Trader order to a ship, with that station as its Home
   Station. Leave **Enable Buying** and **Enable Selling** at their
   defaults (both on).
3. Set **Enable Debug Log** to on.

- [ ] **Step 3: Read the debug trace**

Debug output writes to
`~/.config/EgoSoft/X4/<save-id>/StationTrader/<ship idcode>.txt` (native
Linux build). Let the ship run a few passes, then check for lines
containing `(Selling)`:

```bash
tail -f "$HOME/.config/EgoSoft/X4"/*/StationTrader/*.txt | grep --line-buffered 'Selling'
```

Expected: a line like `<Station Name> (Selling): N total sell offers, ...`
appears, and — if an accessible buyer meeting the price floor exists — a
corresponding "picked buyer ... amount=..." line, followed by a trade order
actually appearing in the ship's order queue in-game.

- [ ] **Step 4: Confirm the buy/sell order**

With the station having both live buy and sell demand simultaneously,
confirm from the debug trace that the `(Selling)` lines for a given pass
appear before that pass's production/Build Storage buying lines — matching
the fixed "sell always runs first" rule. This should hold on every pass,
with no toggle to check (there is no `sellfirst` param).

- [ ] **Step 5: Confirm the price floor**

Note the home station's own current sell price for the exported ware
(visible in-game via its Trade menu) and confirm from the debug trace or
the completed trade order that the ship never accepted a buyer price below
that number. If **Enable Debug Log** shows a "no buyer at or above floor"
line, verify in the map/economy view that no accessible buyer is really
offering at or above that price right now — this is correct behavior, not
a bug.

- [ ] **Step 6: Confirm role toggles work independently**

Turn **Enable Selling** off (leave Enable Buying on) and confirm from the
debug trace that `(Selling)` lines stop appearing on subsequent passes
while buying continues normally. Then flip it the other way (Enable Buying
off, Enable Selling on) and confirm the reverse. Restore both to on when
done.

- [ ] **Step 7: Confirm the Logbook entry**

With **Enable Logbook Entries** on, complete at least one export delivery
and confirm its Logbook entry (Menu → Logbook) reads "Sold ... to ...
exported from `<station>`." — distinct from the existing "Bought ... to
restock/fill Build Storage at `<station>`." messages.

- [ ] **Step 8: If no `(Selling)` lines ever appear despite a station with a real sell offer**

This most likely means `find_sell_offer seller="$homestation"` isn't
returning what's expected in this context (it's already proven working
elsewhere in the file as a seller-side price-reference lookup, so this
would be a scoping/edge-case issue, not a fundamentally wrong tag). To
debug:

1. Confirm the station really has an active sell offer right now (check
   its own Trade menu in-game).
2. Add a temporary unconditional `debug_to_file` right after the
   `<find_sell_offer seller="$homestation" result="$homesselloffers"
   multiple="true"/>` line from Task 3 Step 1, logging
   `$homesselloffers.count`, to confirm whether the query itself returns
   zero results or whether a later filter (Ware Priority List, `$balanced`
   sort) is discarding them incorrectly.
3. Remove the temporary debug line once the root cause is found and fixed.
