# Configuration

All settings live directly in the ship's order panel — there's no separate
menu. Assign the **Station Trader** order to a ship, then set the
parameters below.

## Parameter reference

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| **Home Station(s)** | station list | empty | One or more stations this ship restocks. Pick multiple to have one ship service several stations in rotation. |
| **Ware Priority List** | ware list | empty | The wares this ship is allowed to manage. Wares the home station wants but that aren't on this list are **never** bought, even if the station is short on them. List order is the priority order used in Priority mode (top = first). |
| **Balanced Mode** | on/off | off | Off = **Priority mode**. On = **Balanced mode**. See below. |
| **Max Jump Range** | 0–10 | 3 | How many jumps from a home station's own sector to search for a seller. 0 = the home station's own sector only. |
| **Price Cap Mode** | on/off | on | On = cap is a **percentage below the home station's own price** for that ware (per-ware, automatic). Off = cap is one **flat absolute price** applied to every ware on the list. |
| **Absolute Max Price** | 0–1,000,000 | 10,000 | Used only when Price Cap Mode is **off**. Same ceiling for every ware. |
| **Min Discount %** | 0–95 | 20 | Used only when Price Cap Mode is **on**. Skips a deal entirely if it isn't at least this much cheaper than the home station's own price — filters out barely-better deals that aren't worth the trip. |
| **Max Discount %** | 1–99 | 90 | Used only when Price Cap Mode is **on**. Skips a deal if it's discounted by *more* than this — a sanity guard against implausible outlier prices. Must be greater than Min Discount % or nothing will ever qualify. |
| **Scan Performance** | 1–15 | 5 | How many offer checks run per simulation tick before yielding. Lower this if you assign the order to many ships simultaneously and notice stutter; raise it for faster reactions on a lightly-loaded save. |
| **Enable Logbook Entries** | on/off | on | Writes a Logbook entry (Menu → Logbook) for every completed delivery: ware, amount, seller, price, total cost, and home station, with a "show on map" link. Turn off if you're running many traders and don't want the Logbook flooded. |

## Priority mode vs. Balanced mode

**Priority mode** (Balanced Mode = off): each pass, the ship walks the Ware
Priority List top to bottom. For the first ware the station still wants, it
spends as much cargo space and budget as that single purchase needs before
even considering the next ware. Use this when one ware matters more than
the others — e.g. a station that will stop production entirely without
Ore, but can tolerate running low on Silicon Wafers for a while.

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
  station. Min Discount % and Max Discount % together define an acceptable
  price *band*: `home price × (1 − Max%)` ≤ accepted price ≤
  `home price × (1 − Min%)`. Concretely, with the defaults (Min 20%, Max
  90%): a deal must be at least 20% cheaper than the home station's own
  price to be worth the trip, but if something is priced more than 90%
  below — implausibly cheap — it's skipped as a likely outlier rather than
  snapped up.
- **Absolute mode**: one flat number for the whole list, with no lower
  bound. Useful if your Ware Priority List only contains wares of similar
  value (e.g. all raw minerals), or if you deliberately want a hard credit
  ceiling regardless of economy fluctuation. Not useful if your list mixes
  cheap and expensive wares — a cap generous enough for Hull Parts will be
  far too generous for Energy Cells.

There is no per-ware absolute price table (see the Known Limitations note
in the main README) — this is the one place the two modes genuinely
diverge in capability, not just in formula.

## Example recipes

**Single station, strict priority, tight budget**
- Home Station(s): `Ore Mine Alpha`
- Ware Priority List: `Ore`, `Silicon Wafers`, `Energy Cells`
- Balanced Mode: off
- Max Jump Range: `2`
- Price Cap Mode: on, Min Discount %: `10`, Max Discount %: `90`

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

- **Ship never buys anything**: confirm the home station actually has an
  active buy offer for at least one ware on your Ware Priority List (check
  the station's own Trade menu in-game — "buying" wares are shown there),
  and that Max Jump Range is wide enough to reach a seller your faction can
  dock at.
- **Ship buys the "wrong" ware first**: check Balanced Mode is off if you
  expect strict priority, and that the ware order in Ware Priority List
  matches what you intended — list order is read top to bottom exactly as
  displayed.
- **Ship won't take a deal you can see in the map/economy view**: the seller
  may not pass the access check (`match_relation_to relation="dock"`) — your
  faction needs actual docking/trade rights there, not just knowledge that
  the offer exists. Also check the price against your cap; a deal that
  looks cheap in absolute terms can still be above a tight percent-below-
  home-price cap if the home station's own price for that ware is low.
- **No trade orders queue up even though a valid deal was found**: the ship
  caps itself at 6 simultaneous queued trade orders at a time (to avoid
  runaway queuing); wait for existing orders to clear.
