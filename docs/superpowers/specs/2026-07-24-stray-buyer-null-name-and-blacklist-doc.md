# Stray-Buyer Null-Name Fix + Blacklist Travel Limitation Doc — Design

## Problem 1: "selling to null" in the debug log

The stray-cargo-to-external-buyer fallback (the code path that fires when
no home station wants a piece of stray cargo, so it's sold to the
best-paying accessible buyer instead) logs the buyer's name **after**
`create_trade_order` has already consumed the `$beststraybuyer` offer
reference:

```xml
<create_trade_order name="$beststraybuyer" object="this.object" tradeoffer="$beststraybuyer" amount="[$strayamount,$beststraybuyer.amount].min" immediate="false"/>
<debug_to_file ... text="'...selling '+$strayamount+' to '+$beststraybuyer.buyer.knownname+'...'" .../>
```

Reading `$beststraybuyer.buyer` after that point returns null, so the log
always shows "selling ... to null ... instead" regardless of who the
actual buyer was. This is the exact bug class already fixed in version 9
("Logbook entries show 'from null'") — capturing an offer's name *after*
`create_trade_order` consumes it — just never applied to this one
code path, which was added in a later commit than that fix.

Confirmed as the only remaining instance: every other `.buyer.knownname` /
`.seller.knownname` read in this file is either already captured into a
name variable before its `create_trade_order` call (four existing
instances), or is a "picked buyer/seller" debug line that fires *before*
any `create_trade_order` call for that offer (two instances) — safe
either way. This one path is the sole gap.

**Fix:** capture `$beststraybuyer.buyer.knownname` into a new
`$beststraybuyername` variable immediately before the `create_trade_order`
call, then use that variable in the debug line — mirroring the exact
comment and pattern already used at the four existing capture sites
(`$sellbuyername`, `$sellername`).

This is purely a debug-log cosmetic fix. The actual trade order is
unaffected — `create_trade_order` already runs against a valid,
not-yet-consumed reference; only the *subsequent* log line was reading a
stale one.

## Problem 2: ships routing through blacklisted sectors — documented as a limitation, not fixed

Investigated whether this mod could prevent a ship's automated trade-order
travel from passing through a blacklisted sector as a waypoint en route to
a legitimate (non-blacklisted) destination. Confirmed this is not
possible from this mod:

- This mod never issues travel/movement commands. `create_trade_order`
  hands travel execution to X4's standard, built-in trade-order AI — the
  same mechanism every trade ship in the game uses, modded or not.
- No route-safety/avoidance action exists anywhere in the X4 script
  schema (`aiscripts.xsd` / `common.xsd`) for an order script to invoke.
- Even this mod's own jump-distance reachability check
  (`find_cluster_in_range`) can't establish route safety — two sectors N
  jumps apart can have multiple routes of equal length, some through
  blacklisted territory and some not, with no way to query which one the
  ship's autopilot will pick.

This mod already correctly avoids *searching for or choosing* blacklisted
sectors as trade destinations — confirmed structurally (the sector-level
search-space filter plus five separate `match_use_blacklist` checks, one
per external search) and via debug log evidence. It has no lever over the
flight path taken to reach a chosen, legitimate destination — that's
either a vanilla X4 characteristic (the "Default Safe Travel" blacklist
apparently isn't consulted by automated trade-order pathing the way it is
for manual Travel Mode) or an engine limitation, not a mod defect.

**Fix:** document this as a known limitation in README.md, next to the
existing Absolute Max Price entry, so it isn't mistaken for a bug again.
No code change — there is no code lever to pull.

## Out of scope

- No change to the actual trade order created by the stray-cargo fallback
  (still non-zero price — this is a genuine external sale, unlike the
  home-delivery legs zeroed in the prior free-home-delivery fix).
- No attempt at route-avoidance workarounds (e.g. reducing search range
  near blacklisted sectors) — the existing **Max Jump Range** param
  already gives the player a manual lever for this; no new param needed.

## Versioning

Bump the aiscript `version` attribute (currently 13, so 14) — matching
this project's established convention of bumping on every functional
change. (The blacklist-limitation doc addition alone wouldn't need a
version bump, but the null-name logging fix is a functional change to the
script.)
