# Storage Capacity Checks — Design

## Problem 1: buy/deliver decisions trust a demand target, not real storage room

Investigating why ship LKT-678 bought a full 473-unit hold of Engine Parts
but only ever got 27 of them accepted by its home station (the rest — 446
units — sat in the ship's hold indefinitely) traced back to every buy
pipeline in this file capping its purchase/delivery amount with
`$wantedoffer.amount` alone:

```xml
<set_value name="$amount" exact="[$wareunitcap,$bestseller.amount,$wantedoffer.amount,$remainingbudget/$bestprice].min"/>
```

`$wantedoffer.amount` (logged as `homewants=900` in this case) reflects how
much more of the ware the station's buy offer wants to reach its target
stock level — it is a demand figure, not a live "how much can I physically
store right now" figure. When the station's actual warehouse room for that
ware is much smaller than its target (here: because the station's *own*
Engine Parts sell-surplus can't clear either — its sell-floor sits above
every accessible buyer's price — so the warehouse stays full), the ship
buys and carries far more than the station can currently accept, wasting
cargo hold space and player money on goods that end up stranded.

X4's script schema already has a purpose-built action for exactly this:
`clamp_trade_amount` (`common.xsd`), which clamps a candidate amount
against the buyer's and/or seller's *actual* storage/cargo/money
situation, and reports which limit reduced it (`'seller_cargo'`,
`'buyer_cargo'`, or `'buyer_money'`). This mod has never used it — grep for
`clamp_trade_amount` in `aiscripts/sbe_stationtrader.xml` returns nothing.

**Fix:** after computing the existing `$amount` cap in each buy pipeline,
run it through `clamp_trade_amount` against `$homestation` as buyer, and
use the clamped result for both the external purchase and the home
delivery. Two call sites, matching the two independent decision points:

1. **Before buying from the external seller** (both the fill-cargo pipeline
   and the classic one-ware-at-a-time pipeline) — so the ship never
   commits cargo space and money to more than the home can currently
   absorb.
2. **Immediately before creating each batched delivery order** in the
   fill-cargo pipeline's "deliver everything bought in one batch" step —
   defense in depth, mirroring the stray-cargo fallback's existing pattern
   of re-checking live capacity right before creating a delivery order.
   This matters in fill-cargo mode specifically because multiple wares are
   bought in the same pass before any of them are delivered; re-checking
   at the point of actual delivery-order creation catches any case where
   an earlier ware's delivery in the same batch already used up shared
   storage room. (The classic pipeline only needs call site 1: it buys and
   delivers one ware immediately, reusing the same `$amount` for both
   orders, so there's no batching gap for a second check to catch.)

The clamped amount then flows into the existing `<do_if value="$amount gt
0">` gate exactly as before — a clamp result of 0 already means "don't buy
or deliver this ware," no new gate needed.

## Problem 2: the stray-cargo-to-home fallback marks itself "handled" on a zero-amount order

The stray-cargo fallback (added to catch cargo left over from interrupted
deliveries, and now also catching the overbuying scenario above) has its
own bug that compounds Problem 1's effect: once cargo becomes "stray," it
can get permanently stuck rather than eventually being sold to any
accessible external buyer.

```xml
<do_if value="$strayhomeoffer.available">
  <create_trade_order name="$strayhomeoffer" object="this.object" tradeoffer="$strayhomeoffer" amount="[$strayamount,$strayhomeoffer.amount].min" price="0" immediate="false"/>
  <set_value name="$strayhandled" exact="true"/>
  ...
</do_if>
```

`$strayhandled` is set `true` as soon as the home station has *any* active
buy offer for the ware (`.available`) — regardless of whether
`$strayhomeoffer.amount` is actually greater than 0. When the home's live
buy capacity is 0 (exactly the warehouse-full scenario from Problem 1), this
creates a 0-amount trade order that moves nothing, yet still marks the
stray cargo as "handled." Since the external-buyer fallback only runs when
`not $strayhandled` (a few lines below), the ship never falls through to
try selling the leftover cargo to any accessible external buyer — even
though that path has no price floor and would gladly take it. The result:
cargo can get stuck at the home-delivery step forever, repeating a
zero-progress order every pass, with no other outlet.

This is compounded by a second issue in the same block: the debug log
line reports `$strayamount` (the ship's full held quantity) instead of the
amount actually used in the order:

```xml
<debug_to_file ... text="'Stray cargo: delivering '+$strayamount+' '+$strayware+' to home station '+$strayhome.knownname+'.'" .../>
```

This is why the log for LKT-678 showed "delivering 446 Engine Parts"
repeated identically across dozens of passes — the log always shows the
ship's current on-hand quantity, never the real (possibly zero) amount
the order actually carried, making a permanently-stuck zero-progress state
indistinguishable in the log from a slow-but-working partial delivery.

**Fix:** compute the actual clamped order amount (`[$strayamount,
$strayhomeoffer.amount].min`) into its own variable first. Only create the
order and set `$strayhandled = true` when that amount is greater than 0.
Log that same variable, not `$strayamount`. When the clamped amount is 0,
`$strayhandled` stays `false` and the block correctly falls through to try
an external buyer instead.

## Scope

Four edit points in `aiscripts/sbe_stationtrader.xml`:

1. Fill-cargo buy pipeline: clamp before buying (~line 664, `$amount`
   computation for the wanted-ware loop).
2. Fill-cargo buy pipeline: clamp before each batched delivery order
   (~line 697-698, the `$deliverwares`/`$deliveramounts` delivery loop).
3. Classic buy pipeline: clamp before buying (~line 797, `$amount`
   computation — covers both the buy and immediate-deliver orders since
   both reuse the same `$amount`).
4. Stray-cargo-to-home fallback: fix the `$strayhandled` guard and debug
   log (~line 181-186).

Does not touch:
- The Selling pipeline (station's own production surplus to external
  buyers) — home is the *seller* there, not the buyer; no home-storage
  clamp applies.
- The stray-cargo-to-external-buyer fallback (~line 193-224) — already
  correct; it has no price floor and no analogous "handled" short-circuit
  bug.
- The price-floor logic itself (already fixed/documented behavior from
  prior work) — this fix addresses a different failure mode (leftover
  cargo with no outlet), not the floor's own price comparison.
- The wallet-budget fix (already shipped, version 15) — orthogonal, no
  interaction.

## Out of scope

- No new param, no player-facing toggle. This mirrors the wallet-budget
  fix's precedent: a correctness fix using an already-correct pattern
  elsewhere in the file (the stray-cargo fallback's existing fresh-check
  idiom), applied consistently.
- No change to the Logbook money/amount text in the fill-cargo delivery
  loop — those numbers describe the *purchase* (already executed, already
  paid for), not the delivery, and stay accurate regardless of how much
  the clamp later reduces the delivery amount.
- No attempt to expose `clamp_trade_amount`'s `reason` output
  (`'seller_cargo'`/`'buyer_cargo'`/`'buyer_money'`) as a new debug field
  in this pass — worth a follow-up if diagnosing future capacity issues
  proves difficult without it, but not required to fix the two bugs above.

## Risk note

`clamp_trade_amount` is a real, documented X4 script action
(`common.xsd`), but this mod has never used it before — unlike the other
recent fixes, which reused already-proven-safe patterns already present
elsewhere in this file. Manual in-game verification (Task 2 in the
implementation plan) is more load-bearing for this change than for prior
fixes and should not be skipped.

## Versioning

Bump the aiscript `version` attribute (currently 15, so 16) — matching
this project's established convention of bumping on every functional
change.
