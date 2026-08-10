# Internal Flag + Leg-Pairing Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `internal="true"` to every `create_trade_order` call in `aiscripts/sbe_stationtrader.xml` (picking up the engine's built-in recurring-failure cooldown/backoff), and document — without changing — the fact that multi-ware fill-cargo passes don't get the engine's automatic leg-pairing failure-cancellation safety net.

**Architecture:** Two independent tasks. Task 1 is a pure attribute addition across all 10 `create_trade_order` call sites plus a version bump — no control-flow, variable, or logic changes. Task 2 is documentation-only (two doc files), gated on first re-confirming the queue-ordering claim against the live file, since documentation making a factual claim about the code must be verified against the actual code, not assumed from the spec.

**Tech Stack:** X4: Foundations aiscript XML. No automated test framework exists — `xmllint --noout` plus real schema validation against the cached `aiscripts.xsd`/`common.xsd` is the verification substitute for both tasks.

## Global Constraints

- Full spec: `docs/superpowers/specs/2026-08-09-internal-flag-and-leg-pairing-doc-design.md`.
- Do NOT restructure the commit-phase loop ordering (the existing two-pass "all first-leg orders, then all second-leg orders" structure). Restoring adjacent buy+deliver pairing would reintroduce the one-round-trip-per-ware bug fixed in the v19 fill-cargo defer-commit design (see `docs/KNOWN_ISSUES.md` Pattern Class 5).
- Do NOT build any custom leg-failure detection or cancellation logic. No confirmed real-world evidence this is needed.
- Do NOT add any blacklist-related config, UI, or logic. This mod's own blacklist enforcement (`$blacklistgroup`/`match_use_blacklist`/sector `isblacklisted` checks) is unrelated to and unaffected by this change.
- Version bump target: `20` → `21` (this worktree branch merged `main`, which independently shipped version 20 while this branch was in progress — `20` is no longer available). No `<patch sinceversion="21">` upgrade block needed (precedent: attribute/logic-only changes at versions 13-19 needed no patch block; only param-structure changes have, at versions 2,3,4,8,11,12,20).
- Schema cache for validation: `/home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/` (`aiscripts.xsd`, `common.xsd`, `parameters.xsd`).
- Working directory for every step below: `/home/sidebyeach/Projects/X4StationTrader/.claude/worktrees/internal-flag-and-leg-cancel/` — use this absolute path (or `git -C <path>`) for every git/file operation. Do not assume a different working directory.

---

### Task 1: Add `internal="true"` to every `create_trade_order` call, bump version

**Files:**
- Modify: `aiscripts/sbe_stationtrader.xml:4` (version bump)
- Modify: `aiscripts/sbe_stationtrader.xml:197,235,410,413,505,507,730,735,847,849` (10 `create_trade_order` call sites)

**Interfaces:**
- Consumes: nothing from another task.
- Produces: nothing consumed by Task 2 (Task 2 is documentation-only and independent of this task's line-number-preserving attribute additions).

- [ ] **Step 1: Bump the aiscript version**

Current line 4:
```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="20">
```
Change `version="20"` to `version="21"`:
```xml
<aiscript name="sbe_stationtrader" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="aiscripts.xsd" version="21">
```

- [ ] **Step 2: Add `internal="true"` to the stray-cargo home-delivery call (line 197)**

Before:
```xml
												<create_trade_order name="$strayhomeoffer" object="this.object" tradeoffer="$strayhomeoffer" amount="$strayhomeamount" price="0" immediate="false"/>
```
After:
```xml
												<create_trade_order name="$strayhomeoffer" object="this.object" tradeoffer="$strayhomeoffer" amount="$strayhomeamount" price="0" immediate="false" internal="true"/>
```

- [ ] **Step 3: Add `internal="true"` to the stray-cargo external-buyer call (line 235)**

Before:
```xml
									<create_trade_order name="$beststraybuyer" object="this.object" tradeoffer="$beststraybuyer" amount="[$strayamount,$beststraybuyer.amount].min" immediate="false"/>
```
After:
```xml
									<create_trade_order name="$beststraybuyer" object="this.object" tradeoffer="$beststraybuyer" amount="[$strayamount,$beststraybuyer.amount].min" immediate="false" internal="true"/>
```

- [ ] **Step 4: Add `internal="true"` to the sell fill-cargo pickup-leg call (line 410)**

Before:
```xml
									<create_trade_order name="$selldecidedwares.{$sd}" object="this.object" tradeoffer="$selldecidedwares.{$sd}" amount="$selldecidedamounts.{$sd}" immediate="false"/>
```
After:
```xml
									<create_trade_order name="$selldecidedwares.{$sd}" object="this.object" tradeoffer="$selldecidedwares.{$sd}" amount="$selldecidedamounts.{$sd}" immediate="false" internal="true"/>
```

- [ ] **Step 5: Add `internal="true"` to the sell fill-cargo export-leg call (line 413)**

Before:
```xml
									<create_trade_order name="$selldecidedbuyers.{$sd2}" object="this.object" tradeoffer="$selldecidedbuyers.{$sd2}" amount="$selldecidedamounts.{$sd2}" immediate="false"/>
```
After:
```xml
									<create_trade_order name="$selldecidedbuyers.{$sd2}" object="this.object" tradeoffer="$selldecidedbuyers.{$sd2}" amount="$selldecidedamounts.{$sd2}" immediate="false" internal="true"/>
```

- [ ] **Step 6: Add `internal="true"` to the classic-mode sell pickup-leg call (line 505)**

Before:
```xml
												<create_trade_order name="$sellwantedoffer" object="this.object" tradeoffer="$sellwantedoffer" amount="$sellamount" immediate="false"/>
```
After:
```xml
												<create_trade_order name="$sellwantedoffer" object="this.object" tradeoffer="$sellwantedoffer" amount="$sellamount" immediate="false" internal="true"/>
```

- [ ] **Step 7: Add `internal="true"` to the classic-mode sell export-leg call (line 507)**

Before:
```xml
												<create_trade_order name="$sellbestbuyer" object="this.object" tradeoffer="$sellbestbuyer" amount="$sellamount" immediate="false"/>
```
After:
```xml
												<create_trade_order name="$sellbestbuyer" object="this.object" tradeoffer="$sellbestbuyer" amount="$sellamount" immediate="false" internal="true"/>
```

- [ ] **Step 8: Add `internal="true"` to the buy fill-cargo buy-leg call (line 730)**

Before:
```xml
								<create_trade_order name="$decidedsellers.{$d}" object="this.object" tradeoffer="$decidedsellers.{$d}" amount="$decidedamounts.{$d}" immediate="false"/>
```
After:
```xml
								<create_trade_order name="$decidedsellers.{$d}" object="this.object" tradeoffer="$decidedsellers.{$d}" amount="$decidedamounts.{$d}" immediate="false" internal="true"/>
```

- [ ] **Step 9: Add `internal="true"` to the buy fill-cargo deliver-leg call (line 735)**

Before:
```xml
								<create_trade_order name="$decidedwares.{$d2}" object="this.object" tradeoffer="$decidedwares.{$d2}" amount="$deliverclamped" price="0" immediate="false"/>
```
After:
```xml
								<create_trade_order name="$decidedwares.{$d2}" object="this.object" tradeoffer="$decidedwares.{$d2}" amount="$deliverclamped" price="0" immediate="false" internal="true"/>
```

- [ ] **Step 10: Add `internal="true"` to the classic-mode buy-leg call (line 847)**

Before:
```xml
											<create_trade_order name="$bestseller" object="this.object" tradeoffer="$bestseller" amount="$amount" immediate="false"/>
```
After:
```xml
											<create_trade_order name="$bestseller" object="this.object" tradeoffer="$bestseller" amount="$amount" immediate="false" internal="true"/>
```

- [ ] **Step 11: Add `internal="true"` to the classic-mode deliver-leg call (line 849)**

Before:
```xml
											<create_trade_order name="$wantedoffer" object="this.object" tradeoffer="$wantedoffer" amount="$amount" price="0" immediate="false"/>
```
After:
```xml
											<create_trade_order name="$wantedoffer" object="this.object" tradeoffer="$wantedoffer" amount="$amount" price="0" immediate="false" internal="true"/>
```

- [ ] **Step 12: Verify well-formedness**

Run: `xmllint --noout aiscripts/sbe_stationtrader.xml`
Expected: no output, exit code 0.

- [ ] **Step 13: Verify against the real X4 schema**

Run: `xmllint --noout --schema /home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/aiscripts.xsd aiscripts/sbe_stationtrader.xml`
Expected: `aiscripts/sbe_stationtrader.xml validates` (or equivalent success message), no schema errors — this confirms `internal=` is a legal attribute on `create_trade_order` in this exact position for every call site.

- [ ] **Step 14: Confirm exactly 10 call sites, all now with `internal="true"`, nothing else changed**

Run: `grep -n 'create_trade_order' aiscripts/sbe_stationtrader.xml`
Expected: exactly 10 lines, every one containing `internal="true"`. Then run `git diff aiscripts/sbe_stationtrader.xml` and manually confirm the only changes are: the version bump (line 4) and ` internal="true"` appended to each of the 10 call sites — no other text differs.

- [ ] **Step 15: Commit**

```bash
git add aiscripts/sbe_stationtrader.xml
git commit -m "feat: mark all create_trade_order calls internal so failed auto-generated trades get the engine's built-in retry cooldown"
```

---

### Task 2: Verify and document the leg-pairing adjacency gap

**Files:**
- Modify: `docs/X4_AISCRIPT_NOTES.md` (new subsection under `## \`create_trade_order\``, before the `## \`clamp_trade_amount\`` heading at line 106)
- Modify: `docs/superpowers/specs/2026-07-29-fillcargo-defer-commit-design.md` (append to the existing "Known follow-up" section, before the `## Versioning` heading at line 200)

**Interfaces:**
- Consumes: nothing from Task 1 (independent; Task 1's attribute additions don't shift any line numbers this task reads, since no lines are added or removed by Task 1, only attribute text appended within existing lines).
- Produces: nothing consumed elsewhere.

- [ ] **Step 1: Re-verify the queue-ordering claim against the live sell-pipeline commit phase**

Run: `sed -n '393,418p' aiscripts/sbe_stationtrader.xml` (adjust the range if Task 1 has already run and you're on a fresh checkout — search for the comment text `Every ware has been decided` if line numbers don't match)

Expected content: a comment reading *"Every ware has been decided - now create every trade order back-to-back with no searching (and so no wait) in between, so a queued order can't cut this pass short before the rest are created. Pickup legs first, then export legs..."* immediately followed by a `<do_all exact="$selldecidedwares.count" counter="$sd">` loop containing exactly one `create_trade_order` call (the pickup leg), then a second separate `<do_all exact="$selldecidedwares.count" counter="$sd2">` loop containing exactly one `create_trade_order` call (the export leg) plus the Logbook write. Confirm these are two **separate** `do_all` blocks, not one loop containing both legs.

- [ ] **Step 2: Re-verify the queue-ordering claim against the live buy-pipeline commit phase**

Run: `sed -n '711,736p' aiscripts/sbe_stationtrader.xml` (adjust range as in Step 1 if needed; search for `Every ware has been decided` if line numbers have shifted)

Expected content: same two-separate-loop structure as Step 1 — one `do_all` for all buy legs (`$decidedsellers`), a second separate `do_all` for all deliver legs (`$decidedwares`).

If either Step 1 or Step 2 shows anything other than two separate loops (e.g. a single loop containing both legs), STOP — this means the code has changed since this plan was written and the documentation below would be inaccurate. Do not proceed; report back what the actual structure is instead.

- [ ] **Step 3: Add the new subsection to `docs/X4_AISCRIPT_NOTES.md`**

Insert the following new subsection immediately after line 104 (the line reading `call.`) and before line 106 (`## \`clamp_trade_amount\``):

```markdown

### Multi-ware fill-cargo passes don't get the engine's automatic leg-pairing safety net

`order.trade.perform.xml` — the executor order every `create_trade_order`
call actually queues, regardless of who created it — contains a generic,
queue-position-based mechanism (`order.trade.perform.xml:558-566`) that
auto-cancels a failed leg's immediate successor order if it looks like
the other half of a buy/sell pair:

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

This checks whether the ship's own *very next queued order* is also a
`TradePerform` order — purely positional, not gated by `internal=`, and
not dependent on which script created either order. **This mod's
single-ware passes (classic mode, or a fill-cargo pass that only decided
one ware) already get this protection automatically, for free, with no
code change** — the buy/pickup leg and the deliver/export leg end up
adjacent in the ship's queue in that case.

**Multi-ware fill-cargo passes do not get this protection.** This mod's
v19 fill-cargo commit phase deliberately uses two separate loops — every
first-leg order for every decided ware, then every second-leg order for
every decided ware (`aiscripts/sbe_stationtrader.xml`, both the buy and
sell commit phases) — specifically to avoid producing one round trip per
ware (see `project_create_trade_order_wait_interrupt` memory / `docs/KNOWN_ISSUES.md`
Pattern Class 5). For N ≥ 2 decided wares, the queue order is `leg1(w1),
leg1(w2), ..., leg1(wN), leg2(w1), ..., leg2(wN)` — `leg1(w1)`'s
`nextorder` is `leg1(w2)`, not `leg2(w1)`, so the adjacency check never
matches.

**This is an accepted tradeoff, not a bug to fix.** Restoring adjacency
to regain this protection would reintroduce the exact one-round-trip-per-
ware bug the v19 fix corrected. No confirmed real-world evidence exists
of a leg actually failing and stranding its paired leg in a multi-ware
pass — see `docs/superpowers/specs/2026-08-09-internal-flag-and-leg-pairing-doc-design.md`
for the full design reasoning.
```

- [ ] **Step 4: Add the cross-reference note to the fillcargo-defer-commit-design spec's "Known follow-up" section**

Insert the following paragraph immediately after line 198 (the line ending `gap.`) and before line 200 (`## Versioning`) in `docs/superpowers/specs/2026-07-29-fillcargo-defer-commit-design.md`:

```markdown

**Related, separate gap (documented, also not fixed):** this same
two-pass commit-phase ordering also means multi-ware passes don't benefit
from X4's own engine-level automatic leg-pairing failure-cancellation
(single-ware passes and classic mode do, automatically). See the
"Multi-ware fill-cargo passes don't get the engine's automatic
leg-pairing safety net" subsection in `docs/X4_AISCRIPT_NOTES.md` for the
full mechanism and citation, and
`docs/superpowers/specs/2026-08-09-internal-flag-and-leg-pairing-doc-design.md`
for why this is an accepted tradeoff rather than something to fix.
```

- [ ] **Step 5: Verify both doc files render sensibly**

Run: `grep -n '^## \|^### ' docs/X4_AISCRIPT_NOTES.md` — expected: the new `### Multi-ware fill-cargo passes...` heading appears between the `### CONFIRMED: an infinite default-behaviour order...` heading and the `## \`clamp_trade_amount\`` heading, in that position.

Run: `grep -n '^## ' docs/superpowers/specs/2026-07-29-fillcargo-defer-commit-design.md` — expected: `## Known follow-up (flagged by final review, not fixed here)` is still immediately followed (after the new paragraph) by `## Versioning`, with no heading-level content accidentally inserted between them.

- [ ] **Step 6: Commit**

```bash
git add docs/X4_AISCRIPT_NOTES.md docs/superpowers/specs/2026-07-29-fillcargo-defer-commit-design.md
git commit -m "docs: document that multi-ware fill-cargo passes don't get the engine's automatic leg-pairing failure cancellation"
```
