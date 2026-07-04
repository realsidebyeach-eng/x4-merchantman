# Station Trader Assignments (X4: Foundations mod)

A ship order that turns any trade ship into a dedicated restocker for one or
more home stations: it watches what the station currently wants to buy,
shops for the cheapest accessible seller within a configurable jump range,
and keeps ferrying goods in — respecting a priority or balanced buying
strategy and a price ceiling you control.

- **[docs/INSTALL.md](docs/INSTALL.md)** — how to install and enable the mod.
- **[docs/CONFIGURATION.md](docs/CONFIGURATION.md)** — full parameter
  reference, buying-strategy details, and troubleshooting.

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

## Known limitations (by design, not oversights)

- **Absolute Max Price is a single flat value for every ware in the list**,
  not an independently configurable per-ware table — X4's native ship-order
  parameter UI doesn't support a "table of ware → number" widget. Use
  **Price Cap Mode = on** (percent below home price) for genuine per-ware
  differentiation instead.
- Station build-storage restocking (a separate buyer object some stations
  have) isn't included; only the station's own regular buy offers are read.

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
