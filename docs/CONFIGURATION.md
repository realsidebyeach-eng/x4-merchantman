# Configuration

All settings live directly in the ship's order panel — there's no separate
menu. Assign the **Station Trader** order to a ship, then set the
parameters below.

## Parameter reference

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| **Home Station(s)** | station list | empty | One or more stations this ship restocks. Pick multiple to have one ship service several stations in rotation. |
| **Enable Buying** | on/off | on | On = buy wares the station wants and deliver them home. See [Buying vs Selling](#buying-vs-selling-roles) below. |
| **Enable Selling** | on/off | on | On = sell the station's surplus wares (its active sell offers) to the best accessible buyer. See [Buying vs Selling](#buying-vs-selling-roles) below. |
| **Ware Priority List** | ware list | empty | The wares this ship is allowed to manage, and (in Priority mode) the order to handle them in — applies to both buying and selling. **Leave empty to auto-manage every ware the station currently wants to buy and/or sell** — see below. If you list specific wares, only those are ever bought/sold, even if the station has demand or surplus elsewhere. |
| **Balanced Mode** | on/off | off | Off = **Priority mode**. On = **Balanced mode**. See below. |
| **Fill Cargo Before Returning** | on/off | on | On = buy and/or sell (per whichever roles are enabled) from every wanted/sellable ware first, tracking remaining cargo space (and credits, for buying) as it goes, then deliver/export everything home or to buyers in one trip once the hold is full or nothing more is available/affordable. Off = classic mode, buy or sell and immediately deliver/export one ware at a time. See below. |
| **Also Resupply Build Storage** | on/off | on | On = also buy construction wares a home station's Build Storage currently wants (station under construction or having a module added), same as normal production wares. Off = Build Storage is ignored entirely. See [Build Storage](#build-storage-construction-wares) below. |
| **Build Storage First** | on/off | on | Used only when Also Resupply Build Storage is on. On = Build Storage's wanted wares are fully serviced before the station's own production-wanted wares each pass. Off = production wares first. |
| **Max Jump Range** | 0–10 | 3 | How many jumps from a home station's own sector to search for a seller. 0 = the home station's own sector only. |
| **Price Cap Mode** | on/off | on | On = cap is a **percentage below the home station's own price** for that ware (per-ware, automatic). Off = cap is one **flat absolute price** applied to every ware on the list. |
| **Absolute Max Price** | 0–1,000,000 | 10,000 | Used only when Price Cap Mode is **off**. Same ceiling for every ware. |
| **Min Discount %** | 0–95 | 20 | Used only when Price Cap Mode is **on**. Skips a deal entirely if it isn't at least this much cheaper than the home station's own price — filters out barely-better deals that aren't worth the trip. |
| **Scan Performance** | 1–15 | 5 | How many offer checks run per simulation tick before yielding. Lower this if you assign the order to many ships simultaneously and notice stutter; raise it for faster reactions on a lightly-loaded save. |
| **Enable Logbook Entries** | on/off | on | Writes a Logbook entry (Menu → Logbook) for every completed delivery: ware, amount, seller, price, total cost, and home station, with a "show on map" link. Turn off if you're running many traders and don't want the Logbook flooded. |
| **Enable Debug Log** | on/off | off | Writes a detailed trace to a log file every pass — see [Debugging](#debugging) below. Leave off during normal play; turn on only while troubleshooting a specific ship. |

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

Leave **Ware Priority List** empty and the ship manages *every* ware the
home station currently wants to buy — nothing to configure per-ware, it
just reacts to whatever the station's own buy offers show at the time.
In Priority mode with auto-detect, "top to bottom" isn't meaningful (there's
no list to order), so the ship instead processes whichever wanted ware is
most understocked first (lowest stock level relative to the station's
target).

The same applies to selling: leave the list empty and, with Enable Selling
on, the ship manages *every* ware the home station currently has for sale,
reacting to whatever its own sell offers show at the time. In Priority mode
with auto-detect, selling mirrors buying but in the opposite direction — it
processes whichever sellable ware is most oversupplied first (highest stock
level relative to the station's target), instead of most understocked
first.

List specific wares instead if you want to (a) restrict the ship to only
some of what the station wants or has for sale — e.g. a station that trades
a wide basket but you only want this particular ship handling raw minerals
— or (b) force a specific top-to-bottom priority order in Priority mode
regardless of current stock levels. The same list applies to both roles at
once — see [Buying vs. Selling roles](#buying-vs-selling-roles) above.

## Priority mode vs. Balanced mode

**Priority mode** (Balanced Mode = off): each pass, the ship processes
wanted wares one at a time — either in your Ware Priority List's order, or
by most-understocked-first if the list is empty — spending as much cargo
space and budget as that single purchase needs before even considering the
next ware. Use this when one ware matters more than the others — e.g. a
station that will stop production entirely without Ore, but can tolerate
running low on Silicon Wafers for a while.

**Balanced mode** (Balanced Mode = on): each pass, the ship still only
considers wares the station currently wants, but caps every ware to
roughly `free cargo space ÷ number of wanted wares this pass`, so buying one
doesn't crowd out the others. Over several passes the delivered quantities
stay approximately even across the list. Use this for a general-purpose
input hauler feeding several inputs a production module needs in parallel
(e.g. Energy Cells, Ore, and Silicon all needed together for a Refinery).

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

In Balanced mode, Fill Cargo Before Returning also changes how the "fair
share" cargo cap is computed: instead of a flat `hold ÷ wares wanted this
pass`, it's `cargo space still left ÷ wares still left to consider`,
recalculated as it goes — so earlier wares in the pass don't get an
unfairly generous share just because later wares haven't been priced out
yet.

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

- **Percentage mode (default, recommended)**: scales automatically per
  ware, since it's computed off of that ware's own price at the home
  station. Min Discount % sets the price ceiling: `accepted price ≤ home
  price × (1 − Min%)`. Concretely, with the default (Min 20%): a deal must
  be at least 20% cheaper than the home station's own price to be worth
  the trip.
- **Absolute mode**: one flat number for the whole list. Useful if your
  Ware Priority List only contains wares of similar value (e.g. all raw
  minerals), or if you deliberately want a hard credit ceiling regardless
  of economy fluctuation. Not useful if your list mixes cheap and
  expensive wares — a cap generous enough for Hull Parts will be far too
  generous for Energy Cells.

There was previously also a Max Discount % upper guard (rejecting deals
discounted by *more* than some percent, as a sanity check against
implausible outlier prices). It's been removed for now at the point of a
live troubleshooting session — it's straightforward to reintroduce if it
turns out to be needed later.

There is no per-ware absolute price table (see the Known Limitations note
in the main README) — this is the one place the two modes genuinely
diverge in capability, not just in formula.

## Example recipes

**Fully hands-off restocker**
- Home Station(s): `Trade Station Alpha`
- Ware Priority List: *(leave empty)*
- Balanced Mode: off
- Max Jump Range: `5`
- Price Cap Mode: on, Min Discount %: `20`

Ship auto-detects whatever the station is currently trying to buy,
tackling the most understocked ware first each pass, without you ever
having to name specific wares.

**Single station, strict priority, tight budget**
- Home Station(s): `Ore Mine Alpha`
- Ware Priority List: `Ore`, `Silicon Wafers`, `Energy Cells`
- Balanced Mode: off
- Max Jump Range: `2`
- Price Cap Mode: on, Min Discount %: `10`

Ship will fully restock Ore first, every pass, before ever touching
Silicon Wafers or Energy Cells, only taking deals priced at least 10%
below the station's own listed price for any of them, searching up to 2
jumps out.

**Multi-station balanced feeder**
- Home Station(s): `Refinery One`, `Refinery Two`
- Ware Priority List: `Ore`, `Silicon Wafers`, `Water`
- Balanced Mode: on
- Max Jump Range: `5`
- Price Cap Mode: off, Absolute Max Price: `50`

Ship alternates between the two refineries, and within each visit spreads
its cargo hold roughly evenly across whichever of the three wares that
station currently wants, never paying more than 50cr/unit for any of them.

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

- **Logbook entries show "from null" and a station shows as a hex ID
  (e.g. `0x74572`) instead of its name**: this was a bug in versions up to
  8. The seller was being read from the trade offer *after* it had already
  been consumed by `create_trade_order`, which invalidates that reference,
  and station objects don't auto-render a friendly name in Logbook text
  the way ware names do. Fixed in version 9 by capturing the seller's name
  as a string before the offer is used, and by explicitly using
  `.knownname` for every station reference in Logbook text. Only affects
  the Logbook display text — the actual trades themselves were unaffected.
- **Ship shows up in the home station's Defence subordinate group**: this
  was a bug in versions up to 5 — assigning a single owned home station
  made the ship that station's commander-subordinate to draw purchase
  funds from its account, which the game defaulted into the Defence
  role. Fixed in version 6 by dropping that commander assignment entirely
  (purchases now always draw from the player wallet). If a ship was
  already mis-assigned to Defence before updating, remove it from that
  group manually in the station's subordinates panel — the fix only
  prevents it going forward, it doesn't undo an existing assignment.
- **Ship never buys anything**: confirm the home station actually has an
  active buy offer for at least one ware right now (check the station's
  own Trade menu in-game — "buying" wares are shown there) — and, if you
  populated Ware Priority List, that at least one of those specific wares
  is among them. Also check Max Jump Range is wide enough to reach a
  seller your faction can dock at.
- **Ship buys the "wrong" ware first**: check Balanced Mode is off if you
  expect strict priority, and that the ware order in Ware Priority List
  matches what you intended — list order is read top to bottom exactly as
  displayed.
- **Ship won't take a deal you can see in the map/economy view**: the seller
  may not pass the access check (`match_relation_to relation="dock"`), the
  offer may not be "known" to your faction yet (needs prior scouting), or
  the sector/faction may be on your Empire > Blacklist. Also check the
  price against your cap; a deal that looks cheap in absolute terms can
  still be above a tight Min Discount % cap if the home station's own
  price for that ware is low.
- **No trade orders queue up even though a valid deal was found**: the ship
  caps itself at 6 simultaneous queued trade orders at a time (to avoid
  runaway queuing); wait for existing orders to clear.
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
- **Ship never exports anything**: confirm **Enable Selling** is on, and
  that the home station actually has an active sell offer for at least one
  ware right now (check the station's own Trade menu in-game — "selling"
  wares are shown there). If you populated Ware Priority List, confirm at
  least one of those wares is among what the station is currently selling.
  Also check Max Jump Range reaches a buyer your faction can dock at, and
  remember the price floor is always the home station's own current asking
  price — a buyer offering less than that is correctly skipped, not a bug.
  Turn on Enable Debug Log and look for lines tagged `(Selling)`.

## Debugging

Turning on **Enable Debug Log** writes one line per decision to
`<X4 user data folder>/StationTrader/<ship idcode>.txt` (the same
`debug_to_file` mechanism vanilla X4 and most script mods use — on this
machine that's under `~/.config/EgoSoft/X4/` for the native Linux build,
or your `Documents/Egosoft/X4` folder on Windows; exact subpath depends on
your X4 logging configuration). Each pass logs:

- how many of the home station's buy offers will be managed — either "all
  of them" (auto-detect) or "N match the ware priority list" (explicit
  list);
- which sectors were included in the search this pass (after excluding any
  blacklisted sectors);
- per ware: the computed price ceiling, how many accessible sellers were
  found, and either which one was picked (and at what price) or the
  cheapest price seen if nothing qualified;
- the final purchase amount and the four numbers it was capped by (cargo
  space, seller's stock, home station's demand, affordable quantity) —
  useful if a seller was picked but no trade order actually appeared;
- a note if the ship skipped remaining wares because it already had 6
  trade orders queued;
- with Fill Cargo Before Returning on: the starting cargo space and
  credits for the pass, and a final line once the fill pass ends showing
  how many ware types are being delivered home in one trip and how much
  cargo space is still free (0, if the hold filled up completely);
- if the ship is idle with stray cargo: which ware, and whether it went
  to a home station, to a fallback buyer (with which one and at what
  price), or that neither was found yet and it'll retry next pass.

Turn it back off once you're done — it writes on every pass for every
ware, so leaving it on across a long play session will grow the log file
continuously.
