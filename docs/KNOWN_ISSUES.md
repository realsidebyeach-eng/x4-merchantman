# Known Issues

A catalog of every bug this project has fixed, organized by **pattern
class** — the recurring root-cause shape, not chronological order. This
is a living document: every future fix's implementation plan should add
its own entry here (new instance of an existing class, or a new class if
it doesn't fit one).

**How to use this:** before designing a fix that touches trade-order
creation, storage/capacity checks, or fill-cargo commit logic, skim the
relevant pattern class below — most new bugs in this file turn out to be
a fresh instance of a class already listed, not something new. Copy the
"How to spot it" checks into implementation-plan verification steps and
reviewer dispatch prompts.

See also `docs/TEST_CASES.md` for the matching in-game repro recipes, and
`docs/X4_AISCRIPT_NOTES.md` for the underlying X4 aiscript action
semantics these bugs turned on.

---

## Pattern Class 1: Read-after-consumption

**Rule:** `create_trade_order` (and similarly-shaped actions) consume/
invalidate the offer variable passed as `name=`/`tradeoffer=` — any
property read off that same variable *after* the call (`.buyer`,
`.seller`, `.ware`) returns null.

**Why it matters:** the failure is silent — no error, just a null/blank
value that shows up as "from null" or a missing field in a Logbook entry
or debug line. It's cheap to introduce by accident when adding a new
debug/log line near an existing `create_trade_order` call, and it has
recurred inside code written specifically to fix something unrelated.

**Instances:**

| # | What broke | Fix | Commit(s) | Spec/plan |
|---|---|---|---|---|
| 1a | Logbook entries showed seller as `null`, station names as raw hex IDs | Capture `.knownname` into a plain variable before the consuming call; use `.knownname` for all station substitutions | `449b455` | none (pre-spec era) |
| 1b | Stray-cargo external-buyer debug log always showed "selling ... to null" | Capture `$beststraybuyername` before the call | `6ec354a`, `d903112`, `b6d1553` | `specs/2026-07-24-stray-buyer-null-name-and-blacklist-doc.md` |
| 1c | Fill-cargo batch delivery Logbook would have shown null ware names | Resolved incidentally by v19's `$decidedwarenames`, captured at decision time specifically to avoid this | `1916877` (v19, see Class 5) | `specs/2026-07-29-fillcargo-defer-commit-design.md` |

**How to spot it:** grep `\.ware\b|\.buyer\b|\.seller\b` on every line
*after* a `create_trade_order` call in the same block. Re-check this
specifically whenever *relocating* an existing call (deferring it,
splitting a loop, reordering passes) — moving a consuming call relative
to other reads is exactly how instance 1c's near-miss and the v19 sell-
side commit-phase bug (caught in final review, never shipped broken)
happened.

---

## Pattern Class 2: Wrong-object-scope

**Rule:** A check targets the wrong object — a nominal/default object
instead of the actual one relevant to the current pass (e.g. the base
station instead of its Build Storage, which is the real buyer for that
category).

**Why it matters:** this mod services more than one buyer/seller category
through shared loop code (production wares vs. Build Storage construction
wares). A check that's correct for the common case can silently break the
other category while looking completely fine in review, since nothing
about the code is syntactically wrong — just semantically pointed at the
wrong object.

**Instances:**

| # | What broke | Fix | Commit(s) | Spec/plan |
|---|---|---|---|---|
| 2a | Contested/pirate sectors excluded valid, dockable stations | Removed the redundant sector-ownership pre-filter; rely solely on the existing per-station `match_relation_to relation="dock"` check | `1b72988` | none (pre-spec era) |
| 2b | Every Build Storage buy clamped to 0 — a ship under construction with a 100M credit budget never bought anything | Replaced hardcoded `$homestation.cargo.{ware}.free` with `$querybuyer.cargo.{ware}.free` at all 3 storage-check sites | `32b8ca1`, `e51efda`, `47c740f` | `specs/2026-07-28-buildstorage-querybuyer-scope-fix.md` |

**How to spot it:** whenever a check reads `.cargo`, `.money`, or any
capacity/affordability property, verify it's reading the *actual* buyer/
seller object for the current pass/category (`$querybuyer`), not a
hardcoded assumption like `$homestation`. Instance 2b is the second of a
three-bug regression chain (see Class 3, instance 3c) on the exact same 3
code sites — any future edit to those sites should regression-test both
the production category *and* the Build Storage category, since a fix
correct for one has twice broken the other.

---

## Pattern Class 3: Money/storage entanglement & wrong funding source

**Rule:** A check or funding path meant to be pure storage/capacity gets
tangled with affordability/price, or silently draws from the wrong
account — most often because "free" (`price="0"`) legs get evaluated by
logic that assumes a real price.

**Instances:**

| # | What broke | Fix | Commit(s) | Spec/plan |
|---|---|---|---|---|
| 3a | Ships got permanently stuck holding cargo — home station paying itself a second time for already-owned wares could fail | `price="0"` on every delivery-to-home-station order | `1a38c82`, `2da76f2`, `d5f2f4b` | `specs/2026-07-13-free-home-delivery-design.md` |
| 3b | Traders appeared to do nothing until credits were added to their home station's own account | Deleted a dead-but-reachable commander-money override branch; budget is unconditionally `player.money` | `bf95d5e`, `3202358`, `42600cf` | `specs/2026-07-24-trader-wallet-budget-fix.md` |
| 3c | Ships sat stationary creating no orders at all when their home station's balance was low, even for free (`price="0"`) deliveries | Removed `clamp_trade_amount` (which factors in buyer affordability per schema) for all 3 storage-check sites; replaced with a plain `.cargo.{ware}.free` term | `153f1a7`, `90712d1`, `8a6dab0` | `specs/2026-07-24-storage-clamp-money-regression-fix.md` |

**How to spot it:** for any check gating a trade on "does the destination
have room" or "can the destination afford this," verify separately:
(a) is it reading the correct object's storage (see Class 2), and (b)
does it accidentally also gate on money/price for a leg that executes at
`price="0"`? `clamp_trade_amount`'s own schema documentation
(`common.xsd:22047`) states its result is clamped to what the buyer "can
store **and afford**" — unconditionally, with no price=0 exception. See
`docs/X4_AISCRIPT_NOTES.md` for the full citation. 3b's root cause traces
back to singleton #1 below — a case where removing a feature (commander
assignment) but not its dependent logic left dead code that reactivated
later.

---

## Pattern Class 4: Off-by-one / boundary math

**Rule:** A numeric guard, index, or threshold is wrong at a boundary —
list indexing convention, a cap formula that only holds for some starting
values, or a floor that isn't actually enforced.

**Instances:**

| # | What broke | Fix | Commit(s) | Spec/plan |
|---|---|---|---|---|
| 4a | `$home.{0}` / `$homesell.{0}` silently returned null | X4 aiscript lists are 1-indexed, not 0-indexed; corrected to `{1}` | `38ffbbc` | none (pre-spec era) |
| 4b | An early draft of the v19 fix's order-count guard (`lt 6`) could let total orders reach 7 for an odd starting count | Changed to `le 4` (holds for every starting count) — caught in final review before shipping | `2e82e4b` | `specs/2026-07-29-fillcargo-defer-commit-design.md` |
| 4c | A later, separate feature's `$mincargovolume` guard could be weakened to 0 by `mincargopercent=0` or a null capacity read, below the pre-existing `gt 0` floor | Floored with `[$cargocapacity*$mincargopercent/100, 1].max` — caught in final review | `223d206` (on a sibling worktree branch, not this branch's ancestry — noted for completeness) | `plans/2026-08-07-buyfill-min-cargo-percent.md` (sibling branch) |

**How to spot it:** for any guard involving `+`/`*`/list index math, test
it explicitly against boundary inputs (odd vs. even starting counts, 0,
the max/min of a percent range) rather than just the "typical" case —
instances 4b and 4c were both caught specifically by deliberately testing
boundary values during final review, not by the original implementation
pass.

---

## Pattern Class 5: Wait-interrupt / order-queue-loses-control

**Rule:** Once `create_trade_order` queues an order, the calling script
cannot reliably continue running further logic in the same pass — the
engine appears to take the next `<wait>` yield point as its cue to switch
the ship over to executing the newly-queued order, abandoning the rest of
the pass (any remaining loop iterations, any code after that point). This
applies to any infinite default-behaviour order (like this mod's own) once
anything is queued behind it, independent of the `immediate=` value —
**confirmed directly from Egosoft's own official source** (see
`docs/X4_AISCRIPT_NOTES.md`), not just this project's own empirical
testing.

**Instances:**

| # | What broke | Fix | Commit(s) | Spec/plan |
|---|---|---|---|---|
| 5a | Fill-cargo capped at exactly one ware per pass — cargo filled but only a fraction ever delivered home; other wanted wares never chased | Split into a decision phase (search/compute only, zero order creation) and a commit phase (all order creation, zero `<wait>`s, two separate passes — all first-leg orders, then all second-leg orders) | `ef7bb26`, `e7ad55a`, `d320d47`, `b114687`, `1916877`, `2e82e4b` | `specs/2026-07-29-fillcargo-defer-commit-design.md` |

**How to spot it:** any code pattern that searches, creates a trade
order, and then tries to keep searching/deciding in the same pass will
silently truncate after the first order. **Critical sub-pattern found
only during final review, not the original implementation:** naively
pairing each ware's two legs adjacently in the commit phase (`leg1(w1),
leg2(w1), leg1(w2), leg2(w2)...`) avoids the wait-interrupt bug but
produces one round trip per ware instead of one filling tour, since
`create_trade_order` appends to the queue in creation order — must be two
separate loops (all first legs, then all second legs) instead.

---

## Pattern Class 6: False-positive "handled" completion flags

**Rule:** A completion/handled flag gets set based on an offer merely
*existing* or being *available*, rather than on the actual effect (amount
moved) being non-zero.

**Instances:**

| # | What broke | Fix | Commit(s) | Spec/plan |
|---|---|---|---|---|
| 6a | Stray cargo could get permanently stuck "delivering" to a home station with 0 real capacity, repeating an identical zero-progress log line every pass | Compute the actual clamped order amount first; only create the order and mark handled when that amount is `gt 0`; log the actual clamped amount, not the ship's full held quantity | `8d2c794`, `20b129d`, `8ab91af` | `specs/2026-07-24-storage-capacity-checks.md` |

**How to spot it:** any `set_value name="$xhandled" exact="true"` (or
similar) should be conditioned on the actual amount used in the
resulting order being non-zero, not just on an offer's `.available`/
`.exists` flag — and any debug/log line reporting "amount" should log the
variable actually used in the trade order, not a pre-clamp quantity.

---

## Pattern Class 7: Stale documentation after a late-stage fix

**Rule:** A design spec or implementation plan's own prescriptive text
(a "Fix" section, a code block, a claimed guarantee) falls out of sync
with the code actually shipped, usually because a final-review pass
corrected the code after the docs were already written.

**Instances:**

| # | What went stale | Fix | Commit(s) |
|---|---|---|---|
| — | Buy/sell-roles design assumed a shared-library extraction the real schema doesn't support (`<attention><actions>` has no include mechanism, unlike `<interrupts>`) | Corrected the design to inline duplication *before* implementation | `81fa52d` |
| — | Fillcargo-defer-commit spec overstated what the schema's `immediate` attribute doc actually proved, and its plan showed the wrong (pre-fix) order-guard formula and paired-legs code blocks | Corrected spec reasoning and replaced plan's stale code blocks with the actual shipped code (verified byte-identical) | `2e82e4b`, `41d207e` |
| — | An in-code XML comment overstated what the storage clamp actually guarantees | Corrected the comment | `86afb7a` |

**How to spot it:** after any fix to the storage-check sites (Class 2/3)
or the fill-cargo commit-phase structure (Class 5), diff the shipped
code's own in-line comments and the spec's stated guarantees against what
actually shipped — don't assume the spec/plan is still accurate just
because the code review passed.

---

## Open / deferred — not yet fixed

These are known gaps or accepted tradeoffs, not resolved bugs. Don't
treat them as regressions if they surface in play; do consult them before
touching the related code.

- **Stale storage read across multiple wares in one commit-phase batch.**
  The v19 commit phase's deliver-leg storage clamp reads
  `$querybuyer.cargo.{ware}.free` fresh per ware, but nothing executes
  between queued orders in that loop, so it reads the same live figure
  for every ware in the batch — doesn't account for two decided wares
  sharing a storage pool within one pass. Proposed fix (not implemented):
  a running `$querybuyerfree` decrement, mirroring `$remainingvolume`/
  `$remainingbudget`. See `specs/2026-07-29-fillcargo-defer-commit-design.md`
  ("Known follow-up").
- **Multi-ware fill-cargo passes don't get the engine's automatic
  leg-pairing safety net.** X4's own trade-order executor auto-cancels a
  failed leg's immediate successor order if it looks like the other half
  of a pair — but only works when the two legs are queue-adjacent, which
  v19's two-pass commit ordering breaks for multi-ware passes (by design
  — restoring adjacency would reintroduce the Class 5 bug). No confirmed
  real-world instance of this actually stranding a leg; not being built
  speculatively. See
  `specs/2026-08-09-internal-flag-and-leg-pairing-doc-design.md` and
  `docs/X4_AISCRIPT_NOTES.md`'s "Multi-ware fill-cargo passes don't get
  the engine's automatic leg-pairing safety net" subsection.
- **Stray-cargo priority (home-first vs. external-buyer-first) was
  designed but never merged.** A separate, orphaned branch
  (`worktree-stray-buyer-null-name`) designed swapping stray-cargo
  priority to try an external buyer before the free home delivery, since
  Enable Selling's pickup/export cycle made home-first routing
  self-defeating (free delivery vs. real revenue). Confirmed against the
  current shipped code: still tries home first. Status: identified,
  designed, not merged.

## Not a bug — documented limitation

- **A ship's automated flight path can still transit a blacklisted sector
  as a waypoint**, even though the mod correctly avoids *choosing*
  blacklisted sectors as destinations. Confirmed structurally impossible
  to fix: the mod never issues movement commands itself
  (`create_trade_order` hands flight to X4's standard trade-order AI), and
  no route-safety/avoidance action exists anywhere in the X4 aiscript
  schema. Documented in `README.md`; the only mitigation is lowering Max
  Jump Range. See `specs/2026-07-24-stray-buyer-null-name-and-blacklist-doc.md`.
- **Home-station delivery legs are now blacklist-screened, and can
  silently abort.** Adding `internal="true"` to every `create_trade_order`
  call (2026-08-09/10) made the engine's own `$internalorder`-gated
  blacklist check (`order.trade.perform.xml:156,300`,
  `TRADEPARTNER_BLACKLISTED`) live for every leg, including deliveries to
  a ship's own home station — previously unscreened, since this mod's own
  upstream checks (`match_use_blacklist`/sector `isblacklisted`) only ever
  screen external search-space partners, not the user's own explicit
  home-station selection. **Accepted as intentional**: honoring the
  player's own blacklist against their own station is defensible default
  behavior. If a player blacklists a sector containing one of their own
  home stations, delivery legs to that station will now silently abort
  and the ship falls back to stray-cargo handling instead. See
  `docs/X4_AISCRIPT_NOTES.md`'s "What `internal=\"true\"` actually does"
  subsection for the full mechanism.

## Singletons

Fixes that don't fit a recurring pattern (yet) — still worth recording in
case a second instance shows up later.

- **Ship silently auto-assigned to home station's Defence group**
  (`59cdce0`) — a raw `set_object_commander` call outside the normal
  Role-assignment UI flow defaults into Defence; fixed by removing the
  assignment. Historical root of instance 3b above — the assignment was
  removed but the dependent budget-override logic was left behind.
- **Doubled `%%` rendered literally in param display text** (`6790afb`)
  — X4's text system doesn't need `%%` escaping for a literal percent
  sign (only special before a digit, for `%1`-style substitution).
- **A field named "Max Discount %" actually behaved as a required
  minimum** (`97e2f38`) — renamed to "Min Discount %" (its real meaning)
  and a genuine new "Max Discount %" ceiling was added.
- **Install docs pointed at the wrong extensions path** (`bffbe1f`) —
  corrected to the actual Steam library `extensions/` folder other
  installed mods use.
