# Design: `internal="true"` flag + leg-pairing adjacency documentation

**Date:** 2026-08-09
**Status:** Approved for planning

## Background

While researching X4 aiscript idioms (extracting Egosoft's own official
trade scripts from the base game's packed catalogs via `x4cat`), two
things surfaced by comparing this mod's `create_trade_order` usage
against the official `order.trade.routine.xml`, `order.trade.middleman.xml`,
and `order.trade.perform.xml`:

1. This mod's ~10 `create_trade_order` call sites never set `internal=`.
   The official scripts always set it for script-decided orders. Per
   schema (`common.xsd:24093-24098`): *"Was the trade order created by
   another order? (optional, defaults false)"*.
2. `order.trade.perform.xml` — the executor order every `create_trade_order`
   call actually queues, regardless of who created it — contains a
   generic, queue-position-based mechanism that cancels a failed trade
   leg's immediate successor order if it looks like the other half of a
   buy/sell pair (`order.trade.perform.xml:558-566`). This mod's v19
   fill-cargo fix does not currently benefit from this for multi-ware
   passes; see [Part 2](#part-2-leg-pairing-adjacency-verify-and-document)
   below.

Full research trail: see project memory `project_create_trade_order_wait_interrupt`,
`reference_x4cat_extraction`, and `docs/X4_AISCRIPT_NOTES.md`.

## Goals

1. Add `internal="true"` to every `create_trade_order` call in
   `aiscripts/sbe_stationtrader.xml`, to pick up the engine's built-in
   recurring-failure cooldown/backoff for auto-generated trade orders.
2. Verify, precisely, what queue order this mod's commit-phase code
   actually produces for (a) classic mode / single-ware fill-cargo
   passes and (b) multi-ware fill-cargo passes — then document the
   resulting leg-pairing/auto-cancellation behavior clearly, so a future
   reader (including a future session) doesn't have to re-derive it.

## Non-goals

- **Do not restructure the commit phase to pair buy+deliver (or
  pickup+export) legs adjacently.** That was the exact shape of the bug
  fixed in the v19 fill-cargo defer-commit design (produces one round
  trip per ware instead of one filling tour). This design explicitly
  keeps the existing two-pass loop ordering (all first-leg orders, then
  all second-leg orders) unchanged.
- Do not build custom leg-failure detection or cancellation logic. There
  is no confirmed real-world evidence (debug logs or in-game reports) of
  a leg actually failing and stranding its paired leg. Building
  cross-pass failure-tracking machinery to solve an unconfirmed problem
  is out of scope (YAGNI) — see [[feedback-storage-check-pitfalls]] on
  the risk of speculative changes to this timing-sensitive script.
- Do not add blacklist-related config, UI, or logic changes.
  `internal="true"`'s blacklist-enforcement effect is redundant with
  this mod's own upstream checks (`$blacklistgroup` /
  `match_use_blacklist` / sector `isblacklisted` calls at
  `sbe_stationtrader.xml:79-80,145,203,323,435,639,772`), which run
  during the search phase — before a candidate offer is ever considered
  — and are strictly better UX than an order-level rejection after the
  ship has already committed to the trip. Those checks are unchanged by
  this design.

## Part 1: `internal="true"`

Add `internal="true"` to all `create_trade_order` calls (buy pipeline,
sell pipeline, stray-cargo handling, classic mode — both buy and sell).
Every call site in this mod is emitted by the script's own decision
logic, never a manual/player-initiated order, so `internal="true"` is
correct uniformly across every site. This is a pure attribute addition —
no other logic, control flow, or variable changes.

**What this actually changes, per the official source:**
- `set_order_failed order="..." text="$failuretext" recurring="$internalorder"`
  (`order.trade.perform.xml:535`) and the matching
  `clear_recurring_order_failure` on success
  (`order.trade.perform.xml:573-575`) only apply their cooldown/failure
  bookkeeping when `internalorder` is true. This mod currently has zero
  retry-limiting logic of its own (confirmed: no `failed`/`failure`/
  `retry`/`cooldown` handling exists anywhere in
  `sbe_stationtrader.xml` today) — so this closes a real, previously
  totally-absent gap, at effectively zero implementation risk, since the
  cooldown machinery lives entirely inside the engine's own trade-order
  execution, not in this mod's own script logic.
- Blacklist enforcement gated by `$internalorder` elsewhere in
  `order.trade.perform.xml` is **not** relied upon by this design — see
  Non-goals above.

## Part 2: leg-pairing adjacency (verify and document)

### What the official mechanism actually does

`order.trade.perform.xml:558-566`:

```xml
<!-- if this is the buy of a buy-sell pair,
      and it failed,
      and we have none of the ware needed for the sell,
      cancel the sell. -->
<do_if value="(@this.assignedcontrolled.nextorder.id == 'TradePerform')
              and @this.assignedcontrolled.nextorder.$tradedeal.buyer.exists
              and not this.assignedcontrolled.cargo.{$tradedeal.ware}.exists">
  <cancel_order order="this.assignedcontrolled.nextorder"/>
</do_if>
```

This is **generic and queue-position-based**: it checks whether the ship's
own *very next queued order* is also a `TradePerform` order with a valid
buyer, and whether the ware never actually made it into cargo. It is not
gated by `internal=`, and it does not depend on which script created
either order — `create_trade_order` always queues a `TradePerform` order
regardless of caller, so this logic runs identically for orders this mod
creates as for orders `order.trade.routine.xml` creates.

**Consequence: this protection already applies, automatically, for free,
whenever this mod queues two trade legs back-to-back with nothing
between them in the ship's queue** — no code change needed for that
case.

### Why multi-ware fill-cargo passes don't get it

The v19 commit phase (both buy and sell pipelines) deliberately uses
**two separate loops**: all first-leg orders (every decided ware's
buy/pickup), then all second-leg orders (every decided ware's
deliver/export) — see `project_create_trade_order_wait_interrupt` memory
for why (pairing them adjacently produces one round trip per ware
instead of one filling tour).

For a pass that decides exactly one ware, this still produces
`leg1(w), leg2(w)` adjacently — full protection, automatically. For a
pass that decides N ≥ 2 wares, the queue order is
`leg1(w1), leg1(w2), ..., leg1(wN), leg2(w1), leg2(w2), ..., leg2(wN)` —
`leg1(w1)`'s `nextorder` is `leg1(w2)`, not `leg2(w1)`, so the adjacency
check never matches. A failed first leg in a multi-ware pass will not
trigger automatic cancellation of its paired second leg under the
current architecture.

**This is an accepted, documented tradeoff, not a bug to fix** — per
the Non-goals section, restoring adjacency to regain this protection
would reintroduce the exact bug v19 fixed. Classic mode (always exactly
one ware per pass) is unaffected and always gets the protection.

### Work for this part

1. Re-read the current buy and sell commit-phase loops in
   `aiscripts/sbe_stationtrader.xml` (buy pipeline around the fill-cargo
   commit section, sell pipeline mirror) to confirm the queue-order
   claim above still matches the shipped v19 code exactly (variable
   names / loop structure may have shifted since the summary this design
   was written from — verify against the live file, not memory).
2. Add a new subsection to `docs/X4_AISCRIPT_NOTES.md`'s
   `create_trade_order` section documenting: the `nextorder`
   auto-cancellation mechanism (with citation), that it's automatic and
   `internal=`-independent, and the multi-ware adjacency gap with the
   explicit reasoning for not fixing it.
3. Add a short cross-reference note to the existing "Known follow-up"
   section of `docs/superpowers/specs/2026-07-29-fillcargo-defer-commit-design.md`
   pointing at the new `X4_AISCRIPT_NOTES.md` subsection, so the
   tradeoff is discoverable from the spec that created it, not just from
   the new notes file.

No changes to `aiscripts/sbe_stationtrader.xml` logic for this part —
documentation only.

## Verification

- `xmllint --noout aiscripts/sbe_stationtrader.xml` after Part 1's edit.
- Real schema validation against the cached `aiscripts.xsd`/`common.xsd`
  (per `reference_aiscripts_schema` memory) to confirm `internal=` is
  accepted on every call site's action.
- Manual diff review confirming all ~10 `create_trade_order` call sites
  gained `internal="true"` and nothing else changed.
- Part 2 has no runtime behavior to verify (documentation only) — verify
  by re-reading the shipped commit-phase code, not by inference.
- Live in-game confirmation of Part 1's behavioral effect (fewer
  repeated identical failed-order attempts in the debug log) is
  aspirational, not a blocking gate for this plan — same pattern as
  prior fixes in this project, since I cannot run the game myself.

## Rollout

Version bump: `20` → `21` (root `<aiscript ... version="N">`). Updated
from this spec's original `19` → `20` after merging `main`, which had
independently shipped the `mincargopercent` feature at version `20` in
the meantime — `20` is no longer available. No `<patch sinceversion="21">`
upgrade block needed — precedent from v16-19 (storage-clamp/scope fixes,
also pure internal-logic changes with no param/structural changes) shows
attribute-only or logic-only changes don't require one; only
param-structure changes have historically needed a patch block (see
existing blocks at versions 2,3,4,8,11,12,20). `content.xml`'s version
stays at `1` per established convention.
