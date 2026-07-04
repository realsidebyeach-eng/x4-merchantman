# Configuration

All settings live directly in the ship's order panel — there's no separate
menu. Assign the **Station Trader** order to a ship, then set the
parameters below.

## Parameter reference

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| **Home Station(s)** | station list | empty | One or more stations this ship restocks. Pick multiple to have one ship service several stations in rotation. |
| **Ware Priority List** | ware list | empty | The wares this ship is allowed to manage, and (in Priority mode) the order to buy them in. **Leave empty to auto-manage every ware the station currently wants to buy** — see below. If you list specific wares, only those are ever bought, even if the station is short on something else. |
| **Balanced Mode** | on/off | off | Off = **Priority mode**. On = **Balanced mode**. See below. |
| **Max Jump Range** | 0–10 | 3 | How many jumps from a home station's own sector to search for a seller. 0 = the home station's own sector only. |
| **Price Cap Mode** | on/off | on | On = cap is a **percentage below the home station's own price** for that ware (per-ware, automatic). Off = cap is one **flat absolute price** applied to every ware on the list. |
| **Absolute Max Price** | 0–1,000,000 | 10,000 | Used only when Price Cap Mode is **off**. Same ceiling for every ware. |
| **Min Discount %** | 0–95 | 20 | Used only when Price Cap Mode is **on**. Skips a deal entirely if it isn't at least this much cheaper than the home station's own price — filters out barely-better deals that aren't worth the trip. |
| **Scan Performance** | 1–15 | 5 | How many offer checks run per simulation tick before yielding. Lower this if you assign the order to many ships simultaneously and notice stutter; raise it for faster reactions on a lightly-loaded save. |
| **Enable Logbook Entries** | on/off | on | Writes a Logbook entry (Menu → Logbook) for every completed delivery: ware, amount, seller, price, total cost, and home station, with a "show on map" link. Turn off if you're running many traders and don't want the Logbook flooded. |
| **Enable Debug Log** | on/off | off | Writes a detailed trace to a log file every pass — see [Debugging](#debugging) below. Leave off during normal play; turn on only while troubleshooting a specific ship. |

## Auto-detect vs. an explicit ware list

Leave **Ware Priority List** empty and the ship manages *every* ware the
home station currently wants to buy — nothing to configure per-ware, it
just reacts to whatever the station's own buy offers show at the time.
In Priority mode with auto-detect, "top to bottom" isn't meaningful (there's
no list to order), so the ship instead processes whichever wanted ware is
most understocked first (lowest stock level relative to the station's
target).

List specific wares instead if you want to (a) restrict the ship to only
some of what the station wants — e.g. a station that trades a wide basket
but you only want this particular ship handling raw minerals — or (b) force
a specific top-to-bottom priority order in Priority mode regardless of
current stock levels.

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

## Troubleshooting

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
  trade orders queued.

Turn it back off once you're done — it writes on every pass for every
ware, so leaving it on across a long play session will grow the log file
continuously.
