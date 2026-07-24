# Trader Wallet Budget Fix — Design

## Problem: buying silently switches from the player wallet to the home station's account

While investigating why several trader ships appeared to be doing nothing,
the debug log showed buying activity starting up only after the ship's home
station's own account had credits added to it. Tracing the budget
calculation in both buy pipelines confirms why:

```xml
<set_value name="$remainingbudget" exact="player.money/100"/>
<do_if value="this.ship.commander">
	<do_if value="this.ship.commander.hasownaccount and this.ship.commander.money gt 0">
		<set_value name="$remainingbudget" exact="this.ship.commander.money/100"/>
	</do_if>
</do_if>
```

(and the identical pattern with `$spendablemoney` in the classic
one-ware-at-a-time buy pipeline). Whenever the ship's commander (its home
station) has its own account and that account holds money, the budget check
silently switches from the player's wallet to the station's account —
overriding, rather than adding to, the player money already read into
`$remainingbudget`/`$spendablemoney`.

This directly contradicts this project's own documented fix history. Per
`docs/CONFIGURATION.md`'s Troubleshooting section:

> **Ship shows up in the home station's Defence subordinate group**: this
> was a bug in versions up to 5 — assigning a single owned home station made
> the ship that station's commander-subordinate to draw purchase funds from
> its account, which the game defaulted into the Defence role. Fixed in
> version 6 by dropping that commander assignment entirely (purchases now
> always draw from the player wallet).

Version 6 already established that this mod's intended behavior is
"purchases always draw from the player wallet." The `this.ship.commander`
budget-override branch is leftover/regressed code that was never removed
from the budget calculation itself — it only stopped *this mod* from
assigning a commander. If a ship's commander ever gets set some other way
(e.g. the player manually assigns it as a subordinate to the station for an
unrelated reason, such as a defense or logistics group), this dead branch
reactivates the pre-version-6 behavior: purchases silently start drawing
from the station's account instead of the player's, with no indication to
the player that the funding source changed.

No README or CONFIGURATION.md text documents station-funded buying as an
intentional option — it is not a feature, just unreachable-in-normal-play
dead code that isn't actually unreachable.

**Fix:** delete both `this.ship.commander` override blocks. Budget is always
`player.money/100`, full stop — matching the version 6 fix's stated intent
and this mod's entire design (it's a *player* wallet-funded trade assistant;
`README.md`'s "Deliveries to your own home station are always free" section
already establishes that the wares themselves are paid for once, from the
player's wallet, when bought from an external seller).

## Scope

Two occurrences, both in `aiscripts/sbe_stationtrader.xml`:

1. Fill-cargo buy pipeline (`$remainingbudget`) — the block immediately
   after `<set_value name="$remainingbudget" exact="player.money/100"/>`.
2. Classic one-ware-at-a-time buy pipeline (`$spendablemoney`) — the block
   immediately after `<set_value name="$spendablemoney" exact="player.money/100"/>`.

Both are buy-side budget checks. This does not touch:
- The stray-cargo-to-external-buyer fallback (sells, doesn't buy — no
  budget check involved).
- The Selling pipeline's price-floor logic (unrelated: that's about what
  price to accept when selling, not what account funds a purchase).
- The home-delivery `price="0"` logic (already correct and unrelated — that
  zeroes the *delivery leg's* price, not the *purchase's* funding source).

## Out of scope

- No new param, no player-facing toggle for "let the station fund its own
  purchases" — this was never a documented feature, so there's nothing to
  preserve behind a flag. If a player later wants that as an actual
  feature, it would need its own design (a param, explicit opt-in, and
  probably a Logbook/debug note showing which account funded the
  purchase) — out of scope here.
- No change to `this.ship.commander` assignment itself — this mod already
  never sets it (version 6 fix); nothing to add or remove there.

## Versioning

Bump the aiscript `version` attribute (currently 14, so 15) — this project
bumps version on every functional change, per established convention
(version 6, version 9, version 13's precedents).
