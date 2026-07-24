# Storage Clamp Money-Entanglement Regression Fix — Design

## Problem: clamp_trade_amount silently gates free deliveries on the station's account balance

The prior fix (version 16, "Storage Capacity Checks") added `clamp_trade_amount`
calls to cap buy/deliver amounts against the home station's real storage
capacity, replacing the old `$wantedoffer.amount`-only cap (a demand target,
not real storage room). That fix was correct about the storage problem, but
introduced a new regression: `clamp_trade_amount`'s own schema documentation
says its `result` is "Amount clamped so it matches how much seller has and
how much buyer can store **and afford**" — it factors in the buyer's ability
to pay, using the trade offer's actual configured unit price, not the
`price="0"` this mod always uses for home deliveries.

Live testing confirmed this exactly: several trader ships sat stationary,
never creating any orders, and adding credits to their home station's
account balance immediately unstuck all of them and let them start moving.
That is the `clamp_trade_amount` "buyer_money" clamp firing on a trade that
will actually cost the home station nothing — the price override to `0`
only happens later, at the `create_trade_order` call, well after
`clamp_trade_amount` already evaluated affordability at the offer's real
market price.

This is confirmed by cross-referencing a mature, widely-used reference
implementation already installed in-game: TaterTrade
(`extensions/ws_2082610969/aiscripts/tatertrade.xml`, by Ludsoe/DeadAir).
It never uses `clamp_trade_amount` anywhere. Instead it computes storage and
money as two separate, explicit terms in a plain `min()`, e.g.
(`tatertrade.xml:1260`):

```
[this.ship.cargo.{$stationbuyoffer.ware}.free, $stationbuyoffer.amount, $foundselloffer.amount, $spendablemoney/(2*$wareprice)].min
```

It also confirms (`tatertrade.xml:992`, a debug line labeling the value
"FreeSpace") that `.cargo.{ware}.free` — the same property this file already
uses for the ship's own hold (`this.ship.cargo.{$ware}.free`) — works
identically on a **station** object (there, `$buyoffer.buyer`, the buying
station):

```
'FreeSpace: '+$buyoffer.buyer.cargo.{$buyoffer.ware}.free+' CommandersMoney: '+$buyoffer.buyer.money+...
```

**Fix:** remove all three `clamp_trade_amount` calls added in version 16.
Replace each with a plain `$homestation.cargo.{ware}.free` term folded
directly into the existing `min()` expression at each site — mirroring both
this file's own established pattern (capping against `this.ship.cargo.{...}.free`
for the ship's hold) and TaterTrade's proven approach. This checks storage
only, never money, so the home station's account balance can never again
gate a delivery that will actually be free.

## Scope

Same three edit points the version-16 fix touched, in
`aiscripts/sbe_stationtrader.xml`:

1. Fill-cargo buy pipeline (`$amount` computation before buying from the
   external seller) — add `$homestation.cargo.{$ware}.free` as a fifth
   `min()` term, remove the `clamp_trade_amount` call.
2. Fill-cargo pipeline's batched delivery loop (re-check right before each
   delivery order) — replace the `clamp_trade_amount` call with a
   `set_value` computing `[$deliveramounts.{$d},$homestation.cargo.{$deliverwares.{$d}.ware}.free].min`.
3. Classic buy pipeline (`$amount` computation) — same change as #1.

Does not touch:
- The stray-cargo-to-home fallback (version 16's second fix,
  `$strayhandled`/`$strayhomeamount`) — it never called `clamp_trade_amount`
  in the first place (it already only used `$strayhomeoffer.amount`), so it
  does not have this regression and needs no change here.
- The Selling pipeline, the stray-cargo-to-external-buyer fallback, the
  price-floor logic, or the wallet-budget fix (version 15) — all unrelated
  and already correct.
- `clamp_trade_amount` itself is a legitimate X4 action for genuinely priced
  trades (buyer actually pays something) — this fix doesn't deprecate it in
  general, only removes its use for this mod's always-free home-delivery
  legs, where it's the wrong tool.

## Out of scope

- No new param, no player-facing toggle.
- No change to `$deliverclamped`'s role or naming — it stays a plain local
  variable, just computed via `min()` instead of `clamp_trade_amount`.
- No revisit of the player-owned-vs-NPC-owned "transfer vs sell" idea
  discussed alongside this bug — that turned out to be unnecessary for this
  specific symptom (the stuck-ship cause was this regression, not a
  fundamental need for a transfer mechanic), so it stays parked unless a
  distinct need for it resurfaces later.

## Risk note

Unlike the version-16 fix (which introduced a first-ever, schema-validated
but runtime-unverified action), this fix *removes* that action and replaces
it with a plain arithmetic `min()` term using a property
(`.cargo.{ware}.free`) already proven twice over in this exact codebase
context: once by this file's own pre-existing ship-side usage, and once by
TaterTrade's station-side usage in a real, currently-installed, mature mod.
Confidence is higher than the fix it corrects. Manual in-game verification
remains required regardless, per this project's standing no-test-harness
convention.

## Versioning

Bump the aiscript `version` attribute (currently 16, so 17) — matching this
project's established convention of bumping on every functional change.
