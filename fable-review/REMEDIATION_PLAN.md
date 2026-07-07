# Review remediation plan — fixing the major findings of the 2026-07-06 comprehensive review

Status: 📋 open, not started. Drafted 2026-07-06 by the reviewing agent (Claude / Fable 5),
immediately after completing the review.

**Source review artifacts** (all links below point into these):
- [`REVIEW_SUMMARY.md`](REVIEW_SUMMARY.md) — ranked findings + design verdicts
- [`REVIEW_NOTES.md`](REVIEW_NOTES.md) — the full per-module log; every finding has a unique **F-ID**
- [`ARCHITECTURE_REVIEW.md`](ARCHITECTURE_REVIEW.md) — design review (§1–§8 + keep/evolve/rework table)
- [`REVIEW_PLAN.md`](REVIEW_PLAN.md) — what was reviewed at what depth

**How to read the links.** Each issue cites its F-ID plus a `file:line` pointer at the finding's
first line — e.g. `REVIEW_NOTES.md:16` opens directly in an editor/terminal; the F-ID is unique,
so `grep -n "F-T1-1" docs/meta/review/run-2026-07-06/REVIEW_NOTES.md` always finds the full
explanation (failure scenario, confidence label, code refs) even if line numbers drift. Do not
renumber or edit the review notes; they are the frozen evidence base.

**Scope.** All critical/high findings, all mediums, and the lows that are either one-line fixes
or belong to a workstream already being touched. Deliberately-deferred nits are listed at the
end so nothing silently disappears. A completeness table at the bottom maps every in-scope F-ID
to its workstream.

---

## Sequencing at a glance

| # | Workstream | Urgency driver | Size | Blocks / gated by |
|---|-----------|----------------|------|-------------------|
| WS1 | Lease mutual exclusion | **Critical bug, live today** | S (½ day) | Nothing — do first |
| WS2 | Cutover exit-correctness gates | Must land before Thread A `route:true` | S–M (1 day) | Gates the cutover; not needed for Tue's entry-only/route:false load test |
| WS3 | Data lifecycle (3 unbounded stores + checker) | Bloat grows daily; Thread C's missing chapter | M (1–2 days) | Independent |
| WS4 | Test isolation + coverage floor | The suite mutates live state **today** | S–M (1 day) | Do before habitual test runs |
| WS5 | Drain & plumbing resilience | Before dual-run scale-up | S–M (1 day) | After WS1 |
| WS6 | Trading-layer hardening | Pre-real-money class (with Thread H) | M (2 days) | Independent |
| WS7 | Ledger integrity | Pre-real-money class (with Thread H) | S–M (1 day) | Coordinate with Thread H fix |
| WS8 | Language & validation honesty | Correctness-of-claims; one design decision needed | M (1–2 days + decision) | Ontology decision from David |
| WS9 | Deletions & consolidation | Risk reduction by subtraction | M (1–2 days) | Partly folded into WS1 |

Suggested order: **WS1 → WS4 → WS2 → WS3 → WS5**, then WS6/WS7 together (pre-real-money batch,
alongside roadmap Thread H), then WS8/WS9 as capacity allows.

---

## WS1 — Lease mutual exclusion (the critical)

**What's broken.** The lease-acquire upsert in 4 of 6 daemons sets `owner_id =
EXCLUDED.owner_id` **unconditionally** (the owner-or-expired CASE guards only the timestamps)
and `RETURNING owner_id = %s` therefore reports "acquired" to *any* contender. Two daemons
flip-flop ownership forever, each believing it holds the lease — reproduced against Postgres
during the review. This breaks the one invariant AGENTS.md calls firm
(one-connection-per-account), most dangerously for the Schwab websocket.
- Finding: **F-T3-1** (critical, confirmed) — [`REVIEW_NOTES.md:180`](REVIEW_NOTES.md); design context **F-T3-2** at :199
- Summary entry: [`REVIEW_SUMMARY.md`](REVIEW_SUMMARY.md) §Critical
- Affected files: `alpha_claw/trading/stream_lease.py:8`, `alpha_claw/schwab_stream/lease.py:9`,
  `alpha_claw/news_stream/lease.py:6`, `alpha_claw/sec_stream/lease.py:6`
- Already-correct reference implementations: `alpha_claw/market_events_stream/lease.py`,
  `alpha_claw/earnings_calendar_stream/lease.py` (the `DO UPDATE … WHERE owner-or-expired
  RETURNING true` form — no row updated ⇒ not acquired)

**Fix design.**
1. Create ONE shared module, e.g. `alpha_claw/leases.py`:
   `acquire_or_renew(conn, *, table, key_columns: dict, owner_id, lease_seconds) -> bool` and
   `release(conn, *, table, key_columns, owner_id)`, using the guarded upsert verbatim from
   `market_events_stream/lease.py` (parameterize table + conflict-key columns; keep per-daemon
   tables for now — collapsing to one `stream_leases` table is optional follow-up, see
   [`ARCHITECTURE_REVIEW.md:95`](ARCHITECTURE_REVIEW.md) §5).
2. Re-point all six call sites at it; delete the six local copies (this is the WS9 pattern —
   fix by subtraction).
3. **Add the contention test the correct variant never had** (F-T3-1 explicitly notes neither
   variant is tested): two owners, unexpired lease → second `acquire` returns False and
   `owner_id` is unchanged; expired lease → steal succeeds; same-owner renew extends. Use the
   same `_skip_unless_tables` DB-test pattern but against a **temp table** (see WS4 —
   do not touch live lease rows).

**Verification.** New unit test green; `docker compose up` all daemons + run a one-off
`docker compose run --rm --no-deps schwab-stream --once` and confirm it logs
"lease held by another owner" and does NOT connect (`stream_status.state = blocked`).

**Risk/rollout.** Behavior change is strictly tightening: a second contender now correctly
fails. Watch one daemon-restart cycle after deploy: on restart, the *new* process has a new
owner_id and must wait out the old lease TTL (or the old process's release) — confirm
`stream_lease_seconds` is short enough that a crash-restart doesn't stall reconnection beyond
tolerance. If it does, add a `release_lease` on SIGTERM (the daemons already do this) and keep
TTL as the crash backstop.

---

## WS2 — Cutover exit-correctness gates (before Thread A `route:true`)

These three must land before the cascade's order-routing moves to Entry/PositionWorkflows.
Tuesday's load test (entry-only, `route:false`) is NOT gated on this.

**2a. PositionWorkflow: a terminal-but-unfilled sell is not an exit.**
- Finding: **F-S2-1** (high, confirmed) — [`REVIEW_NOTES.md:229`](REVIEW_NOTES.md)
- What happens: `route_virtual_trade` returns `ok=True, status=routed_terminal` for ANY
  terminal order — including rejected/expired with zero fill (`virtual_portfolios/services.py:705`).
  `PositionWorkflow` gates only on `ok` (`temporal_execution.py:1694`), so a rejected sell closes
  the thesis and retires the position's only watcher. The EntryWorkflow already does this right
  by checking `ledger_entry` presence (`temporal_execution.py:1503`).
- Fix: in `PositionWorkflow.run`, treat `rr.ok and rr.ledger_entry` as exited; on
  `ok=True, ledger_entry=None` log loudly, record an `exit_sell_unfilled` decision (reuse
  `_record_exit`), and keep monitoring (same posture as the existing ok=False branch).
  Consider ALSO returning a distinct status from the router (`routed_terminal_unfilled`) so
  callers can't make this mistake — see WS6c for the shared result-contract change.
- Test: extend `position_workflow_example_test.py` with a rejected-sell fake router result
  (`ok=True`, terminal status `rejected`, no ledger_entry) → workflow keeps monitoring, thesis
  NOT closed. The existing `keeps_monitoring_when_sell_route_fails` test covers only ok=False.

**2b. Non-market entry orders can't route — resolve before cutover.**
- Finding: **F-S2-4** (self-documented in code) — [`REVIEW_NOTES.md:261`](REVIEW_NOTES.md)
- What happens: `EntryWorkflow` skips routing any non-market (extended-hours limit) buy because
  cancel/reconcile of a live limit order isn't built (`temporal_execution.py:1485`); after
  cutover, after-hours catalysts produce intent that never trades.
- Fix (minimum viable): on `order_not_terminal` for a routed entry limit order, cancel via the
  existing `cancel_order` + `wait_for_terminal_state`, then book any partial fill
  (`copy_alpaca_order_to_virtual_portfolio` is idempotent), and only then complete the
  workflow. That removes the double-entry window the comment fears (the symbol mutex holds
  while the workflow lives). Keep the skip-with-warning as the fallback for anything unhandled.
- Test: fake router returning `order_not_terminal` → cancel invoked → decision captured.

**2c. Don't let observability kill the live actor.**
- Finding: **F-S2-2** (medium, confirmed) — [`REVIEW_NOTES.md:241`](REVIEW_NOTES.md)
- What happens: `record_strategy_eval` is the only awaited activity in `LiveStrategyWorkflow.run`
  without try/except (`temporal_execution.py:1983`); 3 failed run-store writes (disk full, DB
  blip) fail the whole workflow.
- Fix: wrap in try/except like eval/routing/prune already are; log + continue (the run_id is
  idempotent, a later relaunch loses only that eval's record).
- Related (same patch, one line each): **F-S2-3** ([`REVIEW_NOTES.md:255`](REVIEW_NOTES.md))
  — only add a buy to `recent_entries` when the route result carries a `ledger_entry`
  (attempted ≠ filled), so a failed buy doesn't block re-entry for 30 evals.

---

## WS3 — Data lifecycle: three unbounded stores + a checker that can't see them

**What's broken.** Three artifact stores grow forever, and the guard that's supposed to catch
this class only inspects hypertables:
- `bar_window_snapshots`: one full bar-window JSONB row per live eval, **no delete path in the
  repo** — 68 MB / 2,696 rows live-verified. **F-S9-1** (high) — [`REVIEW_NOTES.md:377`](REVIEW_NOTES.md)
- `temporal_payload_store` (250 MB) and `data/strategy_runs/` (**2.4 GB**, 2,166 files) — no GC.
  **F-S11-1** — [`REVIEW_NOTES.md:447`](REVIEW_NOTES.md)
- Checker blind spot + un-waivered append-only tables (alpaca audit tables,
  `strategy_wake_decisions`, `strategy_entry_decisions`, `llm_call_costs`, `strategy_theses`):
  **F-T2-1** (broadened) — [`REVIEW_NOTES.md:81`](REVIEW_NOTES.md) and :387;
  **F-S5-3** — :364; run-artifact/index orphaning **F-M1-1** — :169
- Design context: [`ARCHITECTURE_REVIEW.md:95`](ARCHITECTURE_REVIEW.md) §5 (last bullet)

**Fix design** (extends Thread C; same philosophy — explicit, opt-in, no silent loss):
1. **bar_window_snapshots**: snapshots are worthless after their eval; keep a generous margin
   for replay/debugging. Add to `EXPECTED_TIMESCALE_RETENTION_POLICIES`-style handling: either
   convert to a hypertable + `add_retention_policy('2 days')` in a new migration, or (simpler)
   add a `prune_bar_window_snapshots(older_than)` batch to `strategy/data_gc.py` mirroring
   `prune_terminal_wake_events`, wired into the `strategy-data-gc` daemon args. Document the
   choice in the migration.
2. **temporal_payload_store**: GC must respect Temporal history retention (a claim referenced
   by replayable history must stay). Delete rows `created_at_utc < now() - (namespace retention
   + margin)` — look up the namespace retention (default 72 h on dev) and hard-code a
   conservative `>= 30 days` first pass as a data_gc policy. Content-addressing means a
   re-offload after deletion self-heals on the write side.
3. **data/strategy_runs/**: add a `prune-run-artifacts` mode to data_gc (or a small script):
   delete artifact files AND their `strategy_runs` index rows together (fixes the F-M1-1 orphan
   both directions), keeping N days or the newest K per strategy — **opt-in flags, dry-run
   default**, per the no-silent-truncation rule.
4. **Widen the guard** (`scripts/data_lifecycle_check.py`): parse every `CREATE TABLE` in
   migrations/ and require each table to have exactly one of: a Timescale retention policy, a
   data_gc policy registration, or an entry in `REGULAR_APPEND_ONLY_LIFECYCLE_NOTES` (waiver
   with rationale). Add waiver entries now for: the VP ledger + portfolios/instances (book of
   record, bounded by trade volume), `llm_call_costs`, `strategy_theses`,
   `strategy_entry_decisions` (small, decision audit), and explicit policies/waivers for the
   alpaca audit tables (wait_poll spam makes `alpaca_order_events` the one to watch —
   [`REVIEW_NOTES.md:81`](REVIEW_NOTES.md)).

**Verification.** `uv run python scripts/data_lifecycle_check.py --check` fails before, passes
after; dry-run each new GC mode against the dev DB and eyeball the candidate counts; then one
`--execute` pass and re-check table sizes (the review left a baseline:
[`REVIEW_NOTES.md:377`](REVIEW_NOTES.md) and the psql snapshot in the
S9 entry).

---

## WS4 — Test isolation + money-path coverage floor

**4a. Stop tests from consuming live coordination state.**
- Finding: **F-X3-1** (medium-high, confirmed — and demonstrated: the review's two suite runs
  acked 200 real pending wakes) — [`REVIEW_NOTES.md:797`](REVIEW_NOTES.md)
- Related: **F-V2-1** (silent skips make green ≠ tested) — :149; philosophy verdict in
  [`ARCHITECTURE_REVIEW.md:147`](ARCHITECTURE_REVIEW.md) §7
- Fix design:
  1. Immediate, surgical: the drain tests (`wake_emission_example_test.py`,
     `drain_loop_example_test.py` if applicable) must never claim real rows. Options, in order
     of preference: (a) run each DB test inside a transaction against **temp tables** cloned
     with `CREATE TEMP TABLE … (LIKE strategy_wake_events INCLUDING ALL)` + a `conn`-injection
     seam (most functions already accept `conn=`); (b) claim-scope: add an optional
     `sources: list[str]` filter to `fetch_pending_wake_events` and have tests emit+drain a
     test-only source (`source="test_<uuid>"`) — smallest diff, keeps the integration realism.
  2. Structural: a `ALPHA_CLAW_TEST_DATABASE_URL` env honored by `db.connect()` when set, plus
     a conftest fixture that fails loudly (not skips) when the test DB is absent in CI. This
     also fixes F-V2-1's "green without testing anything".
  3. Add a loud skip-summary: if any DB-backed test skipped, print a one-line warning count at
     session end.
- Note for the operator: the 200 acked wakes from the review runs were left as-is on purpose
  (restore SQL in the X3 note if wanted — [`REVIEW_NOTES.md:797`](REVIEW_NOTES.md)).

**4b. Coverage floor for `trading/services.py`.**
- Finding: T1 test-coverage note — [`REVIEW_NOTES.md:69`](REVIEW_NOTES.md)
  (32 test lines for 2,335 LOC); X4 map at :817
- Fix: unit-test the pure helpers first (no network, high value):
  `_marketable_limit_details`, `_compare_symbol_market_data`, `_format_limit_price`,
  `_qty_from_notional`, `_spread_bps`, `_parse_provider_timestamp`, `_managed_buy_status` /
  `_managed_close_status`, `_filled_qty`. Then the managed flows with injected fakes (the VP
  tests show the house pattern — `db_example_test.py` injects submit/wait/copy fns). Target:
  every WS6 fix below lands with its regression test.

---

## WS5 — Drain & plumbing resilience

**5a. One dead signal target must not crash the drain or strand the batch.**
- Finding: **F-S5-1** (medium, confirmed) — [`REVIEW_NOTES.md:351`](REVIEW_NOTES.md)
- Fix: in `live_dispatch.drain_and_signal_wakes`, wrap each `handle.signal(...)` in try/except:
  on `WorkflowNotFoundError`-shaped failures, log + skip the target (the wake is still acked —
  same semantics as "no matching workflow"); on transport errors, collect the affected wake ids
  and **ack only the successfully-signaled events**, leaving the rest `processing` for the
  reaper. Also wrap the `drain_once` call in `run_drain_loop` (drain_loop.py:201) so an
  unexpected exception logs and continues the loop rather than exiting the daemon.
- Test: fake client whose signal raises for one target → other targets signaled, batch acked
  minus the failed event, loop survives.

**5b. Connection discipline.**
- Findings: **F-D4-1** — [`REVIEW_NOTES.md:715`](REVIEW_NOTES.md);
  **F-S5-2** — :361; **F-T2-2** — :87
- Fix: introduce `psycopg_pool.ConnectionPool` behind `db.connect()` (same context-manager
  contract, pool per process, size ~4–8); batch the drain's acks into one
  `UPDATE … WHERE strategy_wake_event_id = ANY(%s)` per pass. Both are drop-in; do them
  together so the pool change is exercised by the highest-frequency caller.
- Optional same-batch one-liners: `lru_cache` on `load_settings` (**F-D4-2** — :719) and a
  `pg_advisory_lock` around `apply_migrations` (**F-D4-3** — :722).

---

## WS6 — Trading-layer hardening (pre-real-money batch; pairs with Thread H)

**6a. `is_tradeable_on_alpaca` cache poisoning.**
- Finding: **F-T1-1** (high, confirmed) — [`REVIEW_NOTES.md:16`](REVIEW_NOTES.md)
- Fix: only cache *definitive* answers. Catch the SDK's not-found (404) → cache False;
  `asset.tradable` → cache the boolean; any other exception → do NOT cache, return a
  conservative False for this call, and log. Add a TTL (e.g. 12 h) so listing changes are
  eventually picked up. Note `drain_gates.py:11` documents the old hazard — update that comment
  when fixed so the Chesterton's-fence note doesn't outlive the fence.
- Test: fake client raising a transport error → second call re-queries (no poisoned cache);
  404 → cached False; tradable → cached True.

**6b. `close_and_confirm` account-position aliasing.**
- Finding: **F-T1-2** (high, confirmed) — [`REVIEW_NOTES.md:25`](REVIEW_NOTES.md)
- Fix: when `portfolio_id` is provided, size the sell from the **VP book quantity**
  (`load_open_positions(portfolio_id, [strategy_instance_id])`, capped by the account position),
  not the whole Alpaca position; refuse (escalate) if book qty > account qty (drift needs a
  human, not a bigger sell). Without `portfolio_id`, current whole-position behavior is fine —
  it's an operator tool — but say so in the docstring.
- Test: fake positions (account 15, book 10) → sell qty 10; book 10 / account 5 → escalation.

**6c. Retry-hostile managed result contract.**
- Findings: **F-T1-3** — [`REVIEW_NOTES.md:35`](REVIEW_NOTES.md);
  **F-T1-4** — :44; supporting **F-T1-6/7** — :57/:60
- Fix: (1) split the managed result into `broker_ok` + `ledger_ok` (keep top-level `ok` =
  both, for compat) and make `virtual_fill_copy_failed` carry `broker_ok=true` +
  `remediation: "copy_order_fill --client-order-id …"`; (2) wrap the `wait_for_terminal_state`
  call inside buy/close_and_confirm in try/except → escalate + return a managed result instead
  of raising through with no escalation (F-T1-4); (3) add the missing account-blocked check to
  `close_and_confirm` (F-T1-6); (4) in `_recover_non_terminal_managed_order`, after a failed
  cancel, re-fetch the order once and copy any fill before returning (F-T1-7).
- The account-level TOCTOU (**F-T1-5** — :51) should be fixed *once with* the Thread-H
  sell-guard race, not separately: one `pg_advisory_xact_lock` keyed
  `(account_id, symbol)` taken by both the VP router's guard-and-submit section and the managed
  flows' check-and-submit section. Note in the Thread H work item.

---

## WS7 — Ledger integrity (pairs with Thread H)

**7a. Reconciliation entries must use the structural dedupe key.**
- Finding: **F-S10-1** (medium, confirmed) — [`REVIEW_NOTES.md:409`](REVIEW_NOTES.md)
- Fix: `reconcile_late_fills` passes `source_type='alpaca_fill'` (or keep `'reconciliation'`
  but then ALSO check for an existing `alpaca_fill` row for the same order) and
  `source_id=_alpaca_order_source_id(...)`-shaped identity
  (`alpaca_order:{account}:{order_id}`) so the per-book unique index + advisory lock apply
  across the stream-copy and reconciler paths. Migration note: existing `'reconciliation'` rows
  keep their provenance; only new writes change. Also switch its money math to Decimal
  (**F-S10-2** — :420).
- Test: book via the copy path, then run the reconciler for the same order → no second row
  (today it would double-book).

**7b. Instance-scoped sell guard.**
- Finding: **F-V1-1** (medium, confirmed) — [`REVIEW_NOTES.md:108`](REVIEW_NOTES.md)
- Fix: in `route_virtual_trade`, when `strategy_instance_id` is provided, compute the guard
  quantity from the instance-scoped fold (`load_open_positions(portfolio_id,
  strategy_instance_id=…)`) instead of the book-level `_virtual_position_quantity`; keep the
  book-level check as a second ceiling. This makes the docstring's promise
  ("a strategy sells only what it opened", services.py:1313) true.
- Test: two instances in one book; B sells A's symbol → `virtual_sell_guard_blocked`.
- Coordinate with Thread H's read-then-submit race fix (same code region; do in one change).
- Same-file one-liner: fix the `datetime.min` sort-key crash fallback (**F-V1-3** — :125) with
  `datetime.min.replace(tzinfo=timezone.utc)`.

**7c. Route failures should escalate.**
- Finding: **F-V1-4** — [`REVIEW_NOTES.md:130`](REVIEW_NOTES.md); design
  context "single escalation chokepoint" in
  [`ARCHITECTURE_REVIEW.md:59`](ARCHITECTURE_REVIEW.md) §3
- Fix: add `record_trade_escalation` to `route_virtual_trade`'s failure returns
  (`submit_failed`, `order_not_terminal`, `virtual_fill_copy_failed`) — the escalations table is
  already the human-attention queue for exactly these. Cheap now; structural chokepoint later.

---

## WS8 — Language & validation honesty (one design decision needed)

**8a. One-line runtime fix: run I/O nodes in topological order.**
- Finding: **F-S1-1** (medium, confirmed-latent) — [`REVIEW_NOTES.md:248`](REVIEW_NOTES.md)
  and resolution at :574
- Fix: `_evaluate_strategy` builds `io_node_ids` from `compile_result.topological_order`
  (filtered to activity kinds) instead of spec declaration order (temporal_execution.py:2056).
  The compiler already computes it. Add a compiler *warning* when declaration order ≠ topo
  order so authors see it at validate-spec time too.

**8b. Stop the safety checks from overpromising.**
- Finding: **F-E1-1** (medium, confirmed) — [`REVIEW_NOTES.md:548`](REVIEW_NOTES.md)
- Fix, two parts: (1) rename the check *labels/messages* to say what they do
  (`no_lookahead_declared`, etc.) — keep check IDs stable if catalogs reference them, change the
  human text; (2) add one behavioral probe that actually verifies: a runner-level
  truncation-invariance property test on fixtures (node outputs at bar *i* must not change when
  bars > *i* are removed). Run it in `validate` (bundle-level) rather than per-eval. This is the
  cheapest real lookahead detector and it covers custom nodes the lint can't know about.
- Doc follow-up: soften ASSISTANT.md's "the neck is what makes the result … point-in-time-safe"
  to name where the real guarantees live (data-layer as_of + runner semantics) — see the C1–C6
  doc-drift entry ([`REVIEW_NOTES.md:768`](REVIEW_NOTES.md) area).

**8c. Backtest fill-timing honesty.**
- Finding: **F-E9-1** (medium, confirmed) — [`REVIEW_NOTES.md:652`](REVIEW_NOTES.md)
- Decision then fix: either (a) execute at next bar's **open** — change `_simulate_equity` to
  apply new weights to `open[i]→close[i]` for the transition bar (needs open prices, which
  MarketBar has), or (b) keep close-fills but relabel `execution_timing` to
  `fill_at_signal_close` and note the daily-frequency gap bias in the runner docstring +
  robustness scorecard. (a) is more work and more correct; (b) is honest immediately. Do (b)
  now, schedule (a) with the next backtest-quality push.

**8d. The ontology decision (needs David).**
- Finding: **F-E3-1** (design) — [`REVIEW_NOTES.md:602`](REVIEW_NOTES.md);
  full argument in [`ARCHITECTURE_REVIEW.md:10`](ARCHITECTURE_REVIEW.md) §1
  and §2; grammar-doc drift **F-C6-1** — [`REVIEW_NOTES.md:768`](REVIEW_NOTES.md)
- The decision: **implement `action` or shrink to the seven real kinds.** Recommendation from
  the review: don't build `action` speculatively; (1) mark `data`/`action`/`notify`/`report`/
  `gate` with 🔮 in `docs/design/agentic-strategy-dag-grammar.md` + ARCHITECTURE Known Gaps
  (30-minute doc fix, closes F-C6-1); (2) pull sizing policy INTO the spec as a first-class
  section (e.g. `execution: {sizing: equity_weight, max_position_weight, whole_shares: true}`)
  that `_orders_from_decision` reads — this moves the semantics into the inspectable object
  without inventing a node kind; (3) revisit a real `action` node only when a second
  venue/market forces the interface (see the extensibility note in
  [`REVIEW_SUMMARY.md`](REVIEW_SUMMARY.md) §Extensibility).

---

## WS9 — Deletions & consolidation (risk reduction by subtraction)

Design case: [`ARCHITECTURE_REVIEW.md:95`](ARCHITECTURE_REVIEW.md) §5 and
the keep/evolve/rework table (:185).

| Target | Finding | Action |
|---|---|---|
| 6 lease copies | F-T3-2 ([`REVIEW_NOTES.md:199`](REVIEW_NOTES.md)) | Done by WS1 (shared module) |
| `strategy/durable_execution.py` + migration-013 table | F-S6-1 (:431) | Delete, or add a module docstring declaring it the non-Temporal seed + a contract test shared with the live path. Decide; don't leave ambient. |
| `strategy/live_worker.py` (deprecated in-process worker) | F-S8-1 (:501) | Move under a `legacy_` name or delete; its raw-L1 default is the documented trap. |
| Module-level `load_live_bars` AUTO-warmup path | F-S7-2 (:479) | Delete or pin to SCHWAB — one warmup policy. |
| `buy_and_confirm`/`close_and_confirm` twins | T1 notes (:10) | Extract the shared managed-order pipeline while doing WS6c (guards → submit → wait → recover → verify → copy as composable steps). |
| 5 stream-daemon chassis | F-D2-1 (:705) | One `StreamDaemon` base (tick loop, lease via WS1 module, status upsert, signals); per-source adapters keep their logic. Do LAST — highest churn, purely structural. |
| `virtual_portfolio_positions` view basis columns | Thread H + M1 note (:161) | When fixing Thread H, prefer DROPPING `average_entry_price`/basis columns from the views (quantity + gross flows only) over re-implementing fold logic in SQL — one basis implementation. |

Smaller items to fold in opportunistically: F-S3-2 NULL-symbol dedupe in
`strategy_entry_decisions` (:313 — default `''` or `NULLS NOT DISTINCT`), F-S3-3 visibility-scan
prefix query (:320), F-E7-1 usage-reporting lint (:641), F-T3-3 optimistic "connected" status
(:206), F-S7-1 sparse-symbol window alignment (:470 — revisit if illiquid holdings appear).

---

## Deliberately deferred (recorded, no action planned)

Nits kept for honesty, not scheduled: F-T1-8 (µs client_order_id collision), F-T2-3 (payload
merge shadowing), F-V1-5 (mandate float), F-V1-6 (404 string-match — fail-closed), F-S2-7
(alphabetical buy priority), F-S2-8 (payload size estimator), F-S7-3 (lookback name-hints),
F-S11-2 (run_id path chars), F-S13-1 (displacer provider bypass — revisit at Thread J phase 5),
F-D4-2/3 (unless done with WS5b), F-X5-1 (verified clean, no action). All findable by F-ID in
[`REVIEW_NOTES.md`](REVIEW_NOTES.md).

---

## Verification protocol (applies to every workstream)

1. `uv run pytest -m "not live"` — **after WS4 lands, this must be side-effect-free**; until
   then, do not run the drain tests while the outbox has a live backlog (F-X3-1).
2. `uv run python scripts/data_lifecycle_check.py --check` (WS3 makes it strict).
3. `uv run alpha strategy live preflight …` before any live session — it already checks table
   sizes, retention policies, task-not-found rate; WS3/WS5 changes should show up green there.
4. Money-path changes (WS2/6/7): add the regression test in the same commit; prefer the VP
   test pattern (injected fakes + real DB via the WS4 isolation seam).
5. Per AGENTS.md: compile/import checks minimum; migrations applied cleanly on a scratch DB;
   note untested live-provider paths explicitly.

## Roadmap integration (suggested, not applied)

Per the roadmap's own convention (update in the same change that advances a thread): add a
thread for WS1+WS5 ("runtime invariants: lease + drain resilience"), fold WS2 into **Thread A**
as explicit pre-cutover checklist items, fold WS3 into **Thread C** (it is Thread C's missing
chapter), fold WS6+WS7 into **Thread H** (rename it "pre-scale/pre-real-money correctness"),
and note the WS8d ontology decision under "Later / deferred / gated" until decided. This plan
doc then becomes the detail link those entries point at.
