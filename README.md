# Station Trader Assignments (X4: Foundations mod)

A ship order that turns any trade ship into a dedicated logistics runner for
one or more home stations: it can buy the wares a station currently wants
from the cheapest accessible seller, sell the station's production surplus
to the best-paying accessible buyer, or both — respecting a priority or
balanced strategy and a price ceiling (buying) or floor (selling) you
control. Buying and selling are independently toggleable per ship via
**Enable Buying** and **Enable Selling** (both on by default).

- **[docs/INSTALL.md](docs/INSTALL.md)** — how to install and enable the mod.
- **[docs/CONFIGURATION.md](docs/CONFIGURATION.md)** — full parameter
  reference, buying-strategy details, and troubleshooting.

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
   before heading home or to a buyer, returning short of a full hold only
   when wares genuinely aren't available or once free cargo space drops
   below a configurable percent of capacity (so it stops chasing one more
   stop) — one gathering (or exporting) trip instead of a round trip per
   ware (toggle this off for the classic immediate buy-then-deliver
   behavior).
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

## Known limitations (by design, not oversights)

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
content.xml                        - extension metadata
aiscripts/sbe_stationtrader.xml     - the order + AI behavior script
libraries/icons.xml                 - order icon mapping (reuses a vanilla icon)
t/0001.xml                          - order/parameter display text (English)
docs/INSTALL.md                     - install & enable instructions
docs/CONFIGURATION.md               - parameter reference & recipes
```

## Reference

Built from patterns verified against the real, shipped "TaterTrade" mod
(order/param schema, offer-query tags, access checks, two-leg
buy-then-deliver trade order construction) rather than guessed syntax.
