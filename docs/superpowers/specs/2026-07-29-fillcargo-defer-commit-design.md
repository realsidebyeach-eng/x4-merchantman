# Fill Cargo Mode: Defer Trade Order Creation Until After Search — Design

## Problem

Two symptoms reported from live play:

1. A trader fills its cargo hold with a ware but only delivers a small
   percentage of it to the home station — the rest sits as stray cargo,
   frequently ending up dumped on an external buyer instead.
2. When the first ware in a station's wanted-ware list only needs a small
   amount, the trader doesn't go on to try filling the rest of its hold
   with other wares (possibly from other stops) before returning home.

## Root cause

Both buy and sell **fill-cargo** pipelines in
`aiscripts/sbe_stationtrader.xml` are structured as: loop over every
candidate ware, search for a trade partner (a search that periodically
yields via `<wait exact="1ms"/>` to avoid overloading a single frame),
create ONE leg of the trade immediately per ware inside that same loop,
defer the OTHER leg into a list, then after the whole loop finishes, walk
that list and create the deferred legs in a final batch.

Analysis of every ship's debug log in this savegame (763 buy-side fill-cargo
passes, 524 sell-side) shows **zero exceptions**: whenever a pass bought or
picked up anything, exactly one ware was ever acted on, and the final
batch step (`"fill pass done, delivering/exporting..."`) frequently never
printed at all — meaning the deferred leg for that one ware, and the
entire rest of the wanted-ware list, never got processed that pass.

The working hypothesis, supported by multiple independent signals:

- Every truncation happens only *after* a `create_trade_order` call earlier
  in the same pass, never before one.
- Direct evidence from a classic-mode ship (ICH-745) shows execution
  *does* continue past the first successful buy — it starts evaluating
  the next ware, gets as far as computing that ware's price ceiling — and
  is cut off specifically once it reaches the next seller search's
  `<wait>` call, not at the `create_trade_order` call itself.
- TaterTrade (`extensions/ws_2082610969/aiscripts/tatertrade.xml`), a
  mature, independently-developed reference mod, never attempts to queue
  more than one trade order per script cycle either — it evaluates every
  candidate, picks the single best deal, creates one order, and explicitly
  `<resume>`s elsewhere rather than looping back for a second.
- Correction from final review: the `immediate` attribute's documented
  interrupt behavior only applies when `immediate="true"` ("the order is
  inserted as the current order... it will be interrupted"). Every
  `create_trade_order` call in this file passes `immediate="false"`, where
  the documented behavior is a plain queue append with no described effect
  on the calling order. This schema text does not corroborate the
  hypothesis the way this spec originally claimed — it neither confirms
  nor rules it out. The hypothesis rests on the log evidence and the
  TaterTrade precedent above, not on this attribute's documentation.

Working theory: once a trade order is queued, the engine takes the next
available script yield point (a `<wait>`) as its opportunity to switch the
ship over to executing that order, abandoning the rest of the current
script pass. Classic mode (both buy and sell) never hits this because its
two `create_trade_order` calls for a single ware are adjacent, with no
search (and therefore no `<wait>`) between them — which is exactly why it
doesn't show this bug, and why it was never noticed until fill-cargo mode
specifically was investigated.

**This is not proven with certainty** — it's the best explanation the
available evidence supports, not a documented engine behavior. The fix
below is designed so its own in-game test is a direct test of the
hypothesis: if it works, multiple wares will show up bought/sold in one
pass for the first time in this project's history.

## Fix: split each fill-cargo pipeline into a decision phase and a commit phase

**Decision phase** (unchanged from today, except no `create_trade_order`
calls): loop over every candidate ware, compute the price ceiling/floor,
search for the trade partner (still has `<wait>` calls — nothing has been
committed yet, so an interrupt here costs nothing), pick the best match,
compute the clamped amount. Instead of creating the first leg's trade
order immediately, record the full decision (the partner offer object,
the amount, the price, the partner's name) into a list — the same
defer-into-a-list pattern this file already uses for the second leg today.

**Commit phase** (new): once every ware has been decided, walk the
recorded decisions and create every trade order for every ware, in **two
separate passes** — every first-leg order for every ware, then every
second-leg order for every ware — with no search, and therefore no
`<wait>`, anywhere in this phase. If the interrupt really is tied to
hitting a `<wait>` after a trade order exists, this phase never gives it
the opportunity.

Correction from final review: an earlier version of this design paired
each ware's two legs immediately adjacent to each other in a single loop
(leg1(w1), leg2(w1), leg1(w2), leg2(w2), ...). Since `create_trade_order`
appends to the ship's order queue in creation order, that produces one
round trip per ware — fly to partner1, back home, to partner2, back home
again — which defeats the entire point of this fix. The two-pass split
(leg1(w1)...leg1(wN), then leg2(w1)...leg2(wN)) produces the actually
intended queue: one tour visiting every partner, then one trip home.

Applies to both pipelines:

- **Buy fill-cargo** (`aiscripts/sbe_stationtrader.xml:588-717` currently):
  defer the buy leg (`create_trade_order name="$bestseller"...`, currently
  line 676) into a new list (the seller offer object itself, since the
  commit phase needs a live reference to create the order against) instead
  of creating it inside the decision loop. The deliver leg (currently
  created in the existing final batch loop, line 705) moves into its own
  second commit-phase loop, run only after every buy leg for every decided
  ware has already been created.
- **Sell fill-cargo** (`aiscripts/sbe_stationtrader.xml:294-406`
  currently): defer the pickup leg (`create_trade_order
  name="$sellwantedoffer"...`, currently line 369) the same way. The
  export leg (currently line 396) likewise moves into its own second
  commit-phase loop, run only after every pickup leg for every decided
  ware has already been created.

Both pipelines also capture the ware macro itself (not just the offer
object) into a dedicated list at decision time (`$decidedwarenames`
buy-side, `$selldecidedwarenames` sell-side). The second commit-phase loop
needs the ware for its Logbook entry and (buy-side) its storage clamp, but
by then the first loop has already consumed the offer object via
`create_trade_order` — reading `.ware` off it at that point would be this
file's recurring read-after-consumption bug (already fixed twice
elsewhere in its history). Capturing the ware separately, before any
consumption, avoids a third instance.

Classic mode (both buy and sell) is untouched — it already only handles
one ware per pass by design and has no batching premise to protect.

## Risk and the verification gate

This is not a small variable fix like the last three — it's a structural
change to a feature (`fillcargo` param, default on) that has apparently
never worked as designed since it was added. The mechanism is a
well-evidenced hypothesis, not a certainty.

The implementation plan must treat in-game verification as load-bearing,
not a formality: deploy, then check the debug log for the first time in
this project's history a fill-cargo pass shows **2 or more** wares
bought/picked-up before its `"fill pass done"` line. If that happens, the
hypothesis is confirmed and the fix works. If every pass still caps at
exactly one ware, the hypothesis is wrong (or incomplete) and the fallback
(make fill-cargo behave like classic mode: one ware, immediate delivery,
rely on pass repetition — discussed and set aside during design) should be
revisited rather than attempting further variations on this same
approach.

## Scope

Touches only the two fill-cargo pipelines' internal structure. Does not
change:

- Classic mode (either direction).
- The stray-cargo-to-home or stray-cargo-to-external-buyer fallbacks.
- The price-floor/price-ceiling decision logic itself — only *when* the
  resulting trade orders get created, not which ware/partner gets picked
  or at what price.
- The storage-capacity checks (`$querybuyer.cargo.{...}.free`,
  version-18) — reused as-is in the commit phase.
- No new param, no player-facing toggle.

## Out of scope

- Root-causing the *exact* engine mechanism with certainty — not
  achievable without official documentation or a build with engine-level
  tracing this project doesn't have access to. The fix is designed to be
  self-verifying instead.
- The classic-mode fallback redesign (Option B from the design
  discussion) — only pursued if this fix's in-game verification shows the
  hypothesis was wrong.

## Known follow-up (flagged by final review, not fixed here)

The buy-side deliver leg's storage clamp
(`$querybuyer.cargo.{$decidedwarenames.{$d2}}.free`) is re-evaluated once
per decided ware in the commit phase's second loop, but nothing executes
between the queued orders in that loop — so it reads the *same* live
storage figure for every ware, never accounting for an earlier ware in
the same batch also being delivered to the same shared storage pool. If
two decided wares in one pass share a storage type and together exceed
the buyer's real free space, both get clamped independently against the
same (stale-by-the-second-ware) number, and the second delivery can be
short — landing the excess back on the stray-cargo fallback. This was
unreachable before this fix (fill-cargo capped at one ware per pass) and
is a direct instance of this spec's own Problem #1 ("only delivers a
small percentage... the rest sits as stray cargo"), now reachable via a
different path than the one this fix closes.

Not fixed in this branch — the storage-capacity check's own logic is
explicitly reused as-is per this spec's Scope section. The right shape for
a follow-up is a running decrement in the decision phase, mirroring how
`$remainingvolume`/`$remainingbudget` already track consumption across
wares in the same pass: seed a `$querybuyerfree` scalar once from
`$querybuyer.cargo.{...}.free` per storage type, subtract each accepted
`$amount` as it's decided, and clamp against that running value instead
of a fresh (and, within one pass, unchanging) read. Task 3's in-game
verification should watch specifically for this symptom (a `committing
2`+ pass where a later ware's home delivery comes up short) so it isn't
mistaken for a new regression when it's actually this known, pre-existing
gap.

**Related, separate gap (documented, also not fixed):** this same
two-pass commit-phase ordering also means multi-ware passes don't benefit
from X4's own engine-level automatic leg-pairing failure-cancellation
(single-ware passes and classic mode do, automatically). See the
"Multi-ware fill-cargo passes don't get the engine's automatic
leg-pairing safety net" subsection in `docs/X4_AISCRIPT_NOTES.md` for the
full mechanism and citation, and
`docs/superpowers/specs/2026-08-09-internal-flag-and-leg-pairing-doc-design.md`
for why this is an accepted tradeoff rather than something to fix.

## Versioning

Bump the aiscript `version` attribute (currently 18, so 19).
