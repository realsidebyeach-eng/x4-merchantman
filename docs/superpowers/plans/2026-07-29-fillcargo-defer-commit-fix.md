# Fill Cargo Defer-Commit Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix Fill Cargo mode (both buying and selling) so it can actually act on more than one ware per pass, by deferring every trade order's creation until after all searching for that pass is done.

**Architecture:** Both fill-cargo pipelines in `aiscripts/sbe_stationtrader.xml` currently create one leg of each ware's trade immediately inside the search loop, and defer only the other leg to a final batch. Evidence from every ship's debug log (763 buy-side passes, 524 sell-side, zero exceptions) shows this always caps at exactly one ware per pass, and the final batch step frequently never runs at all — consistent with the create_trade_order call handing control to the engine at the next `<wait>` yield point, abandoning the rest of the pass. The fix restructures each pipeline into two phases: a **decision phase** (unchanged search/pick/compute logic, but records the decision into a list instead of creating any trade order) and a **commit phase** (walks every recorded decision and creates every trade order for every ware, back-to-back, with zero `<wait>` calls anywhere in between).

**Tech Stack:** X4: Foundations `aiscript` XML. Verification is `xmllint --noout` plus schema validation against the local `aiscripts.xsd` cache (`/home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/aiscripts.xsd`) — both already confirmed to pass for this exact restructuring (both pipelines, spliced together) before this plan was written.

## Global Constraints

- Every task ends with `xmllint --noout aiscripts/sbe_stationtrader.xml` passing.
- Bump the aiscript `version` attribute (currently 18, so 19) — a single bump covering both pipelines' changes.
- Classic mode (both buy and sell) is untouched — it already only handles one ware per pass by design and has no batching premise to protect.
- The stray-cargo fallbacks, price-floor/ceiling decision logic, and storage-capacity checks (`$querybuyer.cargo.{...}.free`, version-18) are reused as-is — only *when* trade orders get created changes, not which ware/partner gets picked, at what price, or how amounts are capped.
- No new param, no player-facing toggle.
- Each decided ware now produces 2 trade orders (buy+deliver, or pickup+export) instead of the old code's implicit 1-at-a-time creation. The decision-phase loop guard must account for this so the ship's total order count (existing orders at pass start, plus 2 per decided ware) never *exceeds* 6 — capture `this.ship.tradeorders.count` once before the decision loop starts (`$startingorders` / `$sellstartingorders`), then gate each further decision on `(($startingorders+($decidedwares.count*2)) le 4)` (buy) / `(($sellstartingorders+($selldecidedwares.count*2)) le 4)` (sell) instead of re-reading `this.ship.tradeorders.count` directly (which would stay 0 throughout the whole decision phase, since no orders exist yet to count, and would no longer cap anything). Correction from final review: an initial `lt 6` version of this formula only correctly capped the ceiling for even starting counts — for an odd `$startingorders` it could let the ship reach 7 total orders. `le 4` (permit one more decision only if the current running total, `$startingorders+($decided*.count*2)`, is at most 4 — i.e. one more pair would bring it to at most 6) holds for every starting count, not just zero.
- This is X4: Foundations aiscript XML — no unit-test framework exists; xmllint plus schema validation is the verification gate for the code tasks. In-game verification (Task 3) is load-bearing, not a formality — this fix rests on an unproven hypothesis about engine behavior, and the whole point of testing is to find out whether it's right.

---

## Task 1: Restructure the buy fill-cargo pipeline into decision + commit phases

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml`

**Interfaces:**
- Produces: `$decidedwares`, `$decidedwarenames`, `$decidedamounts`, `$decidedsellers`, `$decidedsellernames`, `$decidedprices`, `$startingorders` — all scoped locally to this one `do_if value="$fillcargo"` block (buy pipeline), not consumed by Task 2 or by classic mode. `$decidedwarenames` captures the ware macro itself at decision time, separately from the offer object, so the commit phase's second loop never has to read `.ware` off an offer the first loop already consumed via `create_trade_order`.

- [x] **Step 1: Bump the aiscript version**

In `aiscripts/sbe_stationtrader.xml`, find:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="18">
```

Replace with:

```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="19">
```

- [x] **Step 2: Replace the entire buy fill-cargo block**

In `aiscripts/sbe_stationtrader.xml`, find this exact block (starts right after the `$processoffers`/balanced-mode setup, ends right before `<do_else>` which begins classic mode):

```xml
						<do_if value="$fillcargo">
							<!-- Fill-cargo mode: buy from every wanted ware first (tracking a running cargo-space and credit budget across the whole pass), then deliver everything home in one batch at the end. Only stops early if the hold fills up, the budget runs out, or no wanted ware has a valid seller left - i.e. only "returns short" when wares genuinely aren't available. -->
							<create_list name="$deliverwares"/>
							<create_list name="$deliveramounts"/>
							<create_list name="$deliversellers"/>
							<create_list name="$deliverprices"/>
							<set_value name="$seedware" exact="$processoffers.{1}.ware"/>
							<set_value name="$remainingvolume" exact="this.ship.cargo.{$seedware}.free * $seedware.volume"/>
							<set_value name="$remainingbudget" exact="player.money/100"/>
							<do_if value="$debugtrace">
								<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+$categorylabel+': fill pass starting with '+$remainingvolume+'m3 free cargo, '+$remainingbudget+'cr spendable, '+$processoffers.count+' wanted ware(s).'" output="false" append="true" chance="100"/>
							</do_if>

							<do_all exact="$processoffers.count" counter="$w">
								<do_if value="(this.ship.tradeorders.count lt 6) and ($remainingvolume gt 0) and ($remainingbudget gt 0)">
									<set_value name="$wantedoffer" exact="$processoffers.{$w}"/>
									<set_value name="$ware" exact="$wantedoffer.ware"/>

									<!-- Work out this ware's price ceiling: reject anything pricier (not discounted enough to be worth the trip). -->
									<set_value name="$priceceiling" exact="$maxprice"/>
									<do_if value="$usepercentcap">
										<find_sell_offer seller="$homestation" result="$homesell" wares="$ware" multiple="true"/>
										<do_if value="$homesell.count gt 0">
											<set_value name="$refprice" exact="$homesell.{1}.unitprice/100"/>
										</do_if>
										<do_else>
											<!-- Home station has no active sell offer for this ware right now; fall back to what it is currently offering to pay for it. -->
											<set_value name="$refprice" exact="$wantedoffer.unitprice/100"/>
										</do_else>
										<set_value name="$priceceiling" exact="$refprice*(100-$minpercent)/100"/>
										<remove_value name="$homesell"/>
									</do_if>

									<!-- Search every sector in range for accessible sellers of this ware. -->
									<create_list name="$foundsellers"/>
									<do_all exact="$searchspaces.count" counter="$sp">
										<find_sell_offer tradepartner="this.ship" space="$searchspaces.{$sp}" result="$sellers" wares="$ware" multiple="true">
											<match_seller tradesknownto="this.owner">
												<match_relation_to object="this.ship" relation="dock" comparison="ge"/>
												<match_use_blacklist group="$blacklistgroup" type="blacklisttype.objectactivity" object="this.ship"/>
											</match_seller>
										</find_sell_offer>
										<do_all exact="$sellers.count" counter="$se">
											<append_to_list name="$foundsellers" exact="$sellers.{$se}"/>
										</do_all>
										<set_value name="$scantick" exact="$scantick+1"/>
										<do_if value="$scantick gt $scantickrate">
											<set_value name="$scantick" exact="0"/>
											<wait exact="1ms"/>
										</do_if>
									</do_all>

									<!-- Pick the cheapest offer at or below the price ceiling, excluding the home station selling to itself. -->
									<set_value name="$bestseller" exact="0"/>
									<set_value name="$bestprice" exact="$priceceiling"/>
									<set_value name="$cheapestseen" exact="0"/>
									<do_all exact="$foundsellers.count" counter="$fs">
										<set_value name="$candoffer" exact="$foundsellers.{$fs}"/>
										<do_if value="not ($candoffer.seller == $homestation)">
											<set_value name="$candprice" exact="$candoffer.unitprice/100"/>
											<do_if value="$debugtrace and (($cheapestseen == 0) or ($candprice lt $cheapestseen))">
												<set_value name="$cheapestseen" exact="$candprice"/>
											</do_if>
											<do_if value="$candprice le $bestprice">
												<set_value name="$bestseller" exact="$candoffer"/>
												<set_value name="$bestprice" exact="$candprice"/>
											</do_if>
										</do_if>
									</do_all>

									<do_if value="not ($bestseller == 0)">
										<!-- Capture the seller's name now, before create_trade_order consumes $bestseller below (reading $bestseller.seller afterwards returns null). -->
										<set_value name="$sellername" exact="$bestseller.seller.knownname"/>
										<set_value name="$wareunitcap" exact="$remainingvolume/$ware.volume"/>
										<do_if value="$balanced">
											<!-- Fair-share cap across whatever wares are still left to consider this pass. -->
											<set_value name="$remainingcount" exact="$processoffers.count-$w+1"/>
											<set_value name="$wareunitcap" exact="[$wareunitcap/$remainingcount,1].max"/>
										</do_if>

										<!-- $wantedoffer.amount is a demand target, not live storage room. $querybuyer.cargo.{$ware}.free is the actual buyer's (base station, or its Build Storage) real current free space for this ware - storage only, never money, since the home-delivery leg below always executes at price="0" and must never be limited by the station's account balance. -->
										<set_value name="$amount" exact="[$wareunitcap,$bestseller.amount,$wantedoffer.amount,$remainingbudget/$bestprice,$querybuyer.cargo.{$ware}.free].min"/>
										<do_if value="$debugtrace">
											<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$ware+': picked '+$sellername+' at '+$bestprice+'cr/unit, amount='+$amount+' (wareunitcap='+$wareunitcap+', sellerhas='+$bestseller.amount+', homewants='+$wantedoffer.amount+', afford='+($remainingbudget/$bestprice)+')'" output="false" append="true" chance="100"/>
										</do_if>

										<do_if value="$amount gt 0">
											<!-- Buy leg now; the matching sell/deliver leg is queued after the whole hold is filled. -->
											<create_trade_order name="$bestseller" object="this.object" tradeoffer="$bestseller" amount="$amount" immediate="false"/>
											<append_to_list name="$deliverwares" exact="$wantedoffer"/>
											<append_to_list name="$deliveramounts" exact="$amount"/>
											<append_to_list name="$deliversellers" exact="$sellername"/>
											<append_to_list name="$deliverprices" exact="$bestprice"/>
											<set_value name="$remainingvolume" exact="$remainingvolume-($amount*$ware.volume)"/>
											<set_value name="$remainingbudget" exact="$remainingbudget-($amount*$bestprice)"/>
										</do_if>
									</do_if>
									<do_elseif value="$debugtrace">
										<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$ware+': no seller under ceiling. Cheapest seen was '+$cheapestseen+'cr/unit (ceiling was '+$priceceiling+').'" output="false" append="true" chance="100"/>
									</do_elseif>
									<remove_value name="$foundsellers"/>
								</do_if>

								<set_value name="$scantick" exact="$scantick+1"/>
								<do_if value="$scantick gt $scantickrate">
									<set_value name="$scantick" exact="0"/>
									<wait exact="1ms"/>
								</do_if>
							</do_all>

							<!-- Hold is as full as it's going to get this pass (or we ran out of wares/budget) - deliver everything bought in one batch. -->
							<do_if value="$debugtrace">
								<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+$categorylabel+': fill pass done, delivering '+$deliverwares.count+' ware type(s) home in one trip ('+$remainingvolume+'m3 still free).'" output="false" append="true" chance="100"/>
							</do_if>
							<do_all exact="$deliverwares.count" counter="$d">
								<!-- Re-check live storage room right before delivering: an earlier ware in this same batch may have already used up shared capacity. Storage only, never money - this leg always executes at price="0". -->
								<set_value name="$deliverclamped" exact="[$deliveramounts.{$d},$querybuyer.cargo.{$deliverwares.{$d}.ware}.free].min"/>
								<create_trade_order name="$deliverwares.{$d}" object="this.object" tradeoffer="$deliverwares.{$d}" amount="$deliverclamped" price="0" immediate="false"/>
								<do_if value="$enablelogbook">
									<do_if value="$isbuildstorage">
										<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$deliveramounts.{$d}*$deliverprices.{$d}" text="{8834271,301}.[$deliveramounts.{$d},$deliverwares.{$d}.ware,$deliversellers.{$d},$deliverprices.{$d},$deliveramounts.{$d}*$deliverprices.{$d},$homestation.knownname]"/>
									</do_if>
									<do_else>
										<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$deliveramounts.{$d}*$deliverprices.{$d}" text="{8834271,300}.[$deliveramounts.{$d},$deliverwares.{$d}.ware,$deliversellers.{$d},$deliverprices.{$d},$deliveramounts.{$d}*$deliverprices.{$d},$homestation.knownname]"/>
									</do_else>
								</do_if>
							</do_all>
							<remove_value name="$deliverwares"/>
							<remove_value name="$deliveramounts"/>
							<remove_value name="$deliversellers"/>
							<remove_value name="$deliverprices"/>
						</do_if>
```

Replace with:

```xml
						<do_if value="$fillcargo">
							<!-- Fill-cargo mode: decide every wanted ware first (search freely, no trade orders yet), then create every trade order back-to-back at the end with no searching in between - creating a trade order appears to let the game engine take over the ship at the next search wait, so nothing after that point in the same pass would otherwise run. -->
							<create_list name="$decidedwares"/>
							<create_list name="$decidedwarenames"/>
							<create_list name="$decidedamounts"/>
							<create_list name="$decidedsellers"/>
							<create_list name="$decidedsellernames"/>
							<create_list name="$decidedprices"/>
							<set_value name="$seedware" exact="$processoffers.{1}.ware"/>
							<set_value name="$remainingvolume" exact="this.ship.cargo.{$seedware}.free * $seedware.volume"/>
							<set_value name="$remainingbudget" exact="player.money/100"/>
							<set_value name="$startingorders" exact="this.ship.tradeorders.count"/>
							<do_if value="$debugtrace">
								<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+$categorylabel+': fill pass starting with '+$remainingvolume+'m3 free cargo, '+$remainingbudget+'cr spendable, '+$processoffers.count+' wanted ware(s).'" output="false" append="true" chance="100"/>
							</do_if>

							<do_all exact="$processoffers.count" counter="$w">
								<do_if value="(($startingorders+($decidedwares.count*2)) le 4) and ($remainingvolume gt 0) and ($remainingbudget gt 0)">
									<set_value name="$wantedoffer" exact="$processoffers.{$w}"/>
									<set_value name="$ware" exact="$wantedoffer.ware"/>

									<!-- Work out this ware's price ceiling: reject anything pricier (not discounted enough to be worth the trip). -->
									<set_value name="$priceceiling" exact="$maxprice"/>
									<do_if value="$usepercentcap">
										<find_sell_offer seller="$homestation" result="$homesell" wares="$ware" multiple="true"/>
										<do_if value="$homesell.count gt 0">
											<set_value name="$refprice" exact="$homesell.{1}.unitprice/100"/>
										</do_if>
										<do_else>
											<!-- Home station has no active sell offer for this ware right now; fall back to what it is currently offering to pay for it. -->
											<set_value name="$refprice" exact="$wantedoffer.unitprice/100"/>
										</do_else>
										<set_value name="$priceceiling" exact="$refprice*(100-$minpercent)/100"/>
										<remove_value name="$homesell"/>
									</do_if>

									<!-- Search every sector in range for accessible sellers of this ware. -->
									<create_list name="$foundsellers"/>
									<do_all exact="$searchspaces.count" counter="$sp">
										<find_sell_offer tradepartner="this.ship" space="$searchspaces.{$sp}" result="$sellers" wares="$ware" multiple="true">
											<match_seller tradesknownto="this.owner">
												<match_relation_to object="this.ship" relation="dock" comparison="ge"/>
												<match_use_blacklist group="$blacklistgroup" type="blacklisttype.objectactivity" object="this.ship"/>
											</match_seller>
										</find_sell_offer>
										<do_all exact="$sellers.count" counter="$se">
											<append_to_list name="$foundsellers" exact="$sellers.{$se}"/>
										</do_all>
										<set_value name="$scantick" exact="$scantick+1"/>
										<do_if value="$scantick gt $scantickrate">
											<set_value name="$scantick" exact="0"/>
											<wait exact="1ms"/>
										</do_if>
									</do_all>

									<!-- Pick the cheapest offer at or below the price ceiling, excluding the home station selling to itself. -->
									<set_value name="$bestseller" exact="0"/>
									<set_value name="$bestprice" exact="$priceceiling"/>
									<set_value name="$cheapestseen" exact="0"/>
									<do_all exact="$foundsellers.count" counter="$fs">
										<set_value name="$candoffer" exact="$foundsellers.{$fs}"/>
										<do_if value="not ($candoffer.seller == $homestation)">
											<set_value name="$candprice" exact="$candoffer.unitprice/100"/>
											<do_if value="$debugtrace and (($cheapestseen == 0) or ($candprice lt $cheapestseen))">
												<set_value name="$cheapestseen" exact="$candprice"/>
											</do_if>
											<do_if value="$candprice le $bestprice">
												<set_value name="$bestseller" exact="$candoffer"/>
												<set_value name="$bestprice" exact="$candprice"/>
											</do_if>
										</do_if>
									</do_all>

									<do_if value="not ($bestseller == 0)">
										<!-- Capture the seller's name now, before the commit phase's create_trade_order consumes $bestseller (reading $bestseller.seller afterwards returns null). -->
										<set_value name="$sellername" exact="$bestseller.seller.knownname"/>
										<set_value name="$wareunitcap" exact="$remainingvolume/$ware.volume"/>
										<do_if value="$balanced">
											<!-- Fair-share cap across whatever wares are still left to consider this pass. -->
											<set_value name="$remainingcount" exact="$processoffers.count-$w+1"/>
											<set_value name="$wareunitcap" exact="[$wareunitcap/$remainingcount,1].max"/>
										</do_if>

										<!-- $wantedoffer.amount is a demand target, not live storage room. $querybuyer.cargo.{$ware}.free is the actual buyer's (base station, or its Build Storage) real current free space for this ware - storage only, never money, since the home-delivery leg always executes at price="0" and must never be limited by the station's account balance. -->
										<set_value name="$amount" exact="[$wareunitcap,$bestseller.amount,$wantedoffer.amount,$remainingbudget/$bestprice,$querybuyer.cargo.{$ware}.free].min"/>
										<do_if value="$debugtrace">
											<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$ware+': picked '+$sellername+' at '+$bestprice+'cr/unit, amount='+$amount+' (wareunitcap='+$wareunitcap+', sellerhas='+$bestseller.amount+', homewants='+$wantedoffer.amount+', afford='+($remainingbudget/$bestprice)+')'" output="false" append="true" chance="100"/>
										</do_if>

										<do_if value="$amount gt 0">
											<!-- Record the decision only - every trade order for this pass is created together in the commit phase below, after all searching is done. -->
											<append_to_list name="$decidedwares" exact="$wantedoffer"/>
											<append_to_list name="$decidedwarenames" exact="$ware"/>
											<append_to_list name="$decidedamounts" exact="$amount"/>
											<append_to_list name="$decidedsellers" exact="$bestseller"/>
											<append_to_list name="$decidedsellernames" exact="$sellername"/>
											<append_to_list name="$decidedprices" exact="$bestprice"/>
											<set_value name="$remainingvolume" exact="$remainingvolume-($amount*$ware.volume)"/>
											<set_value name="$remainingbudget" exact="$remainingbudget-($amount*$bestprice)"/>
										</do_if>
									</do_if>
									<do_elseif value="$debugtrace">
										<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  '+$ware+': no seller under ceiling. Cheapest seen was '+$cheapestseen+'cr/unit (ceiling was '+$priceceiling+').'" output="false" append="true" chance="100"/>
									</do_elseif>
									<remove_value name="$foundsellers"/>
								</do_if>

								<set_value name="$scantick" exact="$scantick+1"/>
								<do_if value="$scantick gt $scantickrate">
									<set_value name="$scantick" exact="0"/>
									<wait exact="1ms"/>
								</do_if>
							</do_all>

							<!-- Every ware has been decided - now create every trade order back-to-back with no searching (and so no wait) in between, so a queued order can't cut this pass short before the rest are created. Buy legs first, then deliver legs, so the ship visits every seller before returning home once - pairing each ware's two legs together here would turn this back into one round trip per ware. -->
							<do_if value="$debugtrace">
								<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+$categorylabel+': fill pass done deciding, committing '+$decidedwares.count+' ware type(s) home in one trip ('+$remainingvolume+'m3 still free).'" output="false" append="true" chance="100"/>
							</do_if>
							<do_all exact="$decidedwares.count" counter="$d">
								<create_trade_order name="$decidedsellers.{$d}" object="this.object" tradeoffer="$decidedsellers.{$d}" amount="$decidedamounts.{$d}" immediate="false"/>
							</do_all>
							<do_all exact="$decidedwares.count" counter="$d2">
								<!-- Clamp against the buyer's live storage room for this ware. Nothing executes between the queued orders in this loop, so this reads the same value for every ware in this batch - it does NOT yet account for two decided wares sharing the same storage pool within one pass (tracked as a known follow-up). Storage only, never money - this leg always executes at price="0". -->
								<set_value name="$deliverclamped" exact="[$decidedamounts.{$d2},$querybuyer.cargo.{$decidedwarenames.{$d2}}.free].min"/>
								<create_trade_order name="$decidedwares.{$d2}" object="this.object" tradeoffer="$decidedwares.{$d2}" amount="$deliverclamped" price="0" immediate="false"/>
								<do_if value="$enablelogbook">
									<do_if value="$isbuildstorage">
										<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$decidedamounts.{$d2}*$decidedprices.{$d2}" text="{8834271,301}.[$decidedamounts.{$d2},$decidedwarenames.{$d2},$decidedsellernames.{$d2},$decidedprices.{$d2},$decidedamounts.{$d2}*$decidedprices.{$d2},$homestation.knownname]"/>
									</do_if>
									<do_else>
										<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$decidedamounts.{$d2}*$decidedprices.{$d2}" text="{8834271,300}.[$decidedamounts.{$d2},$decidedwarenames.{$d2},$decidedsellernames.{$d2},$decidedprices.{$d2},$decidedamounts.{$d2}*$decidedprices.{$d2},$homestation.knownname]"/>
									</do_else>
								</do_if>
							</do_all>
							<do_if value="$debugtrace">
								<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  commit done: ship now has '+this.ship.tradeorders.count+' trade order(s) (expected '+($startingorders+($decidedwares.count*2))+').'" output="false" append="true" chance="100"/>
							</do_if>
							<remove_value name="$decidedwares"/>
							<remove_value name="$decidedwarenames"/>
							<remove_value name="$decidedamounts"/>
							<remove_value name="$decidedsellers"/>
							<remove_value name="$decidedsellernames"/>
							<remove_value name="$decidedprices"/>
						</do_if>
```

- [x] **Step 3: Validate XML well-formedness**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [x] **Step 4: Validate against the real X4 aiscript schema**

Run:
```bash
xmllint --noout --schema /home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/aiscripts.xsd aiscripts/sbe_stationtrader.xml && echo SCHEMA_VALID
```
Expected: `SCHEMA_VALID` printed, no errors. This restructuring changes deep nesting in a large block — schema validation (not just well-formedness) is the real gate here, not optional.

- [x] **Step 5: Verify no dangling references to the old list names**

```bash
grep -c '\$deliverwares\|\$deliveramounts\|\$deliversellers' aiscripts/sbe_stationtrader.xml
```
Expected: `0` — the old buy-side batch lists (`$deliverwares`, `$deliveramounts`, `$deliversellers`) no longer exist anywhere in the file. (Note: `$deliverclamped` is a different, still-used variable name — it is not part of this grep pattern and should still appear once in the new commit-phase code.)

- [x] **Step 6: Verify the new decision/commit structure is in place**

```bash
grep -c '\$decidedwares\|\$decidedamounts\|\$decidedsellers\|\$decidedsellernames\|\$decidedprices' aiscripts/sbe_stationtrader.xml
grep -n 'startingorders' aiscripts/sbe_stationtrader.xml
```
Expected: the first command shows a positive count (each name appears multiple times: `create_list`, at least one `append_to_list`/read, and `remove_value`). The second command shows two matches — `$startingorders` (buy pipeline, set once and used in the decision-loop guard) — Task 2 will add the sell-side `$sellstartingorders` separately, so only the buy-side name should appear after this task.

- [x] **Step 7: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml
git commit -m "fix: defer buy fill-cargo trade orders to a single no-wait commit phase"
```

---

## Task 2: Restructure the sell fill-cargo pipeline into decision + commit phases

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml`

**Interfaces:**
- Produces: `$selldecidedwares`, `$selldecidedwarenames`, `$selldecidedamounts`, `$selldecidedbuyers`, `$selldecidedbuyernames`, `$selldecidedprices`, `$sellstartingorders` — all scoped locally to this one `do_if value="$fillcargo"` block (sell pipeline), not consumed by Task 1 or by classic mode. `$selldecidedwarenames` captures the ware macro itself at decision time, separately from the offer object, so the commit phase's second loop never has to read `.ware` off an offer the first loop already consumed via `create_trade_order`.

- [x] **Step 1: Replace the entire sell fill-cargo block**

In `aiscripts/sbe_stationtrader.xml`, find this exact block (starts right after the Selling section's `$sellprocessoffers`/balanced-mode setup, ends right before `<do_else>` which begins Selling's classic mode):

```xml
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
```

Replace with:

```xml
							<do_if value="$fillcargo">
								<!-- Fill-cargo mode for selling: decide every sellable ware first (search freely, no trade orders yet), then create every trade order back-to-back at the end with no searching in between - creating a trade order appears to let the game engine take over the ship at the next search wait, so nothing after that point in the same pass would otherwise run. -->
								<create_list name="$selldecidedwares"/>
								<create_list name="$selldecidedwarenames"/>
								<create_list name="$selldecidedamounts"/>
								<create_list name="$selldecidedbuyers"/>
								<create_list name="$selldecidedbuyernames"/>
								<create_list name="$selldecidedprices"/>
								<set_value name="$sellseedware" exact="$sellprocessoffers.{1}.ware"/>
								<set_value name="$sellremainingvolume" exact="this.ship.cargo.{$sellseedware}.free * $sellseedware.volume"/>
								<set_value name="$sellstartingorders" exact="this.ship.tradeorders.count"/>
								<do_if value="$debugtrace">
									<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+' (Selling)'+': fill pass starting with '+$sellremainingvolume+'m3 free cargo, '+$sellprocessoffers.count+' sellable ware(s).'" output="false" append="true" chance="100"/>
								</do_if>

								<do_all exact="$sellprocessoffers.count" counter="$sw">
									<do_if value="(($sellstartingorders+($selldecidedwares.count*2)) le 4) and ($sellremainingvolume gt 0)">
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
											<!-- Capture the buyer's name now, before the commit phase's create_trade_order consumes $sellbestbuyer (reading $sellbestbuyer.buyer afterwards returns null). -->
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
												<!-- Record the decision only - every trade order for this pass is created together in the commit phase below, after all searching is done. -->
												<append_to_list name="$selldecidedwares" exact="$sellwantedoffer"/>
												<append_to_list name="$selldecidedwarenames" exact="$sellware"/>
												<append_to_list name="$selldecidedamounts" exact="$sellamount"/>
												<append_to_list name="$selldecidedbuyers" exact="$sellbestbuyer"/>
												<append_to_list name="$selldecidedbuyernames" exact="$sellbuyername"/>
												<append_to_list name="$selldecidedprices" exact="$sellbestprice"/>
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

								<!-- Every ware has been decided - now create every trade order back-to-back with no searching (and so no wait) in between, so a queued order can't cut this pass short before the rest are created. Pickup legs first, then export legs, so the ship visits every buyer before returning home once - pairing each ware's two legs together here would turn this back into one round trip per ware. -->
								<do_if value="$debugtrace">
									<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="$homestation.knownname+' (Selling)'+': fill pass done deciding, committing '+$selldecidedwares.count+' ware type(s) to their buyers ('+$sellremainingvolume+'m3 still free).'" output="false" append="true" chance="100"/>
								</do_if>
								<do_all exact="$selldecidedwares.count" counter="$sd">
									<create_trade_order name="$selldecidedwares.{$sd}" object="this.object" tradeoffer="$selldecidedwares.{$sd}" amount="$selldecidedamounts.{$sd}" immediate="false"/>
								</do_all>
								<do_all exact="$selldecidedwares.count" counter="$sd2">
									<create_trade_order name="$selldecidedbuyers.{$sd2}" object="this.object" tradeoffer="$selldecidedbuyers.{$sd2}" amount="$selldecidedamounts.{$sd2}" immediate="false"/>
									<do_if value="$enablelogbook">
										<write_to_logbook category="upkeep" title="'Station Trader: '+this.ship.knownname+' ( '+this.ship.idcode+' )'" interaction="showonmap" object="this.ship" money="$selldecidedamounts.{$sd2}*$selldecidedprices.{$sd2}" text="{8834271,302}.[$selldecidedamounts.{$sd2},$selldecidedwarenames.{$sd2},$selldecidedbuyernames.{$sd2},$selldecidedprices.{$sd2},$selldecidedamounts.{$sd2}*$selldecidedprices.{$sd2},$homestation.knownname]"/>
									</do_if>
								</do_all>
								<do_if value="$debugtrace">
									<debug_to_file name="this.ship.idcode" directory="'StationTrader'" text="'  commit done: ship now has '+this.ship.tradeorders.count+' trade order(s) (expected '+($sellstartingorders+($selldecidedwares.count*2))+').'" output="false" append="true" chance="100"/>
								</do_if>
								<remove_value name="$selldecidedwares"/>
								<remove_value name="$selldecidedwarenames"/>
								<remove_value name="$selldecidedamounts"/>
								<remove_value name="$selldecidedbuyers"/>
								<remove_value name="$selldecidedbuyernames"/>
								<remove_value name="$selldecidedprices"/>
							</do_if>
```

- [x] **Step 2: Validate XML well-formedness**

Run:
```bash
xmllint --noout aiscripts/sbe_stationtrader.xml && echo VALID
```
Expected: `VALID` printed, no errors.

- [x] **Step 3: Validate against the real X4 aiscript schema**

Run:
```bash
xmllint --noout --schema /home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/aiscripts.xsd aiscripts/sbe_stationtrader.xml && echo SCHEMA_VALID
```
Expected: `SCHEMA_VALID` printed, no errors.

- [x] **Step 4: Verify no dangling references to the old list names**

```bash
grep -c '\$sellexportwares\|\$sellexportamounts\|\$sellexportbuyers\|\$sellexportbuyernames\|\$sellexportprices' aiscripts/sbe_stationtrader.xml
```
Expected: `0` — the old sell-side batch lists no longer exist anywhere in the file.

- [x] **Step 5: Verify the new decision/commit structure is in place**

```bash
grep -c '\$selldecidedwares\|\$selldecidedamounts\|\$selldecidedbuyers\|\$selldecidedbuyernames\|\$selldecidedprices' aiscripts/sbe_stationtrader.xml
grep -n 'startingorders' aiscripts/sbe_stationtrader.xml
```
Expected: the first command shows a positive count. The second command now shows two matches total across the whole file — `$startingorders` (buy, from Task 1) and `$sellstartingorders` (sell, from this task) — confirming both pipelines' guards are in place and distinctly named.

- [x] **Step 6: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml
git commit -m "fix: defer sell fill-cargo trade orders to a single no-wait commit phase"
```

---

## Task 3: In-game verification (manual — required before considering this fix done)

This mod has no automated test harness for aiscript behavior, and this fix
rests on an unproven hypothesis about engine behavior (see the design
spec, `docs/superpowers/specs/2026-07-29-fillcargo-defer-commit-design.md`,
for the full evidence trail). This task is the actual test of that
hypothesis, not a formality — it determines whether this fix worked or
whether the fallback approach (make Fill Cargo behave like Classic mode)
needs to be pursued instead.

**Files:** none (manual QA against the installed extension).

- [ ] **Step 1: Deploy the updated mod**

```bash
rsync -a --delete --exclude '.git' --exclude '.claude' \
  ~/Projects/X4StationTrader/ \
  "$HOME/.local/share/Steam/steamapps/common/X4 Foundations/extensions/sbe_station_trader/"
```

- [ ] **Step 2: Watch a fill-cargo ship's debug log for a multi-ware pass**

With **Enable Debug Log** on for a ship using default settings (Fill Cargo
Before Returning = on), let it run several buy and sell passes. Check the
debug log for the new `'... fill pass done deciding, committing N ware
type(s) ...'` line (buy) and its sell-side equivalent.

- [ ] **Step 3: Confirm the hypothesis — look for N greater than 1**

Expected (fix working): at least one pass shows `committing 2` or higher
— the first time in this project's history a fill-cargo pass has acted on
more than one ware. Confirm the ship actually receives/delivers multiple
wares from that pass (check cargo hold contents and/or the Logbook for
multiple entries from the same trip).

If every pass still shows `committing 1` with no exceptions, the
hypothesis in the design spec is wrong (or incomplete) — report this back
rather than assuming the fix is done, since it means the interrupt isn't
tied to the `<wait>` calls the way this fix assumes, and the fallback
approach (Classic-mode-style one-ware-per-pass with immediate delivery)
needs to be designed instead.

- [ ] **Step 4: Confirm the ship actually visits multiple sellers/buyers before returning home, not one round trip per ware**

`committing N` greater than 1 on its own is not sufficient — the commit
phase creates every buy/pickup order first, then every deliver/export
order, specifically so the ship's resulting order queue is `buy(w1),
buy(w2), buy(w3), deliver(w1), deliver(w2), deliver(w3)` (one filling tour,
then one trip home), not `buy(w1), deliver(w1), buy(w2), deliver(w2), ...`
(one round trip per ware). Watch the ship's actual flight path (or its
order list in the X4 UI) for a `committing 2`+ pass: confirm it flies to
each seller/buyer in turn and only returns to the home station once at the
end, not after every individual ware.

Also specifically watch a `committing 2`+ buy-side pass for a **known,
pre-existing gap**: the deliver leg's storage clamp reads the buyer's live
free space fresh for every decided ware, but nothing executes between the
queued orders in the commit phase, so it reads the *same* value each time
— it does not account for two decided wares sharing the same storage pool
within one pass. If a later ware's home delivery in a multi-ware pass
comes up short and leaves stray cargo, that's this known gap surfacing
now that fill-cargo can act on more than one ware — not a new regression.
See the design spec's "Known follow-up" section for the fix shape.

- [ ] **Step 5: Confirm the post-commit order-count line matches expectations**

Check the new `'  commit done: ship now has N trade order(s) (expected
M)'` debug line (buy) and its sell-side equivalent, logged right after
each commit phase. `N` should equal `M`. If `N` is lower than `M`, a
trade-offer object recorded during the decision phase went stale before
the commit phase could use it (the known, accepted risk from the design
spec's evidence trail) — report this back with the exact log line, since
it means some decided wares silently failed to get an order at all.

- [ ] **Step 6: Confirm no regression in single-ware passes**

For a pass that only ever finds one viable ware (e.g. a station with only
one wanted/sellable ware active), confirm it still behaves correctly —
buys/sells that one ware and delivers/exports it — matching the fixed
version-18 behavior for the single-ware case.

- [ ] **Step 7: Confirm the order-count ceiling holds**

Watch for any sign of the ship's order queue being over-filled (X4
UI shows a ship's order list; it should never exceed 6 entries at once).
The guard formula (`(($startingorders+($decided*.count*2)) le 4)`)
guarantees this for any starting order count, not just zero — for a ship
starting this pass with 0 existing orders, it should decide exactly 3
wares per pass (3 x 2 = 6 orders) and stop there rather than trying a 4th.
