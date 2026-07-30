# Build Storage Storage-Check Wrong-Object Fix — Design

## Problem: the storage-capacity check queries the wrong object during Build Storage passes

The version-17 fix ("storage clamp money-entanglement regression fix")
replaced `clamp_trade_amount` calls with a plain
`$homestation.cargo.{ware}.free` term folded into each buy pipeline's
`min()` expression. That was correct for a ship's regular production-buyer
category, but broke Build Storage entirely.

This mod services up to two "categories" of buyer per home station in one
pass — the station's own production buy offers, and (if present and
enabled) its Build Storage construction demand — via a shared loop
(`aiscripts/sbe_stationtrader.xml:527-536`):

```xml
<set_value name="$stationbuildstorage" exact="$homestation.buildstorage"/>
...
<do_all exact="$categorybuyers.count" counter="$cat">
	<set_value name="$querybuyer" exact="$categorybuyers.{$cat}"/>
	...
	<find_buy_offer buyer="$querybuyer" result="$homebuyoffers" multiple="true"/>
```

`$categorybuyers` holds `$homestation` for the production category and
`$stationbuildstorage` (`$homestation.buildstorage` — a distinct
construction-staging object) for the Build Storage category. Every buy
offer found and processed inside this loop belongs to `$querybuyer`, not
necessarily `$homestation` — the two are the same object only for the
production category.

The version-17 fix hardcoded `$homestation.cargo.{ware}.free` at all three
of its edit points, inside this same loop. For the production category
this is correct (`$querybuyer == $homestation`). For the Build Storage
category it queries the wrong object: the base station under
construction, not the build storage actually holding/receiving the
delivered wares. Live testing (ship KHT-578, assigned to a station under
construction with a 100M credit budget) confirmed this: every single ware
in every Build Storage buy pass clamped to `amount=0`, despite `wareunitcap`,
`sellerhas`, and `afford` all being comfortably non-limiting — the ship
never bought or moved anything, because Build Storage was the only active
category for a station that isn't built yet, and that category was now
permanently broken.

**Fix:** replace `$homestation.cargo.{...}.free` with
`$querybuyer.cargo.{...}.free` at all three sites. `$querybuyer` is already
in scope at every edit point (set once at the top of the per-category loop,
before any of the three sites run) and is the actual buyer for whichever
category is currently processing — `$homestation` for production,
`$stationbuildstorage` for Build Storage.

## Scope

The same three edit points touched by the version-17 fix, in
`aiscripts/sbe_stationtrader.xml`:

1. Fill-cargo buy pipeline's `$amount` computation (~line 669).
2. Fill-cargo pipeline's batched delivery loop's re-check (~line 704).
3. Classic buy pipeline's `$amount` computation (~line 805).

Each site's explanatory comment is also updated to say "the actual buyer's
(base station, or its Build Storage) real current free space" instead of
"the station's real current free space," so the comment doesn't imply a
single fixed object.

Does not touch:
- `$stationbuildstorage`'s own definition, the category-buyer list
  construction, or any other part of the dual-category loop — those were
  already correct; only the storage-check object reference was wrong.
- The stray-cargo-to-home fallback, the Selling pipeline, the
  stray-cargo-to-external-buyer fallback, the price-floor logic, or the
  wallet-budget fix — all unrelated.

## Out of scope

- No new param, no player-facing toggle.
- No change to how `$querybuyer` itself is determined — this fix only
  changes which variable the storage check reads from, not the category
  logic that produces `$querybuyer`.

## Risk note

Low risk relative to the fix it corrects: this is a variable-name swap
(`$homestation` → `$querybuyer`) at three sites, using a variable already
proven in scope and already used for the actual offer search
(`find_buy_offer buyer="$querybuyer"`) immediately above each site in the
same loop iteration. No new property or action is introduced. Manual
in-game verification remains required per this project's standing
no-test-harness convention, specifically re-testing the Build Storage
scenario that surfaced this bug.

## Versioning

Bump the aiscript `version` attribute (currently 17, so 18) — matching
this project's established convention of bumping on every functional
change.
