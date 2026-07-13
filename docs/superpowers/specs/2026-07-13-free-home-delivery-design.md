# Free Home-Station Delivery — Design

## Problem

A ship can hold cargo it can never actually deliver: the "deliver to home
station" leg of a trade order makes the home station **pay** the ship (at
the home station's own buy-offer price), drawn from the station's own
account. If that station's own account doesn't have enough credits, the
delivery never completes, and the ship sits stuck holding the cargo
indefinitely — reported in-game as ships holding cargo that "won't trade"
with their assigned home station.

## Root cause

The wares were already paid for once — when the ship originally bought
them from an external seller (or, for stray cargo, however they ended up
in the hold). The home-station "delivery" leg is a second, redundant
payment for the same goods, and it's this second payment that can fail
when a player-owned station's own account runs low. Since the ship
delivering and the station receiving are both player-owned in every normal
use of this mod, that second payment shouldn't exist at all.

## Fix

Set `price="0"` on every `create_trade_order` that delivers cargo already
in the ship's hold to a home station:

1. Buy pipeline, Fill-Cargo mode's batch delivery leg.
2. Buy pipeline, Classic mode's delivery leg.
3. The stray-cargo-to-home-station fallback leg (cargo left over from an
   interrupted trade, e.g. after a forced pirate "drop cargo" demand).

No ownership check is added — this mod's entire premise is a ship
delivering to a home station the player assigned it to, so an
unconditional zero-price delivery is correct for every real use of this
order. No new param, no toggle: this always applies.

The Logbook is unaffected — its delivery-completed entries already log the
*acquisition* price (what was paid to the external seller,
`$deliverprices`/`$bestprice`), not the home-delivery price, so a player
never sees a `0cr` line where a real cost previously appeared.

## Out of scope

- The sell pipeline's pick-up leg (ship buying the station's own surplus
  to export) is unaffected — the station is the *seller* there, receiving
  payment, so insufficient funds can never block it, and zeroing that
  price would mean the station gets no credit for producing what it
  sells. Not touched.
- The sell pipeline's export leg (selling to an external buyer) is real
  revenue from outside the player's empire and stays priced normally.

## Versioning

Bump the aiscript `version` attribute (currently 12, so 13) — matching
this project's established convention of bumping on every functional
change, even ones with no new param (e.g. the version-9 Logbook fix had no
new param either).

## Docs

`docs/CONFIGURATION.md` gains a short note (near the existing "Handling
pirates and other interruptions" section, since it touches the same
stray-cargo delivery path) explaining that home-station deliveries are
always free — the station's own account is never charged for what a ship
already paid for externally.
