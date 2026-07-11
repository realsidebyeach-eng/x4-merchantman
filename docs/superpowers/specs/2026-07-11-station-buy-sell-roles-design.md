# Buy/Sell Roles — Design

## Problem

Station Trader Assignments only ever does one direction of trade: it buys
wares a home station currently wants (its own buy offers, and now — since
the Build Storage feature — its Build Storage construction demand too) from
the cheapest accessible external seller, and delivers them home. It has no
proactive way to export a station's surplus production. The only
sell-adjacent behavior today is the incidental stray-cargo dump, which only
fires when the ship happens to be holding cargo not tied to a pending trade
order (e.g. after a forced pirate "drop cargo" demand) — it never goes out
of its way to pick up and sell a station's actual sell-offer surplus.

## Goal

Let an assigned ship also actively export a home station's surplus: pick up
wares the station currently has active **sell** offers for, and deliver
them to the best-paying accessible external buyer in range — the mirror
image of the existing buy pipeline. Expose this as an independent on/off
role alongside buying, so a ship can be buy-only, sell-only, or both.

## New parameters

Added to the existing `<params>` block, matching the style of
`manageconstruction` / `buildstoragefirst`:

- **`enablebuying`** (bool, default `true`) — "Buy wares the station wants."
  Master on/off switch for the existing restock pipeline (production buy
  offers + Build Storage). Defaults `true` so existing assignments keep
  their current behavior unchanged.
- **`enableselling`** (bool, default `true`) — "Sell the station's surplus
  wares." Master on/off switch for the new export pipeline. Defaults `true`
  — existing assigned ships pick up the new capability automatically after
  the update, and new assignments get full buy+sell behavior with no extra
  setup.

No dropdown/enum param is introduced — this codebase's params are only
bool/number/list types today, and a selection-type widget is unverified
here (same risk category flagged below for the library mechanism). Two
independent bools cover buy-only / sell-only / both / neither without that
risk.

## What "sell" role does

For each home station, once per pass (before the existing buy-category
loop — see Ordering below):

1. Query the station's own active **sell** offers (`find_sell_offer
   seller="$homestation"`) — this is its production surplus, the mirror of
   how the buy pipeline reads `find_buy_offer buyer="$homestation"`.
2. Filter/order by the **same** Ware Priority List used for buying (empty =
   auto-manage everything the station is currently selling; populated =
   only export wares that appear in the list, in that priority order).
   One list, one mental model for "wares this ship is allowed to manage,"
   regardless of direction — matches how Build Storage reused the list
   rather than adding a second one.
3. Run through the **same** Priority/Balanced and Fill-Cargo-Before-
   Returning logic already used for buying — no separate sell-specific
   copies of these toggles.
4. Price floor: never sell below the station's own current sell-offer
   price for that ware. No new percent/flat-price param — the floor is
   read directly off the offer already being processed. Simpler than the
   buy side's ceiling logic (which has a Price Cap Mode toggle) because
   there's no equivalent ambiguity to resolve here.
5. Search every sector in `$searchspaces` for accessible external buyers of
   that ware, reusing the exact access/relation/blacklist query pattern the
   stray-cargo dump logic already uses (`find_buy_offer tradepartner="this.
   ship" ... match_buyer tradesknownto="this.owner" match_relation_to
   relation="dock" match_use_blacklist`). Pick the highest-paying buyer at
   or above the floor, excluding the home station buying from itself.
6. Place two trade orders: a buy leg (ship buys from the home station's
   sell offer) and a sell leg (ship sells to the chosen external buyer) —
   structurally the same `create_trade_order` shape the buy pipeline
   already uses, just with source/destination swapped.
7. Logbook (`enablelogbook`) gets a new distinct entry text for exports,
   and debug trace lines (`debugtrace`) follow the same style as the
   existing buy-side lines, so troubleshooting output distinguishes buying
   from selling at a glance.

Selling only ever targets a home station's own sell offers — it does not
apply to Build Storage, which never has anything to sell.

## Ordering

Sell runs before buy, always, for every home station — fixed, no
configurable toggle (unlike `buildstoragefirst`). Selling first frees up
cargo space and generates income before the buy pass spends either, which
is the economically sensible default and doesn't need to be a per-ship
decision. `enablebuying`/`enableselling` gate whether each pass runs at all
for that ship; both off means the ship only still does its stray-cargo
safety-net handling (unchanged, independent of these roles — it's a
never-get-stuck-holding-cargo fallback, not a proactive role).

The existing 6-queued-trade-orders cap is shared and checked live, so the
sell pass (running first) naturally leaves whatever capacity remains for
the buy pass in the same tick — no new bookkeeping needed.

## Implementation approach: shared library

The sell pipeline is structurally a mirror of the buy pipeline (search
direction and price comparison flip: cheapest-seller-below-ceiling becomes
priciest-buyer-above-floor), but unlike the Build Storage feature it can't
reuse the existing category-loop trick, because that trick works by
looping the *same* direction of trade over two different buyer objects —
buy vs. sell is a genuinely different direction. Implementing it as a full
inline copy would roughly double the existing ~275-line buying pipeline and
push the main script past this project's 800-line file guideline.

Instead, the shared "find the best accessible counterparty within a price
bound, then place the matching trade-order pair" logic is extracted into a
new library file (`libraries/sbe_stationtrader_lib.xml`), referenced from
both the buy and sell passes via `<include_actions>` — the same general
mechanism `<include_interrupt_actions ref="GetBlacklistgroup"/>` already
uses for a *vanilla*-provided library, just authored for the first time in
this mod. This keeps the main script's size in check and follows this
project's "many small files" preference.

The exact XML mechanics of authoring and referencing a custom
(non-vanilla) library — parameter passing in/out of an `<include_actions>`
call, whether it can return values the caller needs (best counterparty
found, amount, etc.) — is a plan/implementation-time detail, not a
design-time one, matching how the Build Storage design deferred its own
query-syntax specifics.

## Known risk (flagged, not hidden)

`<include_actions>` for a custom, mod-authored library (as opposed to a
vanilla one like `GetBlacklistgroup`) is unverified in this codebase — no
existing file does this. This is the same category of risk the Build
Storage feature flagged for its `buildstorage` property query, which
turned out fine, but this one is a bigger structural bet since the whole
sell pipeline depends on it working (parameter/return-value passing in
particular).

**Required verification step**: after implementation, test in-game with
**Enable Debug Log** on, with a ship assigned to a station that has both
buy and sell demand, and confirm both passes actually execute and produce
correct trade orders. If custom `<include_actions>` doesn't work as
expected (e.g. no way to pass back a found offer/amount to the caller), the
fallback is inline duplication of the pipeline in the main script, accepted
by you as the explicit fallback during design — this doesn't change any of
the params, ordering, price-floor logic, or docs described above, only
where the code physically lives.

## Docs

- `README.md` — update the top description and "What it does" list to
  mention both directions; the "Known limitations" note about Build
  Storage no longer applies (already removed by that merge) but a new
  short line notes the sell floor is always home-price-or-better with no
  configurable margin.
- `docs/CONFIGURATION.md` — new rows for `enablebuying`/`enableselling` in
  the parameter reference table, a short "Buying vs Selling" explainer
  section (in the style of the existing "Priority vs Balanced" / "Fill
  Cargo" sections), and a troubleshooting entry for the new library
  mechanism specifically (what to check if selling silently never fires).
- `t/0001.xml` — new text IDs for the two new param labels/comments and a
  new logbook entry string for export deliveries (next available ID after
  the existing 300/301 Build Storage entries, e.g. 302).

## Versioning

- Bump the aiscript `version` attribute (currently 11, so 12).
- Add a `<patch sinceversion="12">` block defaulting both new params to
  `true` for ships already assigned this order in existing saves —
  matching the existing pattern used for `fillcargo`,
  `manageconstruction`, etc.

## Out of scope

- No configurable sell-price margin/premium param — the floor is always
  exactly the home station's own current sell-offer price, per your
  explicit choice to keep this dead simple.
- No separate Sell Ware Priority List, no separate sell-specific
  Balanced/Fill-Cargo toggles, no configurable buy/sell ordering toggle —
  all of these reuse the existing single param each, per your choices
  above.
- No dropdown/enum "Trading Role" param — two independent bools instead.
