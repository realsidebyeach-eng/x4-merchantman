# Test Cases

Manual in-game regression checklist, organized by the same pattern
classes as `docs/KNOWN_ISSUES.md` — consult before shipping any change
that touches trade-order creation, storage/capacity checks, or fill-cargo
commit logic. There is no automated test framework for this project (game-
engine XML scripting domain); `xmllint --noout` + schema validation is the
practical substitute for "run the tests," and these are the practical
substitute for "run the regression suite."

Each case: **Scenario** (setup), **Steps**, **Expected result**,
**Originally caught by** (how the real bug first surfaced, for context on
what to actually watch for).

---

## Class 1: Read-after-consumption

**Scenario:** A ship with Enable Logbook Entries on completes a delivery
that involves any `create_trade_order` call — fill-cargo batch delivery,
classic-mode delivery, or a stray-cargo external sale.

**Steps:**
1. Let a ship complete at least one full buy-then-deliver (or
   pickup-then-export) cycle in each of: classic mode, fill-cargo mode
   with exactly one ware decided, fill-cargo mode with 2+ wares decided.
2. Open Menu → Logbook and find the resulting entries.

**Expected result:** every entry shows a real ware name, real seller/buyer
name, and a real non-zero total price (where applicable) — never "null,"
a blank field, or a raw station hex ID.

**Originally caught by:** in-game Logbook entries reading "from null" with
missing price (instance 1a); a debug log line reading "selling ... to
null" (instance 1b).

---

## Class 2: Wrong-object-scope

**Scenario:** A home station with an active Build Storage construction
project (module being added or station under construction), Also Resupply
Build Storage on, assigned to a ship with a large money budget.

**Steps:**
1. Assign a ship to the station with both Enable Buying and Also Resupply
   Build Storage on.
2. Enable Debug Log and watch the trace for several passes.
3. Separately, test a ship assigned to a station in contested/pirate
   territory that the player still has dock rights at.

**Expected result:** Build Storage buy amounts are never clamped to 0
when the station's own construction-storage has real free capacity and
the ship can afford the wares — the ship actually buys and delivers
construction materials. The contested-sector ship still finds and takes
valid nearby deals; it isn't blocked purely by the sector's ambient
ownership.

**Originally caught by:** a ship (KHT-578) assigned to a station under
construction with a 100M credit budget that never bought or moved
anything (instance 2b); valid nearby deals in contested territory never
being taken (instance 2a).

---

## Class 3: Money/storage entanglement & wrong funding source

**Scenario:** A home station with a low or zero account balance, and a
ship with a healthy player-wallet budget.

**Steps:**
1. Drain (or start with) a home station's own account balance near zero.
2. Assign a trader ship with Enable Buying on and a normal player-wallet
   budget.
3. Watch whether the ship buys and delivers wares to that station
   normally.
4. Separately: confirm no ship anywhere has `this.ship.commander` set
   (check via the ship's Behaviour/Info panel — it should show no
   commander / not be in a Defence subordinate group as a side effect of
   this mod).

**Expected result:** a low or zero home-station account balance has **no
effect** on the ship's ability to deliver bought wares home (deliveries
are free, `price="0"`) — the ship is never stuck holding cargo waiting on
the station's own balance. Purchases are always funded from the player
wallet, confirmed by watching `player.money` decrease, never the home
station's own account.

**Originally caught by:** ships permanently stuck holding cargo (3a);
"adding credits to the station unstuck every ship at once" — a direct
user-observed repro (3c); traders appearing to do nothing until credits
were added to a station's own account (3b).

---

## Class 4: Off-by-one / boundary math

**Scenario:** A ship that already has queued trade orders (not starting
from an empty queue) when a fill-cargo pass begins; Min Free Cargo % to
Chase Another Stop (`mincargopercent`) set to its minimum (0) or default.

**Steps:**
1. Manually queue an odd number (1 or 3) of unrelated orders on a ship,
   then let it start a fill-cargo pass that would decide several more
   wares.
2. Enable Debug Log; check the `'commit done: ship now has N trade
   order(s) (expected M)'` line.
3. Separately, set Min Free Cargo % to Chase Another Stop to 0 and let a
   fill-cargo pass run to completion.

**Expected result:** the ship's total order count never exceeds 6,
regardless of whether it started with an even or odd number of queued
orders. With `mincargopercent=0`, the pass still respects the
pre-existing minimum free-cargo floor (never queues a real trade order
into a hold with 0 actual free space).

**Originally caught by:** final-review analysis (not live play) of the
order-count guard formula for odd starting counts (4b); final-review
analysis of the `mincargopercent=0` edge case (4c) — both caught before
shipping, which is why this needs to stay a deliberate manual check
rather than relying on it having "always worked so far" in play.

---

## Class 5: Wait-interrupt / order-queue-loses-control

**Scenario:** A ship with Fill Cargo Before Returning on, at a home
station/sector with at least 2-3 different wanted wares available from
different accessible sellers, and enough cargo space and budget to
plausibly fill the hold across multiple stops.

**Steps:**
1. Assign the ship, enable Debug Log, and let it run several full
   fill-cargo passes (both buying and selling, if both roles enabled).
2. Watch the debug log for the `'committing N'` / `'commit done: ship now
   has N trade order(s) (expected M)'` lines.
3. Watch the ship in-game (or via its order queue) across a full pass:
   does it visit multiple sellers/buyers before returning home once, or
   does it make a separate round trip per ware?

**Expected result:** a pass that decides 2+ wares visits every decided
partner and delivers/exports everything in **one** trip home — never one
round trip per ware. The `'commit done'` line's actual order count always
matches its own expected count.

**Originally caught by:** across one savegame's full log history, 763
buy-side + 524 sell-side fill-cargo passes with **zero exceptions** —
every pass that acted on anything capped at exactly one ware, and the
final "deliver everything" step frequently never ran at all (5a). This is
the single most important case in this file — the entire fill-cargo
feature was silently non-functional for its stated purpose (filling
cargo across multiple stops) until this was found and fixed, despite
passing all prior code review.

---

## Class 6: False-positive "handled" completion flags

**Scenario:** A ship carrying stray cargo (leftover from an interrupted
trade, or a role change) whose home station currently has 0 real buy
capacity for that ware (but still shows the offer as technically
available).

**Steps:**
1. Get a ship into a state with stray cargo and a home station that
   currently wants the ware but has 0 live buy-offer amount (e.g. its
   storage is already full for that ware).
2. Enable Debug Log and watch across several passes.

**Expected result:** the stray cargo does **not** get silently marked
"handled" by a 0-amount order — the debug log shows the actual amount
being moved (not the ship's full held quantity), and if that amount is 0,
the external-buyer fallback is tried instead of the cargo sitting stuck
indefinitely repeating an identical log line.

**Originally caught by:** cargo stuck "delivering" 446 units of Engine
Parts across dozens of passes with zero real progress, the debug log
reporting the full held quantity rather than the actual (zero) amount
used (6a).

---

## Class 7: Stale documentation after a late-stage fix

**Scenario:** Not an in-game test — a review-time check.

**Steps:**
1. After any code change to the storage-check sites (Class 2/3) or the
   fill-cargo commit-phase structure (Class 5), re-read the relevant
   design spec's "Fix" section and any in-line XML comments near the
   changed code.
2. Confirm both still accurately describe what the shipped code actually
   does — not what an earlier draft did.

**Expected result:** no contradiction between the spec/plan's prose, any
in-line code comments, and the actual shipped logic.

**Originally caught by:** three separate post-hoc doc-correction commits
in this project's history (`81fa52d`, `2e82e4b`, `86afb7a`, `41d207e`) —
all caught by a deliberate final-review pass, none by a user bug report.
This is a real, recurring failure mode for this project specifically
(prose docs drifting from code during iterative fixes), not a
hypothetical concern.

---

## Open/deferred items — watch for, don't assume fixed

- **Multi-ware storage sharing within one commit-phase batch:** if two
  wares decided in the same fill-cargo pass share a storage type at the
  buyer, watch for the second delivery coming up short and landing on
  the stray-cargo fallback (see `KNOWN_ISSUES.md`'s "Open/deferred"
  section). Not currently fixed — if you see this in play, it's a known
  gap, not a new regression.
- **Multi-ware leg-pairing failure protection:** if a leg of a multi-ware
  fill-cargo pass fails at runtime, its paired leg is not automatically
  cancelled (single-ware passes and classic mode are unaffected). No
  confirmed real-world instance yet — if you find one, it's evidence this
  gap should move from "accepted tradeoff" to "worth fixing."
