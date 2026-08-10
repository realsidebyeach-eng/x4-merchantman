# X4 AI Script Notes

Reference notes on X4: Foundations aiscript actions relevant to this mod's
trade-order logic. Combines **schema facts** (from the official Egosoft
`aiscripts.xsd`/`common.xsd` schema, cached locally — see
[Schema location](#schema-location)) with **empirically-verified runtime
behavior** discovered while building this mod, none of which is documented
anywhere else we could find (checked the Egosoft Community Wiki and general
web search — no comprehensive aiscript-actions behavioral reference exists
publicly as of 2026-08).

This is a living document. Add to it whenever a new action's behavior is
pinned down, especially anything that cost real debugging time.

## `create_trade_order`

**Schema** (`common.xsd:24057`):

> Creates an AI trade order and adds it to an object's order queue. A trade
> deal is created based on the given trade offer, so the trade can be
> performed in the future even if the trade offer changes or disappears in
> the meantime. The trade order ID is hard-coded. The trade deal is
> associated with the order (accessible via `$order.trade`) and is passed
> to the order as the "trade" parameter.

Attributes: `object` (required), `tradeoffer` (required), `amount`
(required), `price` (optional, defaults to the offer's price), `bundle`,
`unbundle`, `immediate`, `internal`, `salvage`, `name`.

**No `result` attribute exists.** Unlike `clamp_trade_amount` (below) or
`add_build_to_modify_ship` (seen in third-party mod source), there is no
way to read back whether the order was actually created/queued
successfully in the same pass. Don't design logic that assumes you can
check the outcome of a `create_trade_order` call.

### Empirical finding: reference invalidation ("read-after-consumption")

Reading any property off the variable passed as `name=`/`tradeoffer=`
**after** the `create_trade_order` call — `.buyer`, `.seller`, `.ware` —
returns null. Not documented in the schema; discovered by hitting this bug
repeatedly across unrelated fixes in this mod (null-name Logbook entries,
stray-cargo buyer names, fill-cargo delivery ware names — see project
memory `feedback_read_after_consumption_pattern`). **Always capture any
needed property into a separate variable before the call that consumes
it.**

### CONFIRMED: an infinite default-behaviour order is aborted at its next `<wait>` once anything is queued behind it

This was originally an empirical, "well-evidenced but not provable"
finding — now **confirmed directly from Egosoft's own official source**,
extracted from the base game's packed catalogs with `x4cat` (see
[x4cat extraction](#extracting-official-egosoft-scripts) below).
`aiscripts/order.trade.routine.xml:859-861`, immediately after two
back-to-back `create_trade_order` calls in the same script pass:

```xml
<!-- short wait to prevent this script from doing more actions before
     it's aborted if trade orders are created.
     if this is an infinite order with orders after it in the order queue,
     the order will be aborted at this point. -->
<wait exact="1ms"/>
```

This is Egosoft's own developer documenting the exact mechanism this mod
spent 2026-07-29 reverse-engineering — and it generalizes further than we
originally assumed:

- **Not specific to `create_trade_order`.** It's about any infinite
  default-behaviour order (like `SBEStationTrader`, `infinite="true"`)
  getting aborted at its own next `<wait>` once *anything* is queued
  behind it in the ship's order list.
- **Not specific to `immediate="false"`.** The official script above uses
  `immediate="true"` for both calls and still gets aborted at the wait
  right after. `immediate` only controls whether the *new* order preempts
  the ship's *current* order — it has no bearing on whether the *calling*
  infinite order survives past its own next wait once something exists
  behind it in the queue.

Egosoft's own fix is the same shape as ours: do all order creation before
the wait, and don't rely on anything after that wait running (their own
code only does a harmless debug-dump loop between the `create_trade_order`
calls and the `<wait>`). This mod's v19 fix (decision phase / commit phase
split, described in project memory `project_create_trade_order_wait_interrupt`)
follows the same principle independently derived from empirical testing —
now validated against the official implementation.

**Aside on `immediate="true"` interaction between two orders:** the same
script's line 834 comment — *"Queue the sell order first as immediate. It
will be displaced by the buy order also being queued as immediate"* —
shows `immediate="true"` has different queuing semantics than this mod's
`immediate="false"`: two `immediate="true"` calls in a row cause the
*second* to displace the first as current, rather than both appending to
the queue in creation order. This mod's two-pass commit-phase design
(first-leg loop, then second-leg loop) specifically relies on
`immediate="false"`'s plain append-in-creation-order behavior — don't mix
in `immediate="true"` without re-deriving the ordering guarantees.

**No `result` attribute exists on `create_trade_order`**, confirmed by the
full schema attribute list (`common.xsd:24057`) — unlike
`clamp_trade_amount` (below) or `add_build_to_modify_ship` (seen in
third-party mod source), there is no way to read back whether the order
was actually created/queued successfully in the same pass. Don't design
logic that assumes you can check the outcome of a `create_trade_order`
call.

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

## `clamp_trade_amount`

**Schema** (`common.xsd:21988`):

> Clamp trade amount value based on buyer/seller storage situation
> (bundle/unbundle states are taken into account and the appropriate
> storages - cargo, ammo or units - are checked)

The `result` attribute (`common.xsd:22044`) is documented as:

> Amount clamped so it matches how much seller has and how much buyer can
> store **and afford**.

And `reason` (`common.xsd:22051`) can be `'seller_cargo'`, `'buyer_cargo'`,
or **`'buyer_money'`**.

### Confirmed pitfall: folds in buyer affordability even for `price="0"` trades

This is the schema-level confirmation of a real bug this mod shipped in
v16: this mod's home-delivery legs execute at `price="0"` (free internal
transfer), but `clamp_trade_amount` still factors in whether the buyer can
*afford* the trade at its real configured price — so ships sat stationary
delivering to a home station with a low account balance, even though the
delivery itself would cost nothing. The schema text confirms this isn't a
misunderstanding of the action, it's how it's documented to work. Fixed by
replacing `clamp_trade_amount` with a plain `.cargo.{ware}.free` storage
check for any leg that executes at `price="0"`. See project memory
`feedback_storage_check_pitfalls`.

**How to apply:** never use `clamp_trade_amount` for a leg that executes
at `price="0"` — check `reason` for `'buyer_money'` if you must use it on
a mixed-price code path, or just use a direct storage check
(`.cargo.{ware}.free`) instead when price is always zero.

## General bug-pattern classes (not schema-specific)

These recur across this codebase regardless of which action triggers
them — see project memory for full detail and instance history:

- **Read-after-consumption** (`feedback_read_after_consumption_pattern`) —
  any property read off a variable passed to `create_trade_order` after
  the call returns null. Grep for `\.ware\b|\.buyer\b|\.seller\b` on the
  lines following any new `create_trade_order` call.
- **Wrong-object-scope** (`feedback_storage_check_pitfalls`) — a
  storage/capacity check must target the actual buyer/seller for the
  current pass (`$querybuyer`), not a hardcoded assumption like
  `$homestation` — this mod services two different buyer categories
  (production wares vs. Build Storage) through the same shared loop.

## Extracting official Egosoft scripts

The base game and DLCs ship all scripts packed into `.cat`/`.dat` catalog
archives, not as loose XML. Egosoft's own extraction tool is Windows-only.
On Linux, [`x4cat`](https://github.com/meethune/x4cat) (MIT, pure Python,
zero runtime dependencies) works directly without installing anything:

```bash
git clone https://github.com/meethune/x4cat.git && cd x4cat
GAME="/path/to/X4 Foundations"

# List matching files (extensions/DLCs need -p ext_ for their ext_NN.cat naming)
python3 -m x4_catalog list "$GAME" -g 'aiscripts/*.xml'

# Extract matching files
python3 -m x4_catalog extract "$GAME" -o ./out -g 'aiscripts/order.trade.*.xml'
```

**Note:** the DLC extensions (`ego_dlc_terran`, etc.) have **no
`aiscripts/` directory at all** — all core trade-order logic lives in the
**base game's** catalogs (180 aiscripts total). The trade-relevant ones:
`order.trade.*.xml`, `trade.station.xml`, `trade.find.*.xml`,
`orders.base.tradecomputer.xml`, `lib.units.trade.xml`. This is the
highest-authority source available for any uncertain action behavior —
check it before relying on third-party mod source or schema text alone.

## Schema location

The official Egosoft schema (`aiscripts.xsd`, `common.xsd`,
`parameters.xsd`) is cached at
`/home/sidebyeach/Projects/X4SatelliteScout-worktree-implementation/.schema-cache/`
(a sibling project, not part of this repo). Action definitions
(`<xs:element name="...">`) live in `common.xsd` — `aiscripts.xsd` mostly
just includes it. To look up any action:

```bash
grep -n '<xs:element name="ACTION_NAME"' .schema-cache/common.xsd
```

Then read forward from that line number for the full `<xs:complexType>`
block (attributes, types, and any `<xs:documentation>` annotations). This
schema documents attribute *types and existence* authoritatively — it does
**not** document runtime behavior/ordering/interrupt semantics beyond what's
written in free-text `<xs:documentation>` blocks, which is why the
empirical findings above still had to be discovered by hand.
