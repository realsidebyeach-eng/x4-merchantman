# Build Storage Resupply — Design

## Problem

Station Trader Assignments currently restocks a home station's normal
production buy offers (`find_buy_offer buyer="$homestation"`). It never
looks at a station's **Build Storage** — the separate resource pool a
station under construction (initial build, or a queued module addition)
needs filled to keep building. Build Storage is a distinct object from the
finished station in X4's economy (visible in-game as a separate "trade
with build storage" option when right-clicking a station under
construction), which is exactly why the existing query misses it. Other
X4 mods (RandomTrade, Supply Needs Planner) exist specifically to fill
this gap, confirming it isn't already covered by this mod's auto-detect.

## Goal

Extend the mod so an assigned ship also resupplies a home station's Build
Storage when one is present, alongside its existing production-ware
resupply behavior.

## New parameters

Added to the existing `<params>` block, matching the style of `balanced`
and `fillcargo`:

- **`manageconstruction`** (bool, default `true`) — "Also resupply Build
  Storage." Master on/off switch for this feature.
- **`buildstoragefirst`** (bool, default `true`) — "Service Build Storage
  before production wares." Decides which category gets first claim on
  cargo space and budget when both are competing within the same pass.

## Detection & filtering

- For each home station, check whether it currently has an active Build
  Storage (station under construction or mid-module-addition). If present
  and `manageconstruction` is on, query its buy offers the same way
  production buy offers are queried today — against the build-storage
  object instead of the finished station.
- The existing Ware Priority List filter applies identically to both
  categories: empty list = auto-manage everything either side wants;
  populated list = only buy wares that appear in it, whether the demand
  came from Build Storage or from the station's own production buy
  offers. This keeps one mental model for "what this ship is allowed to
  buy" instead of a second, separate list just for construction.

## Ordering

Build Storage's wanted offers and the station's own production-wanted
offers each run as an independent pass through the *existing* Priority /
Balanced / Fill-Cargo-Before-Returning logic, unmodified — just scoped to
one category's offer list at a time. The two passes run back-to-back per
home station, in the order set by `buildstoragefirst`.

This is what makes "first" meaningful: if cargo space or credit budget
runs out partway through a pass, whichever category is configured to run
first gets serviced in full before the second category gets whatever is
left. A simple append-order tweak into one shared offer list would not
guarantee this under every mode (e.g. auto-detect Priority mode re-sorts
by stock level, which would ignore append order), so both categories go
through the full existing pipeline independently.

Implementation reuses the current ~275-line buying pipeline for both
passes rather than duplicating it inline — the concrete mechanism (X4
aiscript's local library / reusable `actions` block support, referenced
the same way `GetBlacklistgroup` is referenced today) is a plan/
implementation-time detail, not a design-time one.

## Debug log & Logbook

- Debug trace lines (`debugtrace`) and Logbook entries (`enablelogbook`)
  get a "(Build Storage)" style tag when the delivery target was
  construction demand, so troubleshooting output still distinguishes the
  two without extra effort.

## Docs

- `docs/CONFIGURATION.md` gains rows for `manageconstruction` and
  `buildstoragefirst` in the parameter reference table, a short paragraph
  explaining Build Storage behavior (in the style of the existing "Priority
  vs Balanced" / "Fill Cargo" explainer sections), and a troubleshooting
  entry.

## Versioning

- Bump the aiscript `version` attribute.
- Add a `<patch sinceversion="N">` block defaulting both new params to
  `true` for ships already assigned this order in existing saves —
  matching the existing pattern used for `fillcargo`, `debugtrace`, etc.

## Known risk (flagged, not hidden)

Build Storage being a distinct object from the finished station is
confirmed by external research (in-game UI behavior, and the existence of
other mods solving this exact gap). What is **not** independently
verified is the exact X4 aiscript property/query needed to reach that
object from a `<find_buy_offer>` call (best guess: something like
`$homestation.buildstorage`, used as the `buyer` on the same
`find_buy_offer` action already in use). This cannot be confirmed without
running the game.

**Required verification step**: after implementation, test in-game with
**Enable Debug Log** on against a station actually under construction,
and confirm buy offers are actually found for its Build Storage. If the
initial query syntax is wrong, only that one query needs correcting — the
rest of the design (params, filtering, ordering, logging, docs,
versioning) does not depend on getting this detail right on the first
try.

## Out of scope

- No separate ware list / priority order specifically for construction
  wares — the existing Ware Priority List is reused as-is (per design
  decision above).
- No UI beyond the two new bool params — no dropdown/enum widget, since
  none exists elsewhere in this mod's order parameters.
