# Valuation & Performance Calculations — Architecture Review and Target Design

**Status:** Proposal
**Last updated:** 2026-07-25
**Scope:** Holdings snapshot pipeline, daily valuation history, FX resolution, performance
(TWR/MWR/XIRR) computation, and their SQLite storage footprint.
**Goals:** Reduce recalculation time by an order of magnitude on realistic portfolios, make
historical valuation reproducible, and reduce database size by 60–90% — without weakening
calculation semantics (Decimal precision for accounting values, flow provenance, valuation
quality statuses).

---

## Revision history

**2026-07-25 — Revision 2.** Substantial revision after a closer read of the FX layer, the
returns math, and the lot schema, plus a measured storage model.

- **New §2.2 (FX correctness) is now the highest-priority finding.** Historical FX resolution
  is unbounded, bidirectional, and hop-count-optimised, which makes stored history mutable
  under backfill and multi-hop conversion non-reproducible. This is a correctness issue that
  outranks every performance item below.
- **New §2.4:** TWR telescopes across flow-free days, so the dense daily valuation table is
  not required for returns correctness — only for charting.
- **New §2.5:** `lots` + `lot_disposals` already support as-of temporal reconstruction, which
  removes the need to replay from inception for backdated edits.
- **New §3:** measured per-row sizes and growth laws, calibrated against a reported real
  420 MB database.
- **New §7:** assessment of PR #1218 against this design.
- **Corrections to Revision 1.** Two claims in the initial draft were wrong and are corrected
  in place:
  - *Old S3 over-weighted `daily_account_valuation`.* Measurement puts it at 5–31 MB across
    realistic profiles — wasteful, not pathological. `quotes` is the table that becomes
    dominant once snapshots are fixed, and the initial draft dismissed it as "secondary".
  - *Old P3 claimed there is no FX cache.* There is one (`FxService.converter`,
    `fx_service.rs:18`), rebuilt after every market sync. The real problem is that the call
    pattern defeats it (§2.2, F7).
- **Workstream order changed:** FX determinism becomes **WS-0**, ahead of all storage work.

**2026-07-23 — Revision 1.** Initial review and four-workstream target design.

---

## 1. Current Architecture (as-built)

### 1.1 Pipeline

```
activities ──► holdings snapshots (sparse keyframes) ──► daily_account_valuation (dense, 1 row/account/day)
                        │                                          │
                        ├─► lots / lot_disposals (dual-write)      ├─► performance (TWR/MWR/XIRR) — computed on demand
                        └─► snapshot_positions (dual-write)        └─► scoped aggregation — computed on read, memory-cached
                        ▲                                          ▲
                        └──────────── FX service / CurrencyConverter ─────────────┘
```

1. **Holdings snapshots** (`crates/core/src/portfolio/snapshot/snapshot_service.rs`)
   `recalculate_holdings_snapshots` replays activities day-by-day per account
   (`calculate_daily_holdings_snapshots`, line 946). A keyframe is persisted only for days
   with activity (plus the first day), so `holdings_snapshots` is sparse. Each keyframe
   serialises the **entire** account state — every `Position` including its full
   `lots: VecDeque<Lot>` — into the `positions` TEXT column as JSON.

2. **Daily valuation** (`crates/core/src/portfolio/valuation/valuation_service.rs:1661`)
   `calculate_valuation_history` calls `get_daily_holdings_snapshots`
   (`snapshot_service.rs:1209`), which **materialises one full snapshot clone per calendar
   day** by carrying keyframes forward, then values each day with
   `calculate_valuation_with_price_factors` (`valuation_calculator.rs:51`) and writes one
   `daily_account_valuation` row per day.

3. **FX** (`crates/core/src/fx/`) A `CurrencyConverter` (graph of pairs, `BTreeMap<date, rate>`
   per pair) is cached in `FxService.converter` and rebuilt on startup and after each market
   sync. Valuation prefetches a dense per-day rate map via `fetch_fx_rates_for_range`
   (`valuation_service.rs:342`).

4. **Performance** (`crates/core/src/portfolio/performance/performance_service.rs`)
   Not persisted. TWR (`compute_time_weighted_returns`, line 550), MWR/XIRR and attribution
   are computed per request from the daily valuation series.

5. **Scoped aggregation** (`valuation_service.rs:1999–2173`) "ALL"/portfolio views load all
   member accounts' daily rows, re-derive external flows from activities, and merge at read
   time, with an in-memory cache keyed by `max(calculated_at)`.

### 1.2 Orchestration

Activity/asset changes flow through the domain-event queue
(`apps/{tauri,server}/src/domain_events/queue_worker.rs`). The planner derives
`since_date` = earliest changed activity date; the job runs `SnapshotRecalcMode::SinceDate(d)`
then `ValuationRecalcMode::SinceDate(d)` (`queue_worker.rs:317–324`). Startup and market sync
use `IncrementalFromLast`. Snapshot recalculation processes all accounts inside one sequential
day loop; valuation runs per account via `join_all` (`apps/tauri/src/listeners.rs:330–342`).

### 1.3 Storage shapes (`crates/storage-sqlite/src/schema.rs`)

| Table | Granularity | Notable columns |
|---|---|---|
| `holdings_snapshots` | account × activity-day | `positions` TEXT (JSON incl. **all lots**), `cash_balances` TEXT |
| `snapshot_positions` | keyframe × position | relational dual-write, currently write-only |
| `lots` / `lot_disposals` | current lot book / disposal events | normalized, indexed |
| `daily_account_valuation` | account × **calendar day** | 23 columns, decimals as TEXT |
| `quotes` | asset × day × source | decimals as TEXT; redundant `day` + `timestamp` |

All writes use `replace_into`; range rewrites delete then bulk-insert in one transaction,
chunked at 1000.

---

## 2. Review — Findings

Ordered by severity: correctness first, then performance, then storage.

### 2.1 FX resolution — correctness (highest priority)

**F1 — Nearest-neighbour lookup is bidirectional and unbounded; history mutates retroactively.**
`get_direct_rate` (`currency_converter.rs:79-115`) takes the closest rate on-or-before the date
and the closest on-or-after, and returns whichever is nearer in days. A **future** rate wins
whenever it is strictly closer, and there is no staleness ceiling — `test_single_static_rate_works_anywhere`
(line 260) asserts a 2023 rate is returned for both 2000 and 2050.

Consequence: **syncing a new FX rate can change the rate resolved for a past date.** But
`IncrementalFromLast` and `SinceDate` only recompute forward, so those historical rows are
never revisited. Stored history silently diverges from what a full rebuild would produce.

This is the deepest issue in the review: *a materialized view whose inputs mutate retroactively
cannot be maintained incrementally without versioning those inputs.* Every incremental mode in
the system currently assumes the past is immutable, and for FX it is not.

**F2 — Path search optimises hop count, never rate quality.**
`convert_amount` (`currency_converter.rs:119-159`) BFS-finds the fewest-hops path. A single
stale direct rate (say USD→CHF from 2015) beats a two-hop path with daily rates. Hop count and
date proximity are different quality metrics and only the first is optimised.

**F3 — Multi-hop path selection is not reproducible.**
`for neighbor in neighbors` iterates a `HashSet<String>` (`adj`, line 13). Rust's default hasher
is randomly seeded, so iteration order varies between process runs and between converter
rebuilds. With `visited` marked at enqueue, two equal-length paths (GBP→USD→CHF vs GBP→EUR→CHF)
are resolved by hash order. Since stored cross-rates do not triangulate exactly, the same recalc
can yield different monetary values on different runs.
*Status: mechanism verified by reading; not yet reproduced. A ~20-line test that builds a
converter with two equal-length paths and asserts stability across rebuilds will confirm or
rule this out, and should be written before acting on it.*

**F4 — Silent fallback to today's rate for historical dates.**
`get_rate_for_date_between_normalized` (`fx_service.rs:165-176`): if the converter cannot
resolve, it loads the **latest** rate and applies it to the requested historical date with a
`warn!` only. In practice this rarely fires precisely because F1 almost always returns
something.

**F5 — FX has no provenance, unlike everything else.**
`ValuationStatus` and `ExternalFlowSource` are rigorous: the system refuses to fabricate a
number when quote coverage or flow boundaries are unknown. FX, which multiplies into every
base-currency figure, has no equivalent. There is no way to distinguish an exact same-day rate
from one forward-filled 400 days, borrowed from the future, or substituted from today.

**F6 — Three independent inverse implementations, all lossy.**
`Decimal::ONE / rate` at `currency_converter.rs:60`, `fx_service.rs:89`, and
`valuation_calculator.rs:395`. Decimal division rounds at 28 digits, so A→B→A is not identity
and direct-vs-inverse paths disagree in the last places.

**F7 — The prefetch defeats the cache: days × pairs full graph traversals, per account.**
`fetch_fx_rates_for_range` (`valuation_service.rs:342-379`) loops every date × every pair and
calls `get_exchange_rate_for_date`, each of which runs a complete BFS allocating a `VecDeque`,
a `HashSet<String>`, and two `String`s per edge check inside `get_direct_rate`. For 3,650 days
× 4 pairs that is ~15k BFS traversals **per account**, none shared between accounts. The
converter's `BTreeMap` provides O(log n) date lookup, and the caller then discards that by
materialising a dense per-day map keyed by `(String, String)`.

**F8 — Converter rebuild loads all history.** `initialize_converter` (`fx_service.rs:41`) calls
`get_historical_exchange_rates()` unfiltered and stores both directions — a full reload and
graph rebuild on every market sync, before the recalc starts.

### 2.2 Compute hot spots

**P1 — Dense per-day snapshot materialisation with full-state clones.**
Both the write path (carry-forward `previous_holdings_snapshot.clone()`,
`snapshot_service.rs:1024`) and the valuation path (`current_state.clone()`, line 1313)
deep-copy the whole account state — every `Position`, every `Lot` — for **every calendar day**,
though holdings change only on activity days. This is the largest CPU and allocator cost in the
pipeline, and the source of the reported 1.5–2 GB peak RSS.

**P2 — Per-day per-lot recomputation of interval-constant values.**
`calculate_cost_basis_in_currency` loops every lot every day (`valuation_calculator.rs:218`) to
redo the same acquisition-FX conversion. Cost basis and quantities are constant between
keyframes; only prices and FX vary. Correct complexity is
`O(days × held_assets + keyframes × lots)`; the current one is `O(days × lots)`.

**P3 — The quote path materialises a dense grid of String-heavy structs, per account.**
`get_quotes_in_range_filled` (`quotes/service.rs:1130`) issues **one query per symbol**, then
`fill_missing_quotes` (line 2429) clones a full `Quote` struct — with ~6 owned `String`s — for
**every (asset, day) pair**. `calculate_valuation_history` calls this per account, so N accounts
holding the same ETFs each rebuild the same grid.

**P4 — No real parallelism.** Snapshot recalculation is one sequential loop over accounts and
days. Valuation uses `join_all` over synchronous CPU + blocking repo calls, so it largely
serialises; nothing uses `spawn_blocking`/rayon.

**P5 — Read path re-aggregates and re-parses per request.** Every scoped history request loads
all member accounts' daily rows (23 TEXT columns, parsed via `parse_decimal_lossy`), re-queries
activities to rebuild flows, and merges in memory. The cache key requires a `MAX(calculated_at)`
scan per request, and any recalc of any member invalidates all scopes.

**P6 — Full-resolution, full-width payloads to the frontend.** `useValuationHistory`
(`apps/frontend/src/hooks/use-valuation-history.ts`) fetches every daily row with all fields
for the range, with no interval parameter and no downsampling.

### 2.3 TWR telescopes — the daily table is not needed for returns

`compute_time_weighted_returns` (`performance_service.rs:629-663`) computes
`twr = (curr + outflow − prev − inflow) / (prev + inflow)` and chains
`cumulative *= (1 + twr)`. On a day with no external flow this reduces to `curr/prev`, so a run
of flow-free days telescopes **exactly**:

```
Π (V_d / V_{d-1})  =  V_end / V_start
```

Exact TWR therefore requires valuation points only at **external-flow days** (and the day
before each), **status transitions**, and window endpoints — a few hundred points per decade,
not 3,650 per account. The dense daily table exists **for the chart**, not for correctness.

Caveat: the `excluded_from_compounding` and `value_status` gating deliberately break the chain,
so boundaries must include those days too — but those are precisely the days worth keeping.

### 2.4 The lot book is temporal data — query it as-of, don't replay it

`lots` carries `open_date`, `original_quantity`, `original_cost_basis`; `lot_disposals` carries
`lot_id` (FK), `disposal_date`, `quantity`
(`migrations/2026-05-26-000001_lot_disposals/up.sql`). So:

```
quantity_of_lot_L_as_of(D) = L.original_quantity − Σ disposals(lot_id = L, disposal_date ≤ D)
       over lots where L.open_date ≤ D
```

That is an indexed temporal query, not a replay. It correctly answers the "buy 100 / sell 40 /
replay from mid-January" case (100, not today's 60) and removes the need to rebuild from
inception for backdated edits.

Caveat: `split_ratio` is persisted as the *current* cumulative ratio, so an as-of ratio needs
splits after `D` divided out — derivable from `SPLIT` activities, but it must be explicit.

### 2.5 What is already good (keep)

- Sparse keyframe model for holdings (only activity days persisted).
- `SinceDate` incremental planning and the split-restart guard (`snapshot_service.rs:853-872`).
- Deterministic snapshot IDs (`stable_id`) enabling idempotent upserts.
- Normalized `lots`/`lot_disposals`/`snapshot_positions` tables.
- **The flow-provenance and valuation-quality model** (`ExternalFlowSource`, `ValuationStatus`).
  This is unusually rigorous — most trackers silently emit a plausible wrong number where this
  system refuses. Every proposal below preserves it; F5 proposes extending the same discipline
  to FX.
- Chunked transactional bulk writes.

---

## 3. Storage — measured sizing model

Sizes below are modelled from the actual serde shapes and schema, calibrated against one
reported real database (420 MB total, ~343 MB `holdings_snapshots`, PR #1218). The model
predicts 379 MB of snapshots for that profile — within ~10%. **These are estimates, not
measurements**; §5 Phase 0 replaces them with `dbstat` numbers.

### 3.1 Per-row sizes

| Row | Bytes |
|---|---|
| One serialised `Lot` (JSON, camelCase, 19 fields) | **572** |
| One `Position` without lots (incl. PR #1218's cost-basis scalars) | 432 |
| One `daily_account_valuation` row (23 cols) | 579 + 172 index = **751** |
| One `quotes` row (14 cols) | 259 + 160 index = **419** |

### 3.2 Growth laws — only one table is pathological

```
holdings_snapshots  =  K·P·432  +  572 · Λ·(K+1)/2     ← QUADRATIC in activity-days
daily_valuation     =  accounts · days · 751           ← linear
quotes              =  assets · trading_days · 419     ← linear
lots                =  Λ · 650                         ← linear in lots only
```
(K = activity-days/keyframes, P = positions, Λ = lots created)

Every keyframe re-serialises **all lots accumulated so far**, so total lot serialisations
≈ Λ·K/2. That single term is the entire bloat story.

### 3.3 Modelled profiles (today)

| Profile | `holdings_snapshots` | `daily_valuation` | `quotes` | `lots` |
|---|---|---|---|---|
| PR-author-like (5y, 900 activity-days, 28 assets, 4 acct) | **379 MB** | 5 MB | 17 MB | 1.5 MB |
| Modest DCA (10y, 240 activity-days, 12 assets) | **80 MB** | 5 MB | 18 MB | 1.2 MB |
| Active trader (10y, 2,200 activity-days, 80 assets) | **3.7 GB** | 13 MB | 86 MB | 6 MB |
| Long-horizon (20y, 1,000 activity-days, 40 assets) | **699 MB** | 31 MB | 92 MB | 2.5 MB |

Three conclusions:

1. **`holdings_snapshots` is the only pathological table.** The active-trader case reaches
   multiple GB while every other table stays double-digit MB.
2. **`daily_account_valuation` is wasteful, not pathological** (5–31 MB). Revision 1
   over-weighted it.
3. **`quotes` becomes the largest table once snapshots are fixed** — 3–6× the valuation table
   in every profile — and it is source data, so it can only be encoded better or pruned, never
   dropped.
4. **`lots` is 1–6 MB** — holding the same information the snapshots duplicated hundreds of
   times. That asymmetry (1.5 MB canonical vs 379 MB duplicated) is the clearest argument for
   making it canonical.

### 3.4 Target storage

| Table | Post-#1218 | Target | How |
|---|---|---|---|
| snapshots | aggregate JSON | **−42%** | drop the `positions` JSON column; read `snapshot_positions` relationally; stop repeating `inception_date`/`created_at`/`last_updated` per keyframe |
| daily valuation | dense daily | **−92%** | keep only flow-boundary / status-transition / endpoint rows (§2.3 makes this lossless for returns); chart resolution computed on demand |
| quotes | TEXT decimals, 4 indexes | **−64%** | numerics as scaled INTEGER or REAL; drop the duplicate `timestamp` column (`day` already exists); drop one redundant index |
| lots / disposals | unchanged | unchanged | canonical, plus as-of queries (§2.4) |

| Profile | Today | Post-#1218 | Target |
|---|---|---|---|
| PR-author-like | ~404 MB | 31 MB | **13 MB** |
| Modest DCA | 105 MB | 25 MB | **9 MB** |
| Active trader | 3.8 GB | 140 MB | **59 MB** |
| Long-horizon | 827 MB | 137 MB | **45 MB** |

In the target state ~75% of the database is source data (quotes + activities + lots), which is
the correct end state; today derived data is 90%+.

### 3.5 Three redundant indexes — free win, available now

Verified against the final schema state (no up-migration drops them):

- `idx_dav_account_id` — redundant prefix of `idx_daily_account_valuation_account_date`
- `idx_holdings_snapshots_account_id` — redundant prefix of `idx_holdings_snapshots_account_date`
- `idx_quotes_asset_day` — redundant prefix of `uq_quotes_asset_day_source`

Each costs ~45–55 B/row of write amplification and disk for no query benefit. A three-line
migration recovers roughly 10–15% of those tables.

---

## 4. Target Design

Five workstreams. **WS-0 is a prerequisite for trusting anything else**: a faster pipeline that
produces irreproducible numbers is worse than a slow one.

### 4.1 WS-0: Deterministic, bounded, provenanced FX (correctness)

1. **Forward-fill only for market-sourced pairs**, with an explicit max-staleness window. Never
   look ahead. This alone makes historical valuation *immutable under backfill*, which is what
   restores soundness to every incremental mode (F1).
2. **Keep bidirectional/unbounded lookup only for pairs explicitly marked static or `MANUAL`.**
   That is the legitimate case `test_single_static_rate_works_anywhere` protects; it should be a
   declared property of the pair, not an emergent property of the algorithm.
3. **Deterministic, quality-aware path selection.** Resolve each `(from, to)` to a path once per
   run — prefer direct, then by data density/recency — and iterate a `BTreeSet` rather than a
   `HashSet` so ties break identically every run (F2, F3).
4. **Add `fx_status` to the valuation row**, mirroring `ValuationStatus`:
   `Exact | ForwardFilled(days) | Stale(days) | LatestFallback`. Cheap to compute, and it makes
   F1/F4 visible instead of silent (F5).
5. **Batch API:** one `rates_for(pairs, start..end)` returning columnar arrays, resolved once
   per recalc run and shared across accounts — replacing ~15k BFS traversals per account (F7).
6. Single shared inverse helper (F6); date-bounded converter rebuild (F8).

### 4.2 WS-A: Normalized tables canonical; lots out of keyframes (storage)

Keyframes stop serialising `lots`; position aggregates live in `snapshot_positions`; lot detail
comes from `lots`/`lot_disposals`. **This is what PR #1218 implements** — see §7.

Follow-on beyond that PR: drop the `positions` JSON column entirely and read
`snapshot_positions` relationally (§3.4), and add the as-of lot query from §2.4 to remove the
rebuild-from-inception cliff.

### 4.3 WS-B: Interval-based compute + a sane quote path (CPU/memory)

Replace `get_daily_holdings_snapshots` + per-day valuation with an interval evaluator:

```
for each keyframe interval [k_i, k_{i+1}):          # holdings constant here
    per-interval (once):  cost_basis, book_basis, net_contribution, cash per currency,
                          held = [(asset_id, qty, ccy, multiplier)]
    per day d:            investment(d) = Σ qty × close(asset,d) × fx(ccy→acct,d)   # O(held)
                          cash(d)       = Σ cash(ccy) × fx(ccy→acct,d)
                          emit row (existing quote-gating rules unchanged)
```

Alongside it, fix the quote path (P3): batch the per-symbol queries into one, replace the dense
`Quote` grid with an `(asset, date) → close` map of numbers, and share it across accounts in a
run. Add account-level parallelism via rayon/`spawn_blocking`.

This is the change most likely to resolve the reported RAM and timeout symptoms (issue #1347),
and it needs no schema change.

### 4.4 WS-C: Read path — materialized TOTAL series + interval API

1. Upsert a materialized aggregate series (`account_id = 'TOTAL'`, and per saved portfolio
   scope) during recalculation, so dashboard reads become a single indexed range scan.
2. Add `interval: daily | weekly | monthly` to the history queries and downsample in SQL.
   **This is a compute optimisation, not just a payload one:** a monthly ALL-time chart needs
   only month-end quotes — 1/30th of the quote I/O — which makes the most expensive view the
   cheapest.
3. A slim chart DTO — `(date, total_value_base, net_contribution_base, currency)` — for
   dashboard/net-worth charts; the full row stays for the performance page.

### 4.5 WS-D: Storage encoding & retention

1. **Sparse valuation keyframes** at flow/status boundaries (§2.3) — the −92% row reduction,
   lossless for returns.
2. **Narrow the quotes row**: scaled-integer or REAL numerics, drop the duplicate `timestamp`,
   drop the redundant index (§3.5). Optional: prune quotes for dates before an asset was ever
   held.
3. **Bounded decimal precision at rest** for derived valuation fields (8 fractional digits);
   canonical inputs (activities, lots, quotes) keep full precision.
4. **Housekeeping**: `PRAGMA auto_vacuum = INCREMENTAL`, or a scheduled `VACUUM` after
   migrations and compaction.

---

## 5. Migration & Verification Plan

**Phase 0 — Baseline harness (prerequisite).** Seeded benchmark DB generator (accounts × years
× cadence knobs); timing harness around snapshot + valuation + scoped read; per-table size via
`dbstat`. **Parity harness:** golden dump of all valuation rows + performance summaries,
reproduced byte-for-byte (modulo `calculated_at`) by every subsequent phase. Add the FX
determinism test from F3 here.

**Phase 1 — WS-0 (FX).** Correctness first. Expect *intended* value changes; the parity harness
baseline is re-cut once, deliberately, with a documented diff.

**Phase 2 — WS-A.** PR #1218 plus the follow-ons in §4.2.

**Phase 3 — WS-B.** No schema change; ship with a debug assertion cross-checking the interval
evaluator against the legacy path.

**Phase 4 — WS-C**, then **Phase 5 — WS-D**. Retention ships opt-in with a "compact history"
action.

The three redundant indexes (§3.5) can land at any time, independently.

---

## 6. Risks & Open Questions

1. **WS-0 changes numbers.** Bounding staleness and forbidding future rates will move
   historical values for multi-currency portfolios. This is a correction, but it is visible to
   users and needs release-note treatment and a re-cut parity baseline.
2. **Static/manual pairs must keep working.** Forward-fill-only breaks pairs whose only rate is
   in the future (newly registered, backfill pending). The static-pair carve-out in §4.1.2 is
   load-bearing, not optional.
3. **As-of lot reconstruction (WS-A follow-on):** audit every reader of
   `AccountStateSnapshot.positions[*].lots` (holdings service, agent tools, addons via
   type-bridge) before dropping the JSON column; keep a compatibility shim if any addon-facing
   API exposes historical lots.
4. **Retention vs device-sync:** compaction deletes rows; confirm derived tables are outside
   the sync scope before enabling WS-D.
5. **HOLDINGS-mode (manual snapshot) accounts:** their keyframes are *source data*. Exclude
   from any pruning.
6. **TOTAL materialisation staleness:** the aggregate must be rewritten in the same job as any
   member account; the event queue serialises portfolio jobs, but multi-writer server mode
   needs a check.
7. **Precision rounding (WS-D.3):** confirm no consumer does exact-equality reconciliation
   against persisted valuation strings. Note the read path already does
   `Decimal::from_str(..).unwrap_or(ZERO)` (`valuation_service.rs:40`), which silently zeroes on
   parse failure — worth fixing regardless.
8. **Quote numeric encoding (WS-D.2)** is a precision judgment call. Prices are observations
   with 4–6 meaningful digits, not accounting values, but the change should be a deliberate
   decision rather than a side effect.

---

## 7. Assessment of PR #1218

[`perf(db): cut holdings_snapshots bloat ~8×, tune SQLite, fix TWR headline`](https://github.com/wealthfolio/wealthfolio/pull/1218)

**Direction: correct.** The PR is WS-A, and the maintainer review converged on exactly the model
this document recommends: aggregate-only snapshots, `lots` as the current lot book, append-only
seeding gated by a high-water mark, backdated edits rebuilt from inception, and a
drop-and-rebuild migration for derived read models rather than an in-migration FX
reconstruction. All review points were addressed in the revision, verified in the diff.

**Issues to resolve before merge:**

1. **The disk-space win may not materialise.** The revised migration is plain SQL `DELETE`s
   with no `VACUUM` and no `auto_vacuum` anywhere on the branch — deleted pages go to the
   freelist and the file stays large. The headline "420 MB → 51 MB" came from the earlier
   strip-migration path that ended in `VACUUM`. Fix: run `VACUUM` conditionally in
   `run_migrations` when the reset migration was applied (it cannot go inside the migration,
   which Diesel wraps in a transaction), and re-verify the measurement.
2. **`hydrate_seed_lots_from_table` fails open.** On a lots-query error it warns and proceeds
   with an empty-lot seed, silently degrading cost basis and risking stale lot rows. The
   append-only gate itself fails closed to `Full`; hydration should do the same.
3. **Stale title/description and dead code.** The PRAGMA tuning and strip-migration described
   in the body no longer exist; `backup_database_from_path`, added for that removed path, now
   has no callers. Either delete it or wire an actual pre-migration backup.

**Accepted trade-off worth noting:** after the first activity of a day creates a keyframe, every
subsequent same-day edit has `since == high-water mark` and triggers a full rebuild from
inception — the most common editing flow. The maintainer explicitly accepted this. The as-of
lot query in §2.4 removes it without the checkpoint table that was floated as the alternative,
and WS-B makes full rebuilds cheap enough that the cliff stops mattering either way.
