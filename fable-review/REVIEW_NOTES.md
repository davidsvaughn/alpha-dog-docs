# Review notes — append-only log (run 2026-07-06)

Format per entry: date, item, findings (each with file:line, concrete failure scenario,
confidence: confirmed|suspected), severity (critical|high|medium|low|nit). Design-level
symptoms are cross-posted to ARCHITECTURE_REVIEW.md.

Finding IDs: F-<item>-<n> (e.g. F-T1-1) so CROSS_FLAGS and the summary can reference them.

---
## 2026-07-06 — T1: trading/services.py (depth: full)

Context established: the live strategy routes orders via `temporal_execution._route_virtual_trade`
→ `virtual_portfolios.services.route_virtual_trade` (reviewed under V1). `buy_and_confirm` /
`close_and_confirm` are the agent/CLI-facing managed flows. Both funnel into `submit_order`.

- **F-T1-1 (high, confirmed): `is_tradeable_on_alpaca` caches `False` on any transient error,
  permanently per process.** services.py:131–154 — bare `except Exception: ok = False` then
  `_ALPACA_TRADEABLE_CACHE[key] = ok`; no TTL, no error/absence distinction. Called on the live
  routing path at temporal_execution.py:919 inside a long-lived worker. Failure scenario: one
  Alpaca 500/timeout while routing NVDA → NVDA cached untradeable → every subsequent entry for
  NVDA silently `rejected` until the worker restarts. drain_gates.py:11 *names* this exact
  poisoning as the reason the check is kept out of the drain — the hazard is known but unfixed at
  the layer that actually gates orders. Fix shape: cache only successful lookups (or cache errors
  with short TTL), distinguish 404-not-found (cacheable False) from transport errors (don't cache).
- **F-T1-2 (high, confirmed): `close_and_confirm` liquidates the whole account-level position,
  then credits the entire fill to one virtual portfolio.** services.py:1403–1404 takes
  `qty = starting_positions[0].qty` from the *Alpaca account* position, not the VP's book
  quantity; the fill copy (services.py:1491–1500) then books that qty into `portfolio_id`.
  Failure scenario: two VPs (or a VP + a manual/operator position) hold AAPL on the same paper
  account; `close_and_confirm(symbol=AAPL, portfolio_id=A)` sells *all* shares and books a sell
  larger than VP A's holding → VP A over-sold (negative/short ledger position), VP B's real
  shares gone with no ledger event. Single-account/single-book today masks it; breaks the moment
  account↔portfolio stops being 1:1 (which multi-strategy on one paper account implies). Verify
  in V1 how route_virtual_trade sizes sells by contrast.
- **F-T1-3 (medium, confirmed): `virtual_fill_copy_failed` flips a genuinely-filled order to
  `ok=False`, inviting caller retries that would double-trade.** services.py:1259–1261 and
  1502–1504: broker fill succeeded, only the ledger copy failed, yet the result reads as overall
  failure with no distinguishing flag beyond the status string. Any caller/agent that retries a
  "failed" buy resubmits a *new* order (client_order_id is regenerated per call,
  services.py:1059). Escalation row is written, which mitigates for humans; the contract is
  still retry-hostile. Fix shape: separate `broker_ok` from `ledger_ok` in the result, and/or
  make the copy retryable standalone (it is: copy_alpaca_order_to_virtual_portfolio by
  client_order_id) and say so in the status.
- **F-T1-4 (medium, confirmed): unhandled exception from `wait_for_terminal_state` aborts the
  managed flow with no escalation and no VP fill copy.** services.py:1227 / 1470 call it bare;
  itself raises TradingError on any poll/DB error (services.py:840–851). Failure scenario:
  submit succeeds → first status poll hits a transient network error → TradingError propagates
  out of buy_and_confirm → no managed_ event, no escalation, no virtual fill copy; the filled
  order exists only at Alpaca until reconcile/late-fill machinery notices. Mitigated by
  reconcile_late_fills (S10) but the flow's own promise ("and_confirm") silently breaks.
- **F-T1-5 (medium, confirmed): open-order guard is read-then-submit (TOCTOU) at the account
  level.** services.py:1096–1113: `get_orders(open)` then `submit_order` with no lock/atomic
  reservation; two concurrent managed buys for the same symbol both pass the check and both
  submit. Same family as the tracked Thread-H sell-guard race (VP router) — the pattern spans
  layers, worth fixing once with a shared per-(account,symbol) advisory lock rather than
  point-fixing the VP router only.
- **F-T1-6 (low, confirmed): `buy_and_confirm` checks `trading_blocked` flags; `close_and_confirm`
  never checks account status.** services.py:1082–1094 vs 1325ff. A blocked account fails later
  at submit with a less actionable escalation; asymmetry looks accidental.
- **F-T1-7 (low, confirmed): managed-flow timeout recovery can miss a fill that lands after the
  poll timeout but before cancel.** services.py:1658–1699: cancel of an already-filled order
  raises → reconcile snapshot → returns the stale non-terminal result; `_copy_virtual_fill…`
  then sees filled_qty=0 and skips the ledger copy even though the broker filled. Escalation is
  recorded (ok=False) and reconcile_orders wrote the true state to audit, so recoverable — but
  the VP ledger is wrong until the late-fill reconciler runs.
- **F-T1-8 (nit): `_managed_client_order_id` is wall-clock-microsecond based** (services.py:1610)
  — two concurrent same-action-same-symbol calls in the same µs collide; astronomically unlikely
  today, trivially avoidable with a uuid suffix.
- **Test coverage (design-level, cross-posted to ARCHITECTURE_REVIEW §7):** 2,335 LOC of order
  flow; the colocated test is 32 lines covering two argument-validation branches. All managed-flow
  logic (guards, recovery, alignment math, marketable-limit pricing, VP copy wiring) has zero
  `not live` coverage. The alignment/limit-pricing helpers are pure and eminently unit-testable —
  this is the clearest instance of the `_example_test` philosophy under-testing money logic.

## 2026-07-06 — T2: trading/audit.py + migrations 010/011 (depth: full)

- Good: fill snapshots are idempotent via unique index (account, order, filled_qty,
  filled_at)+ON CONFLICT DO NOTHING (010:…fill_snapshot_uidx; audit.py:312); `submit_requested`
  is recorded *before* the broker call (fail-closed if audit DB is down); escalations carry full
  steps+raw payloads.
- **F-T2-1 (medium, confirmed): no lifecycle/retention story for `alpaca_order_events` /
  `alpaca_fill_events` / `alpaca_trade_escalations`.** wait_for_terminal_state writes a full-raw
  `wait_poll` row every poll_interval (2 s default) per in-flight order (services.py:857–864),
  and every reconcile run re-records every order it sees (services.py:427–435). AGENTS.md
  mandates a lifecycle story per append-only table; check against data_gc expectations in S9 —
  flagged. Same bloat class that produced the 12 GB schwab_l1_events incident.
- **F-T2-2 (low, confirmed): one fresh DB connection per audit row** (audit.py:29, 278 —
  `connect()` per call, twice per event when a fill snapshot fires). In the 2 s poll loop this is
  ~1 conn/s per order. Paper-scale fine; will matter under concurrency (Thread A's parallel
  EntryWorkflows all write audit rows through this path).
- **F-T2-3 (nit): `record_order_event` merges `{**payload, **summary}`** (audit.py:27) — summary
  keys silently shadow raw-payload keys of the same name for column extraction; currently
  harmless because summaries are subsets of raws, but it's an implicit contract.

## 2026-07-06 — V1: virtual_portfolios/services.py (depth: full)

Positive findings worth stating: all money math is `Decimal` end-to-end; ledger appends are
idempotent (advisory xact lock + per-book unique indexes on source_id / source_fill_event_id +
mismatch-raise on conflicting replays, services.py:864–904); `route_virtual_trade` does a
pre-submit reconcile by client_order_id (services.py:549–596) so replays with a *stable*
client_order_id are safe; binding is validated before any broker submit (services.py:420);
`_terminal_fill_event_for_order` accepts canceled/expired snapshots with filled_qty>0, so
partial-fill-then-cancel *is* booked (services.py:1727–1728).

- **Thread H status check (tracked, still open):** sell guard is read-then-submit
  (services.py:429–439 vs submit at :599) — confirmed unchanged; `virtual_portfolio_positions`
  view still lifetime-average (see M1). Not re-reported as new.
- **F-V1-1 (medium, confirmed): the sell guard is book-scoped, never instance-scoped — one
  strategy instance can sell another instance's shares.** services.py:430
  `_virtual_position_quantity(portfolio_id, symbol)` ignores `strategy_instance_id` even when
  provided. Failure scenario: instances A and B share a book; B (or an operator CLI call with
  B's instance id) sells a symbol only A holds → guard passes on book quantity → fill booked
  with strategy_instance_id=B → per-instance folds (`load_open_positions(strategy_instance_id=…)`,
  used by live exits at temporal_execution.py:~950) now show A still long (shares gone) and B
  short-but-hidden (long-only filter drops it). Book-level totals stay right, so account
  reconciliation never flags it. The docstring at services.py:1313–1316 promises "a strategy
  manages/sells only the positions it opened" — the guard doesn't enforce that promise.
- **F-V1-2 (low-medium, confirmed): the same real fill can be booked into two different books.**
  Uniqueness is per-(portfolio, source); nothing claims a fill globally. Scenario: routed fill
  booked to book A; a later manual `virtual_portfolio_copy_order` with the wrong --portfolio-id
  books the identical fill into B → both ledgers hold the same real shares. Mitigated:
  `reconcile_virtual_portfolio_positions` sums books per account and would flag virtual>real
  drift — but only when someone runs it. A cross-book uniqueness (or an explicit
  "fill claimed by book X" table) would make the invariant structural.
- **F-V1-3 (low, confirmed): `fold_ledger`/`fold_open_positions` sort key crashes on a NULL
  received_at_utc row.** services.py:1158–1163 uses `row.get("received_at_utc") or datetime.min`
  — naive `datetime.min` is unorderable against tz-aware timestamps (TypeError). Schema makes
  the column NOT NULL so it's latent; the fallback as written can only ever crash, which
  suggests it wasn't exercised. Same pattern duplicated at :1247–1251.
- **F-V1-4 (low, confirmed): route failure paths don't escalate.** Unlike the managed flows
  (`_managed_order_result` → `record_trade_escalation`), `route_virtual_trade`'s failure returns
  (`order_not_terminal`, `virtual_fill_copy_failed`, `submit_failed`) write no escalation row;
  visibility depends on the caller persisting the result. Check in S1 what the live activity
  does with these (flagged).
- **F-V1-5 (nit): `load_strategy_mandate` converts allocated_capital to float** (services.py:1372)
  — the one place Decimal discipline breaks; it feeds sizing so imprecision is tolerable, but
  it's an inconsistency.
- **F-V1-6 (nit): `_looks_like_order_not_found` string-matches exception text** (services.py:1508)
  — brittle across alpaca-py versions; a changed 404 message turns a benign not-found into
  `pre_submit_reconcile_failed` (fail-closed, so safe direction).

## 2026-07-06 — V2: virtual_portfolios tests + tools.py (depth: full)

- Coverage is genuinely good here (contrast T1): fold semantics incl. partial-sell average-cost
  invariance, entry_time open/close/re-open transitions, point-in-time cutoff; route sell-guard
  block, binding validation before submit, terminal-fill booking, pre-submit reconcile replay;
  multi-book pooled-account independence + replay reconciliation; a durable-workflow
  submits-once/books-once test. This is the standard the trading/ tests should be held to.
- **F-V2-1 (low, confirmed): DB-backed tests skip silently when Postgres is down or migrations
  unapplied** (db_example_test.py:27–45 `pytest.skip`). `-m "not live"` green therefore does NOT
  imply the ledger/router logic ran — on a machine without the Compose DB the whole money-path
  suite quietly skips. A CI job that doesn't start Postgres would look green while testing
  nothing here. Worth a required-DB CI lane or a loud skip summary.
- tools.py: thin argparse wrappers over services; JSON-in/JSON-out; no findings.

## 2026-07-06 — M1: migrations 015/016 (depth: full)

- Schema quality is high: CHECKed enums, composite FK (strategy_instance_id, portfolio_id) so a
  ledger row can't bind an instance to the wrong book, partial unique indexes for idempotency,
  sensible covering indexes.
- **Thread H (tracked, confirmed open at schema level):** `virtual_portfolio_positions.average_entry_price`
  = lifetime `sum(buy cost)/sum(buy qty)` (015: view body) — never resets when a position closes
  and reopens, so it diverges from `fold_ledger`'s lot-aware basis after any sell+rebuy. Both
  per-book and per-instance views share the flaw. Roadmap Thread H's plan (make the view match
  fold_ledger) is the right fix; consider *dropping* the derived columns from the view instead
  and keeping quantity-only (views recomputing basis in SQL will keep re-diverging as fold rules
  evolve — one basis implementation, in code, is the safer shape). Cross-posted to
  ARCHITECTURE_REVIEW §3 (state placement: derived financial state duplicated in SQL).
- **F-M1-1 (low, confirmed): `strategy_runs.artifact_path` points at unmanaged filesystem state.**
  016 keeps results as files with a DB index (good pattern) but nothing enforces existence/GC
  pairing; a deleted run directory silently orphans the index row (and vice versa). Acceptable
  for now; note for the lifecycle sweep (C5).
- 015 ledger has no retention policy — correct: it IS the book of record (bounded by trade
  volume), consistent with AGENTS.md "unbounded-on-purpose" waiver class. No finding.

## 2026-07-06 — T3: trading/tools.py + stream_daemon.py + stream_lease.py + stream_status.py
(depth: full for daemon/lease; skim for tools.py/stream_status.py — argparse wrappers and a
status upsert, nothing money-bearing)

- **F-T3-1 (CRITICAL, confirmed — reproduced in SQL): the stream lease does not mutually
  exclude. Any contender "steals" the lease while it is still held, and both owners are told
  they acquired it.** trading/stream_lease.py:8–41 — in the upsert, `owner_id =
  EXCLUDED.owner_id` is set **unconditionally**; the owner-or-expired CASE guards only the
  timestamps; `RETURNING owner_id = %s AS acquired` therefore always returns true for whoever
  ran last. Reproduced against Postgres (temp table, exact upsert shape): B acquires while A's
  lease is unexpired → owner=B, acquired=true; A's next heartbeat → owner=A, acquired=true.
  Two daemons flip-flop ownership indefinitely, each streaming concurrently.
  **The same broken SQL is copy-pasted into `schwab_stream/lease.py`, `news_stream/lease.py`,
  and `sec_stream/lease.py`** — i.e. the *Schwab* one-connection-per-account lease, which
  AGENTS.md and the roadmap name as the single firm invariant, does not enforce anything.
  Meanwhile `market_events_stream/lease.py` and `earnings_calendar_stream/lease.py` carry the
  **correct** variant (`DO UPDATE … WHERE owner-or-expired RETURNING true` — no-match ⇒ not
  acquired). Failure scenario (realistic, not exotic): operator runs a one-off
  `docker compose run … schwab-stream` or `--once` debug tick while the service replica is up →
  both connect; Schwab force-drops one side, daemons reconnect-fight, subscriptions and
  receive-time stamping interleave; for alpaca_trade_stream both sides also double-write
  `stream_*` audit events. Fix: adopt the WHERE-guarded form everywhere (one shared helper —
  see design note), and add the contention unit test the correct variant never got.
- **F-T3-2 (design, cross-posted §5/§8): six hand-copied lease implementations for one
  concept.** Four tables (`stream_leases`, `news_source_leases`, `sec_stream_source_leases`,
  `market_event_stream_source_leases`, `earnings_calendar_stream_source_leases`…), two SQL
  shapes, one of them wrong — the copy-paste is *how* the bug survived: it was fixed (or written
  correctly) in the two newest daemons and never back-ported. One `stream_leases(source,
  account_id)` table + one shared acquire/release module would make the invariant a single
  point of correctness.
- **F-T3-3 (low, confirmed): daemon marks state "connected" before the websocket is up.**
  stream_daemon.py:179 sets `state.connected = True` when the thread *starts*; auth/connect
  failures surface only when the thread dies a tick later. Status consumers (preflight?) see a
  brief false "connected". Also `_on_trade_update` does blocking DB writes (2 fresh connections)
  on the stream's async callback — fine at paper volume, noted for scale.
- Cross-flag resolved: the trade-stream daemon *does* feed the audit tables
  (record_order_event with `stream_<event>` types → `_latest_stream_terminal_event` and, via
  filled snapshots, `_terminal_fill_event_for_order`), and wait_for_terminal_state degrades to
  REST polling when the daemon is down — no hard dependency. Confirmed sound.

## 2026-07-06 — S1+S2: strategy/temporal_execution.py (activities + workflows) (depth: full)

Quality note first: this file shows the strongest engineering discipline in the repo so far —
deterministic client_order_ids minted workflow-side (`{workflow_id}-{launch_id}-eval-{n}-{side}-{symbol}`,
:1113; exits keyed per lifecycle via entry_key, :1683), `launch_id` explicitly designed against
relaunch collisions (:226–230), `workflow.patched()` for replay-safe evolution (:1830),
continue-as-new on Temporal's suggestion (history-cap incident encoded at :1771), completed-bar
as_of computed deterministically from workflow.now() (:1228), an in-flight entry guard bridging
the fill→snapshot lag (:1923–1931), and a payload-blowup guard that trims with explicit markers
and loud logs rather than silently (:2167). Cross-flags resolved: client_order_id stability ✓
(fully idempotent with the router's pre-submit reconcile); route-failure handling = logged +
run-store/decision-capture, still no escalation row (folded into F-V1-4).

- **F-S2-1 (high, confirmed): PositionWorkflow treats any *terminal* sell as a successful exit —
  a rejected/expired/zero-fill sell closes the thesis and retires the workflow while the position
  is still held.** temporal_execution.py:1694 gates on `rr.get("ok") is True`, but
  `route_virtual_trade` returns ok=True/status=routed_terminal for ANY terminal order, including
  rejected/canceled/expired with filled_qty=0 (services.py:705–713 skips the copy and still
  returns ok). The EntryWorkflow handles exactly this by checking `ledger_entry` presence
  (:1503–1529, filled vs no_fill); PositionWorkflow doesn't. Failure scenario (post-cutover,
  entry_route=true): VDD stop fires → market sell submitted → Alpaca rejects (halted symbol at
  submit, blocked account, fat-finger guard) → wait sees terminal `rejected`, ok=True → thesis
  closed, workflow returns "exited" → the held position has NO PositionWorkflow watching it and
  no thesis; it drifts unmanaged until a human notices. Fix: mirror the EntryWorkflow's
  `ledger_entry`-presence check before declaring the exit done; keep monitoring otherwise.
- **F-S2-2 (medium, confirmed): `record_strategy_eval` failure kills the long-lived actor.**
  :1983–1997 — the only awaited activity in LiveStrategyWorkflow.run not wrapped in try/except
  (eval, routing, prune are all guarded). 3 failed attempts to write the run-store (full disk,
  transient DB outage — it writes both a file and a DB row) → exception → the live strategy
  workflow FAILS. Recording is observability; it shouldn't be load-bearing for actor liveness.
  Same shape in EntryWorkflow for load_live_bars (:1438) — there a failure arguably *should*
  fail the entry, so only the LiveStrategyWorkflow/PositionWorkflow instances are findings.
- **F-S1-1 (medium, suspected): I/O nodes execute in YAML declaration order, not topological
  order.** `_evaluate_strategy` at :2056 collects `io_node_ids` from `spec["nodes"]` list order,
  then runs them sequentially seeding each into the next. An I/O node consuming another I/O
  node's output (data → agentic, agentic → agentic) declared *before* its producer would
  assemble kwargs against a missing seed and fail the eval. Whether this is reachable depends on
  validation enforcing declaration order ≈ topo order — verify in E2 (flagged). If validation
  doesn't enforce it, this is a spec-authoring landmine.
- **F-S2-3 (low-medium, confirmed): a failed buy still poisons re-entry for 30 evals.** The
  cascade routing loop marks any non-"rejected" route status as `routed` (:1953–1964), including
  ok=False outcomes (submit_failed / order_not_terminal), and enters the symbol into
  `recent_entries` — so a buy that never filled blocks re-entry until the
  `_ENTRY_INFLIGHT_COOLDOWN_EVALS=30` backstop expires (~30 min at 1m cadence). Deliberate
  backstop, but the status taxonomy conflates "attempted" with "succeeded".
- **F-S2-4 (low, confirmed, self-documented): entry routing supports market orders only** —
  non-market (extended-hours) entries are skipped with a warning (:1485–1495) because
  cancel/reconcile of a live limit order isn't built. In-code "review finding", not in the
  roadmap: after cutover, an after-hours catalyst produces intent that can never route. Belongs
  on Thread A's checklist before `route:true`.
- **F-S2-5 (low, suspected): PositionWorkflow feeds yesterday's peak to the trailing-stop check.**
  check_position_exit computes `new_peak = max(payload.peak, peak_since_entry)` (:976) but calls
  `manage_exits_vdd` with `peak_by_symbol={symbol: payload.peak}` — the *old* peak (:983). If
  manage_exits_vdd treats peak_by_symbol as authoritative (E4 will tell), a run-up + ≥trail%
  crash within one cadence window measures the trail from the stale peak → exit delayed a tick.
  Verify in E4 (flagged).
- **F-S2-6 (low, suspected): wake dedupe (`_seen`) resets across continue-as-new** (:1739–1749)
  — a wake redelivered after a CAN boundary would double-process. Whether redelivery can happen
  depends on drain-side outbox semantics — verify in S4/S5 (flagged).
- **F-S2-7 (nit): buys are prioritized alphabetically under cash constraint**
  (`sorted(desired.items())`, :1146) — when several candidates arrive in one eval and cash runs
  out, AAPL beats ZM by ticker, not by conviction. Deterministic but semantically arbitrary.
- **F-S2-8 (nit): `_estimate_jsonable_size` extrapolates list size from the last element**
  (:2160) — a list with one huge early element under-measures, skips the trim, and the payload
  cap error it guards against still fires. Fail-visible, edge-case only.
- Design observations cross-posted to ARCHITECTURE_REVIEW: §2 (the Temporal separation largely
  *does* hold — workflow bodies are dict-orchestration only; but LiveStrategyWorkflowInput now
  carries 30+ fields mixing ontology (spec, universe) with backend tuning (timeouts, payload
  caps, CAN bounds) — the "one backend among many" story is thinning); §4 (this one file holds
  activities, 6 workflow classes, sizing, reconciliation, payload guards — it is the de-facto
  runtime kernel and wants decomposition).

## 2026-07-06 — S3: entry_spawn.py + entry_decisions.py + temporal_launch (command builders) +
entry/position workflow tests (depth: full)

Cross-flag resolved (double-entry mutex): the "symbol-keyed mutex" = Temporal workflow-id
uniqueness — EntryWorkflow id `live-strategy/{instance}/entry/{SYMBOL}` +
`id_conflict_policy=USE_EXISTING` + `ALLOW_DUPLICATE` reuse (entry_spawn.py:135–148). Correct
even under Temporal visibility lag (a re-spawn attaches instead of duplicating). The drain also
pre-filters: held symbols classify as held_candidate, per-instance in-flight cap + per-cycle cap
(fails closed when the running set can't be measured — the 07-02 pile-up lesson encoded).
Broker idempotency is durable: build_entry_command folds the wake's (source, source_event_id)
into launch_id (temporal_launch.py:185–190), so a wake redelivered after a drain crash re-mints
the SAME client_order_id and the router reconciles. Test coverage here is exemplary (caps,
strict route-gate truthiness, failure isolation, durable-token durability, sizing parity,
route-fail-keeps-monitoring).

- **F-S3-1 (medium, suspected): post-fill re-entry window between EntryWorkflow completion and
  holdings visibility.** The USE_EXISTING mutex protects only while entry #1 RUNS. After it
  completes with a fill, a second wake for the same symbol within the next ~1–2 bar-widths may
  still classify as entry_candidate: the EntryWorkflow's own held-check folds the ledger with
  `cutoff = as_of` (one completed bar in the past — load_live_bars → load_open_positions), so a
  fill booked seconds ago is *excluded* from the snapshot. The drain-side `_held_symbols` guard
  is the remaining defense — verify its fold has NO as_of cutoff (flagged → S5). If it reads
  current ledger state the hole is only drain-latency wide; if it also lags, this is a real
  double-buy at cutover. (The LiveStrategyWorkflow cascade has `recent_entries` for exactly this;
  EntryWorkflows have no equivalent cross-spawn guard.)
- **F-S3-2 (low, confirmed): `strategy_entry_decisions` dedupe breaks for NULL symbol /
  source_event_id.** Migration 023 UNIQUE(workflow_id, source_event_id, symbol) — Postgres
  treats NULLs as distinct, so `ON CONFLICT` (entry_decisions.py:40) never matches rows where
  either is NULL → repeated captures insert duplicates instead of upserting. The no_bars capture
  path passes `source_event_id or ""` (empty string, fine) but `symbol or None` (entry_decisions.py:49)
  → NULL for symbol-less wakes. Scoreboard counts inflate. Fix: default to '' not NULL, or
  UNIQUE ... NULLS NOT DISTINCT.
- **F-S3-3 (low, confirmed): `running_entry_workflow_ids` scans ALL running workflows** in the
  Temporal namespace and filters client-side (entry_spawn.py:168). Correctness fine; at scale
  (many instances × PositionWorkflows) each drain cycle pays a full visibility scan per
  configured instance. A `WorkflowId STARTS_WITH` / prefix query (or advanced visibility search
  attribute) would bound it.
- `_load_spec` lru_cache by bundle path: a spec edit + redeploy of the drain picks it up only on
  process restart — consistent with images being baked; nit only.

## 2026-07-06 — S4+S5: wake_events.py + drain_loop.py + live_dispatch.py + drain_gates.py +
drain_shadow.py + migrations 014/022/027 (depth: full)

This is a well-built transactional outbox: in-transaction emit with sticky terminal verdicts on
re-emit (wake_events.py:232–250), FOR UPDATE SKIP LOCKED claim, explicit discard-with-reason (no
silent drops), a reaper for orphaned `processing` rows, source-fair claiming behind an index
guard (falls back to FIFO if the index is missing rather than seq-scanning), and coalesced
market-bar signals. Route building is source-scoped to stop cross-strategy firehose flooding
(incident-driven). Delivery is explicitly at-least-once with workflow-side dedupe.

Cross-flags resolved:
- **F-S2-6 closed (benign):** yes, a wake can be redelivered across a continue-as-new boundary
  (signal sent → ack fails → reaper requeues at 300 s → `_seen` was reset by CAN). But the money
  layer is idempotent against it: EntryWorkflow client_order_ids derive from durable wake
  identity, and the cascade's `recent_entries` guard is carried across CAN. Double-*eval* is
  possible (wasted LLM spend), double-*trade* is not. Downgraded to accepted-behavior note.
- **F-S3-1 downgraded to low:** drain-side `_held_symbols` reads the *current-state*
  `virtual_portfolio_strategy_positions` view with no as_of cutoff (drain_shadow.py:68–75), and
  the fill row commits before EntryWorkflow #1 completes (route awaits fill+copy) while the
  USE_EXISTING mutex still holds. The double-entry window is effectively zero; the layered
  guards compose correctly.

New findings:
- **F-S5-1 (medium, confirmed): one failed signal crashes the drain daemon and strands the whole
  claimed batch for up to the reaper horizon (300 s).** drain_and_signal_wakes signals each
  target bare (live_dispatch.py:110–123); a WorkflowNotFound (e.g. a PositionWorkflow that
  completed between route resolution and signal — routes are only re-resolved per pass) raises
  through drain_once → run_drain_loop has no try/except around it (drain_loop.py:201) → the
  daemon exits; every event claimed this pass stays `processing` until the reaper requeues it.
  Failure scenario: position closes at the exact moment held-name news arrives → drain crash →
  all wake delivery pauses for restart + up to 5 min for the stranded batch. Roadmap Thread A
  says "watch task-not-found" — this is that class; fix shape: per-target try/except (skip dead
  targets, still ack), and/or ack-what-was-signaled on partial failure.
- **F-S5-2 (low, confirmed): per-event ack opens a fresh DB connection per wake**
  (mark_wake_event_processed → connect() each; live_dispatch.py:133–135). A 100-wake batch does
  100 connect/commit cycles per pass, every 2 s. Same connection-discipline issue as F-T2-2.
- **F-S5-3 (low, confirmed): `strategy_wake_decisions` has no retention/GC pairing in its
  migration** (022) — one row per (wake × instance) per drain pass in shadow mode; the exact
  growth pattern that bloated strategy_wake_events. Verify data_gc covers it in S9 (flag stays).
- Migrations 014/027: partial indexes for pending/pending-by-source are right; unique
  (source, source_event_id) backs the sticky-verdict upsert. Good schema.

## 2026-07-06 — S9: data_gc.py + health_monitor.py + migration 026 + scripts/data_lifecycle_check.py
(depth: full for data_gc/lifecycle-check; skim for health_monitor — file-based status/alert loop,
no money path)

- data_gc itself is well-shaped: preserve-everything default, explicit opt-in per-source
  retention, dry-run default, batched FOR UPDATE SKIP LOCKED deletes — fully consistent with the
  no-silent-truncation charter.
- **F-S9-1 (high, confirmed — verified against the live DB): `bar_window_snapshots` has no
  lifecycle story at all.** One JSONB row per live eval (the full aligned bar window for every
  active symbol), written by load_live_bars (live_bars.py:443); grep shows NO delete path in the
  repo; migration 018 has no retention; it is absent from both EXPECTED_TIMESCALE_RETENTION_POLICIES
  and the lifecycle-check waiver dict. Live DB today: 68 MB / 2,696 rows in ~18 days — already
  the 7th-largest table, and growth scales with evals × symbols × strategies (dual-run +
  per-position workflows each call load_live_bars per cadence tick). This is precisely the
  bloat→starve class Thread C was built for (schwab_l1 12 GB, wakes 8.7 GB), one table over from
  the ones that got fixed. Fix: retention policy (snapshots are worthless after their eval; even
  '2 days' is generous) + a lifecycle-check entry.
- **F-T2-1 confirmed and broadened (medium): the lifecycle guard only sees hypertables plus a
  2-entry waiver dict.** scripts/data_lifecycle_check.py is migration-static and pairs
  create_hypertable↔add_retention_policy; REGULAR_APPEND_ONLY_LIFECYCLE_NOTES waives only
  strategy_wake_events + market_time_series_points. Unwaivered, no-GC append-only tables:
  alpaca_order_events (wait_poll spam), alpaca_fill_events, alpaca_trade_escalations,
  strategy_wake_decisions (5 MB already, one row per wake×instance per drain pass),
  strategy_entry_decisions, llm_call_costs, strategy_theses, bar_window_snapshots,
  temporal_payload_store (250 MB — verify its GC in S11). The AGENTS.md rule is stated as
  binding but the guard can't see most of the tables it governs. Fix: make the checker demand a
  waiver-or-policy line for every CREATE TABLE in migrations, not just hypertables.
- Migration 026 correctly codifies the two manual retention policies. EXPECTED list matches ✓.

## 2026-07-06 — S10: reconcile_late_fills.py + reconcile_stream_subscriptions.py (depth: full
for late-fills; skim for stream-subscriptions — reconciles consumer sub rows, no money path)

Cross-flag resolved: the late-fill reconciler covers ONLY orders whose client_order_id starts
with `live-strategy/{instance}` (reconcile_late_fills.py:76) — i.e. workflow-routed orders. The
F-T1-4/F-T1-7 gap classes (managed `alpha-managed-*` CLI orders and `vp-*` router-default ids
whose VP copy failed) are NOT reconciled by anything automatic; they surface only as unresolved
escalations. Acceptable if documented, but the "and_confirm" flows' safety net is thinner than
the routed path's.

- **F-S10-1 (medium, confirmed): reconciliation entries bypass the ledger's structural dedupe.**
  reconcile_late_fills.py:110–129 calls append_ledger_entry with source_type='reconciliation'
  and NO source_id / source_fill_event_id → no advisory lock, no existing-entry check, no unique
  index protection (both partial indexes require the columns non-null). Dedupe is only the
  in-process `client_order_id ∉ booked` set read before writing. Failure scenario: the trade
  stream (or a manual copy-order-fill) books the same order via the alpaca_fill path while the
  reconciler daemon's cycle is between its read and its write → two ledger rows for one fill →
  position/cash double-count that no constraint stops (and account reconciliation would flag
  hours later). Fix: give reconciliation entries the SAME source identity the copy path uses
  (source_id=f"alpaca_order:{account}:{order_id}", source_type='alpaca_fill' or a shared type)
  so the per-book unique index applies across paths.
- **F-S10-2 (low, confirmed): the reconciler does money math in float** (fill_delta,
  reconcile_late_fills.py:57–63; round(cash_delta, 6)) where every other ledger writer uses
  Decimal. Sub-cent drift only, but it's the book of record.
- Good: scope-guard by client_order_id prefix (shared-account safety), dry-run default posture,
  provenance-rich entries, daemon mode with clean SIGTERM handling.

## 2026-07-06 — S6: durable_execution.py + migration 013 (depth: full)

- The primitive itself is clean: insert-or-bump-attempts, canonical-JSON request-mismatch guard,
  reconcile-before-resubmit, and a fail-closed `DurableStepNeedsReconciliation` when an active
  step can't be reconciled (conservative and right for money).
- **F-S6-1 (medium, confirmed — design): this whole mechanism is dormant.** `durable_order_step`
  and `PostgresDurableStepStore` have no callers outside their own tests; the live path uses the
  Temporal-native `submit_or_reconcile_order` activity + router pre-submit reconcile instead
  (and `DurableStrategyOrderWorkflow` is registered on the worker but nothing starts it — no
  start_workflow call for it outside tests). So the repo carries TWO durable-order-idempotency
  implementations, one live and one parked, plus migration 013's `strategy_durable_steps` table
  feeding neither. Either delete the parked one (the Temporal path has strictly more capability)
  or document why it's kept (a non-Temporal backend seed?). Cross-posted to ARCHITECTURE_REVIEW
  §2/§5 — this is exactly the "backend-neutral seam" question: if it IS the seed of a
  Temporal-free execution path, it should say so and be tested against the same contract.

## 2026-07-06 — S11: temporal_payload_store.py + migration 019 + run_store.py (depth: full)

- The claim-check driver is genuinely good: content-addressed dedupe, integrity re-hash on
  retrieve, loud failure on missing rows, hard cap that refuses rather than truncates, one
  connect helper so all clients share the codec. No-silent-truncation honored.
- **F-S11-1 (medium-high, confirmed — live-verified): neither offload store has a GC story.**
  `temporal_payload_store` is 250 MB with no delete path (rows must outlive Temporal history
  retention, so GC must key on age ≥ namespace retention — currently absent);
  `data/strategy_runs/` is **2.4 GB / 2,166 JSON artifacts** with no pruning and no
  index↔file consistency check. Both belong in the F-T2-1/F-S9-1 lifecycle sweep — the
  filesystem one is currently the second-largest data mass in the deployment.
- **F-S11-2 (low, confirmed): run_id containing "/" writes outside flat artifact-root listing**
  — run_store.py:59 tolerates separators by mkdir-ing parents; ids are sanitized at the caller
  (temporal_execution.py:1987 replaces "/"), so only non-live callers can produce nested paths;
  path traversal via `..` in a run_id is theoretically open (run ids are internal, not
  user-supplied — nit).

## 2026-07-06 — S7: live_bars.py + market_data.py + stream_data.py + market_bar_consumers.py +
migrations 017/018 (depth: full for live_bars/stream_data; skim market_data/market_bar_consumers)

Point-in-time discipline checks out where it matters: `as_of` = completed-bar cutoff computed
workflow-side; both read paths bound `bucket_start_utc <= as_of` / `ts_utc <= as_of`, which
(bucket-start semantics + one-bucket step-back) correctly includes the last *completed* bar and
excludes the forming one. WS-derived bars override cached REST history on overlapping buckets
(authoritative receive-time data wins). Warmup on the live path pins RetrievalSource.SCHWAB
explicitly against the yfinance-delayed-data trap. Dropped symbols/timestamps are always
reported in provenance, never silent.

- **F-S7-1 (low-medium, confirmed): multi-symbol alignment intersects timestamps — one sparse
  symbol shrinks everyone's window.** load_bars_window aligns >1-symbol windows to the exact
  timestamp intersection (live_bars.py:407–421), and on empty intersection silently keeps only
  the deepest symbol. The bar-builder's synthetic empty bars keep *subscribed* symbols dense, so
  the live region is fine; but REST-history backfill has only traded minutes, so adding one
  illiquid holding to the book prunes the shared warmup window for every symbol in the eval
  (weakening SMA/RSI-gated logic with only a provenance breadcrumb). Design pressure: the
  equal-length matrix contract (runner) is forcing a worst-common-denominator grid — consider
  per-symbol series with forward-fill at the engine boundary instead.
- **F-S7-2 (low, confirmed): two warmup implementations with different source policies.** The
  activity path (ensure_warm_window, live_bars.py:509) pins SCHWAB explicitly; the module-level
  `load_live_bars`/`_load_warmup_bars` path (live_bars.py:609–618) uses RetrievalSource.AUTO —
  the exact yfinance-delayed fallback the newer path's docstring warns about. That older path
  appears unused by the live runtime (only CLI/paper surfaces); either delete it or align its
  source policy before someone wires it back in.
- **F-S7-3 (nit): `strategy_required_lookback` scans param NAMES for window-like hints**
  (live_bars.py:483–506) — a custom node with `lookback_days: 365` would demand a 370-bar warmup
  threshold; harmless (threshold, not fetch size), but it's convention-by-string-matching.
- migration 017 (market_derived_bars): hypertable + retention ✓ (in 026). 018: see F-S9-1.

## 2026-07-06 — S8: live_worker.py + temporal_worker.py + temporal_launch.py (live_dispatch.py
covered under S4/S5) (depth: full)

- temporal_launch is where several good invariants live: launch_id minted per launch
  (client_order_id/run-id collision design), idempotent relaunch via one-workflow-id-per-instance,
  event-source stamping for source-scoped wake routing, the entry/position sub-DAG sink contracts
  documented with WHY-comments (incl. the "manage_exits_vdd needs a non-empty target series"
  contract). build_live_command enforces portfolio↔instance pairing at launch, matching the
  workflow-side refusal. No new findings.
- temporal_worker: all clients connect through the claim-check helper ✓; worker registers the
  full activity set; agentic provider injected at worker start (uniform provider seam). Clean.
- **F-S8-1 (low, confirmed — design): `LiveStrategyWorker` (live_worker.py) is a deprecated
  in-process live path kept for "tests and diagnostic comparisons"** — its default provider is
  the legacy raw-L1 tick-as-bar path the runbooks warn against. Same family as F-S6-1/F-S7-2:
  retired mechanisms retained without a quarantine marker in code structure (only a docstring),
  inflating the runtime surface a reviewer/agent must reason about.

## 2026-07-06 — S12: live_preflight.py + live_status.py + portfolio_status.py +
portfolio_context.py (depth: full for preflight structure + portfolio_context; skim for the two
read-only status modules)

- live_preflight codifies an unusually complete operational checklist: DB/Temporal/payload-store
  connectivity, stale-image detection, daemon-running + daemon-gate checks, portfolio binding
  (reusing check_virtual_trade_binding — same failure the router would raise, surfaced early),
  Alpaca account, derived-bars freshness, wake flow, **table sizes**, L1 tick flow, bar-builder
  state, **task-not-found rate**, retention policies, Schwab token age. This is the right way to
  encode incident lessons. No correctness findings.
- Cross-flag resolved (F-V1-1 context): preflight checks binding existence/activeness only —
  nothing anywhere enforces or verifies instance-scoped sell semantics. F-V1-1 stands.
- portfolio_context.py: pure, well-contracted (dict/scalar handles only, #5 cutoff respected,
  equity = cash + Σ marked holdings with None-price holdings valued at 0 — conservative and
  documented). No findings.

## 2026-07-06 — S13: portfolio_displacer.py + displacement_shadow.py (parked Thread J) (depth: full)

- Correctly quarantined: no live side effects, no order path, deterministic gate + bottom-K
  pre-rank pure and unit-tested; shadow runner reads the live book read-only. The evidence-
  symmetry property the roadmap calls out is structurally present (same evidence assembly for
  incumbent and candidate).
- **F-S13-1 (nit, design): hardcodes `DeepSeekAgenticProvider`** rather than going through the
  uniform `make_agentic_provider` factory — deliberate per the sandbox findings (pro+flash
  split), but it's the first module to bypass the uniform provider seam; when Phase 5 wires it
  into the EntryWorkflow, route the model choice through node params like every other agentic
  node so it stays operator-tunable.

## 2026-07-06 — S14: thesis.py + engine/thesis_nodes.py + migration 021 (depth: full)

- open_thesis is idempotent via partial unique index on open rows (ON CONFLICT ... WHERE
  status='open' DO NOTHING); close/append are guarded UPDATEs; keying by (strategy_id,
  portfolio_id, symbol) matches what the EntryWorkflow/PositionWorkflow and full DAG both use ✓.
- No new findings. (The F-S2-1 rejected-sell path closing a thesis that shouldn't close is a
  workflow bug, not a thesis-store bug.)

## 2026-07-06 — E1: spec.py + validation.py (depth: full)

- spec.py itself is exemplary: frozen dataclasses, enum-validated literals, `extra` round-trip
  for forward-compat, clean YAML I/O. The ontology carries a rich per-node policy surface
  (artifact/cost/retry/state/safety policies + timeout).
- **F-E1-1 (medium, confirmed — design+correctness): the flagship safety checks are
  self-attestation, not verification.** `check_no_lookahead` fires only if a node *declares*
  `lookahead: true` (validation.py:60–74); `check_completed_bar_only` only if it declares
  `uses_incomplete_bar` (:76–89); `check_point_in_time_data` only on declared `point_in_time:
  false`. A node that actually reads future bars but says nothing passes all three. The real
  point-in-time protection lives in the data layer (as_of bounds) and runner semantics — which
  is fine — but the check *names* promise behavioral verification the checks don't do, and both
  charters lean on "passes the compiler/linter" as the safety neck. Failure scenario: an agent
  authors a custom feature node whose params include a future-window offset under an unlisted
  name → validate-spec passes `no_lookahead` → backtest silently optimistic. Fix shape: either
  rename these checks to `*_declared`, or add behavioral probes (e.g. runner-level assertion
  that node outputs at bar i are invariant to truncating bars > i — a property test the runner
  could run on fixtures).
- **F-E1-2 (low, confirmed): per-node `timeout_seconds`/`retry_policy` are lint-required for
  agentic nodes but ignored at runtime** — compiler demands them (compiler.py:392–395) yet
  temporal_execution runs every node activity with the global
  `node_activity_timeout_seconds=300`/`maximum_attempts=3`. Declared-but-unenforced policy is
  ontology theater; either wire node.timeout_seconds into the per-activity timeout or stop
  requiring it. (Cost bounds: → E6 checks whether the harness enforces cost_policy.)

## 2026-07-06 — E2: compiler.py + test (depth: full)

- Solid: Kahn topo-order with deterministic tie-breaks, cycle → error with the cyclic set named,
  template-contract lint (params/handles vs registry), mode/replayability compatibility,
  evidence-requirement lint, `risk-before-live-action` ordering check, agentic policy lint,
  edges validated against handles.
- **Cross-flag resolved → F-S1-1 (medium, confirmed-latent):** nothing enforces YAML declaration
  order, and the compiler's `topological_order` is computed but NOT used by
  `_evaluate_strategy`'s I/O-node loop (declaration order, temporal_execution.py:2056). Today's
  specs/fixtures have no I/O→I/O edges (the multi-model pipeline lives INSIDE one node), so it's
  latent — but the first spec wiring data→agentic or agentic→agentic out of declaration order
  fails at runtime with a missing-input error that validate-spec said was fine. One-line fix:
  iterate io_node_ids in compile_result.topological_order.
- `_lint_risk_before_live_action` uses topo position min(risk) < action — right idea; note it
  passes if ANY risk node precedes the action, even if it's not on the action's path (weak but
  warn-level acceptable).

## 2026-07-06 — CORRECTION to F-E1-2 (from E6 read)

The agentic harness (engine/agentic.py) **does** enforce the declared bounds at execution time:
`_ProviderBudget` raises on max_calls / max_tool_calls / max_tokens / max_usd overrun (:203–230),
the node's own timeout_seconds is enforced in-harness via monotonic check (:109), and
`_validate_bounds` refuses to run without a positive bound. F-E1-2 is narrowed to: (a)
node `retry_policy` is a spec field nothing reads; (b) the Temporal activity timeout is a global
300 s that doesn't scale with a node's declared timeout (a >300 s node would be killed by the
backstop); (c) token/cost budgets depend on provider adapters filling `usage` — an adapter that
returns empty usage silently disables all but max_calls. Downgrade F-E1-2 to low. F-E1-1
(self-attestation lookahead checks) stands unchanged.

## 2026-07-06 — E3: engine/registry.py + engine/example_test.py (depth: full for registry
census + template contracts; skim for individual template bodies)

- 79 NodeTemplates. Census by kind: feature 29, evaluator 18, signal 11, condition 9, risk 9,
  trigger 2, agentic 1 — and **zero** templates for `data`, `action`, `notify`, `report`, `gate`.
- **F-E3-1 (design, high-importance — cross-posted §1): five of the twelve node kinds have no
  implementation, and two of them are load-bearing in the docs' story.** ARCHITECTURE's Known
  Gaps admits only `gate`. In reality: there is no `data` node (bars/evidence are seeded into
  the DAG by the runtime, not declared as nodes), and **no `action` node — orders never appear
  in the DAG at all**. The trade decision leaves the ontology at `desired_weights_raw` and is
  turned into orders by `_orders_from_decision` + the router, entirely outside the spec. So the
  "typed DAG contains the trading logic" claim is ~60% true: entry/exit *judgment* is in the
  DAG; *acting* is runtime folklore. This matters for extensibility (an options/crypto action
  would have nowhere to live) and for the "understandable without Temporal" claim (the
  sizing/reconciliation semantics live in temporal_execution.py). Fix direction: either
  implement `action` (an order-intent node whose params carry sizing policy, executed by the
  backend), or shrink the ontology to the seven kinds that exist and move sizing policy into a
  first-class spec section instead of Python.
- Template contracts themselves (params schema, handles, mode/replayability declarations) are
  consistently structured; engine/example_test.py exercises them broadly.

## 2026-07-06 — E4+E5: engine/exits.py, allocation, selection, factors, primitives, triggers,
branching, divergence, liquidity_screen, news_triage (depth: full for exits/triggers/primitives;
skim for allocation/selection/factors/branching/divergence/liquidity_screen/news_triage)

- **Cross-flag resolved → F-S2-5 retracted:** manage_exits_vdd merges the durable peak with the
  current-window peak via max() (exits.py:203–209), so the PositionWorkflow passing the
  *previous* durable peak is correct — no stale-trail lag. Good design, clearly commented.
- exits.py guard structure (stop/take-profit/trail/VDD, first-fires) is clean; `_clean_tail`
  handles None-tails; entry_time pins the peak window (pre-entry spikes excluded — matches the
  position_workflow test).
- triggers.event_trigger: strict point-in-time alignment of wakes to completed bars in backtest;
  live staleness horizon relative to the latest completed bar (not wall clock) — consistent with
  the drain gate's wall-clock note (deliberate asymmetry, documented both sides). Good.
- allocation (equal_weight/inverse_vol/risk_parity): float math on weights (fine — weights are
  not money), degenerate-input guards present. No findings at skim depth.

## 2026-07-06 — E6+E7: agentic.py + agentic_structuring + native/deepseek/pipeline/langgraph
providers + make_agentic_provider factory (depth: full for agentic.py/deepseek/factory core;
skim for native/pipeline/langgraph internals)

- The bounded-harness contract is real (see correction above), artifacts record
  evidence/config/usage, and backtest evidence must carry received_at_utc (:255–261) — the
  grounding invariant is enforced at the right layer.
- **F-E7-1 (low, confirmed): budget enforcement quality depends on adapter usage-reporting.**
  `_usage_number` treats missing usage as 0 (agentic.py:232–237); a provider adapter that omits
  token/cost fields makes max_tokens/max_usd bounds silently vacuous. max_calls always works.
  Worth a lint or runtime warning when a bound is declared but the adapter reports no usage.
- Uniform provider factory (make_agentic_provider) + per-node override resolution keeps the
  four providers behind one interface; deepseek provider splits search (pro) from structuring
  (flash) with merged usage. Keys come from settings/env, never logged — spot-checked clean.

## 2026-07-06 — E9: runner.py + runner tests (depth: full for simulation/decision path; skim
for metrics/report assembly)

- **F-E9-1 (medium, confirmed): backtest fills happen at the signal bar's close while being
  labeled "next_bar_after_signal".** `_simulate_equity` applies weights[index-1] to the
  close[index-1]→close[index] return (runner.py:945–984, _symbol_bar_return:1005–1015) — i.e.
  the strategy earns the full bar-over-bar return from the moment of the signal close, which is
  only achievable by filling AT the observed close. Trade rows and decisions are stamped
  "execution_timing: next_bar_after_signal" (:970, :1286), which implies open[next]-ish fills.
  On 1-minute bars the gap is negligible; for DAILY-frequency families the overnight gap is
  systematically credited to the strategy — classic close-vs-open optimism, partially offset by
  slippage_bps. Failure scenario: an earnings-drift daily strategy looks profitable in backtest
  purely on gap capture it could never trade. Fix: execute at next bar's open (needs open in the
  return math) or relabel to "fill_at_signal_close" so the report doesn't overstate rigor.
- Otherwise the simulation is honest: turnover-proportional costs from cost_policy bps, explicit
  zero-preserving desired_weights_raw channel for the router, per-symbol return contributions
  recorded. Determinism: pure over bars+spec (no wall-clock reads in the sim path).

## 2026-07-06 — E8/E10/E11/E12: company_prefill + evidence_features + evidence_resolution;
library/registry.py + indicators; skeletons/bundles/skeleton_coverage; cli/catalog/subdag/links/
robustness (depth: skim, with subdag/robustness/evidence_resolution read full — small files)

- library/registry.py mirrors engine registry discipline for indicator capabilities; indicator
  impls are thin pandas-ta wrappers with colocated example tests. No findings at skim depth.
- bundles/skeletons: declarative YAML + loader validation; skeleton_coverage checks family
  coverage. No findings at skim depth.
- evidence_resolution: resolver seam defaulting to empty evidence with the real retrieval
  resolver injected on the worker — consistent with the provider seam pattern. OK.
- Not deeply reviewed (recorded honestly): evidence_features.py internals, per-indicator math,
  CLI arg wiring. Nothing in these paths touches money directly.

## 2026-07-06 — D1: schwab_stream/ (depth: full for bar_builder core + lease; skim for daemon/
events/tools/status/token_status)

- bar_builder is careful work: receive-time bucketing with the policy named in metadata
  (`bucket_policy: received_at_utc_floor`), Decimal prices, synthetic-empty bars carrying the
  previous close (keeps subscribed symbols on a dense grid — this is what makes F-S7-1 mostly
  benign in the live region), explicit `is_partial` flags for volume-baseline gaps and large
  in-bucket gaps, signature-compared idempotent upserts, only-closed-buckets building, and
  wake emission gated by freshness + active-consumer checks (the firehose-gate incident fix).
- **Lease flag (F-T3-1) noted per daemon:** schwab daemon AND bar_builder both use the broken
  `schwab_stream/lease.py` upsert. Downstream blast radius of a double-builder is contained by
  the idempotent upsert + sticky wake dedupe; the dangerous exposure is the *websocket* daemon
  (connection fight against Schwab's one-connection rule).
- No new findings beyond the lease.

## 2026-07-06 — D2+D3: news_stream/, sec_stream/, market_events_stream/,
earnings_calendar_stream/ (depth: full for news events/adapters emit path; skim for the rest —
the five daemons share one scaffold shape)

- The P5-1 atomicity promise is real: source-row upsert and `emit_strategy_wake_event` share
  the same connection/transaction in every stream (news events.py:264–269, sec events.py:160,
  earnings events.py:148), with per-source dedupe keys (source_message_id / (source, filing_id) /
  (symbol, fiscal_year, fiscal_quarter)) and sticky terminal wake verdicts on re-emit.
- Lease variants (F-T3-1): news + sec = broken pattern; market_events + earnings = correct
  pattern. Flag resolved (recorded per daemon).
- **F-D2-1 (design, cross-posted §4): five stream daemons re-implement the same chassis** —
  tick loop, lease, status upsert, signal handling, once-mode — with small drifts (the lease
  bug being the costly one). One shared daemon base + per-source adapters would collapse
  ~1,000 lines and make invariants single-point.

## 2026-07-06 — D4: __init__.py (env autoload) + db.py + settings.py + market_cache.py +
market_bars.py + market_halts.py (depth: full for init/db/settings; skim for market_*)

- `.env` autoload is carefully done: setdefault semantics, sandbox-safe (no pathlib.resolve),
  silent no-op when absent; matches AGENTS.md's documented contract exactly.
- **F-D4-1 (low-medium, confirmed — the F-T2-2/F-S5-2 root): `db.connect()` opens a fresh
  psycopg connection per call, no pooling anywhere.** Every audit row, wake ack, status upsert,
  ledger read = one TCP+auth round trip. Fine at today's volume; Thread A's parallel
  EntryWorkflows and the 2 s drain cadence multiply it. psycopg_pool is a drop-in shape.
- **F-D4-2 (low, confirmed): `load_settings()` re-reads ~40 env vars on every call** (no
  lru_cache) and `connect()` calls it each time — harmless, but it means env mutation
  mid-process changes DB targets silently; cache it for predictability.
- **F-D4-3 (low, confirmed): `apply_migrations` has no concurrency guard** (db.py:24–42) — two
  containers migrating simultaneously race the CREATE/INSERT sequence (no advisory lock,
  single-commit-at-end). Compose startup ordering makes it unlikely today.

## 2026-07-06 — D5: entity.py + llm_costs.py + source_utils + sec_suspensions + evidence/tools +
hacker_news + fda + migration 020 (depth: full for llm_costs structure + entity resolver entry
points; skim for the rest)

- llm_costs: per-call rows with provider/model pricing lookup, fail-soft recording, durable
  independent of run-store policy (the "invisible spend" fix codified in migration 020's
  comment). No truncation of raw usage. No findings.
- entity.resolve_tradeable_symbols / is_tradeable: registry-backed set-membership, no network —
  the cheap gate drain_gates relies on. Consistent with its use.

## 2026-07-06 — R1–R4: retrieval/, research/, capabilities/ (depth: full for retrieval/services
fallback+contract paths; skim for registry/tools/volume_delta, research/, capabilities/)

- retrieval/services honors the standard response contract (tool/source/received_at_utc/request/
  attempts/data/raw_payload) uniformly; `received_at_utc = now()` is correct observation-time
  semantics for live retrieval. The consequence — event-driven families can't backtest because
  retrieval re-stamps receipt time — is already tracked (roadmap "As-of retrieval", not
  re-reported). AUTO fallback (Schwab → yfinance) carries the delayed-data hazard, which the
  live path avoids by pinning SCHWAB (E7/S7); the remaining exposure is agent/CLI usage, where
  provenance in `attempts` makes the source visible. No new findings.
- retrieval/registry.py: declarative tool metadata (side_effects labels, examples) — the
  machine-registry layer works as advertised; buy/close managed flows are correctly labeled
  `broker_paper`. Skim only.
- research/: sandboxes + model-comparison harnesses, no money path, honestly quarantined. Skim.
- capabilities/: derived read-only catalog with its own tests. Skim.

## 2026-07-06 — C1–C6: charters + core docs as review targets (depth: full for AGENTS.md,
ASSISTANT.md, ARCHITECTURE.md, RESOURCE-MODEL.md, STRATEGY-LIFECYCLE.md, ROADMAP.md,
grammar doc §structure; skim for strategy-runtime-architecture.md)

Drift verdicts (docs vs the code read in this review):
- **RESOURCE-MODEL.md / STRATEGY-LIFECYCLE.md: accurate.** Every claim I could check against
  code held (recipe-vs-bundle split, object chain, lifecycle steps, live-runtime behavior list
  matches LiveStrategyWorkflow exactly). Minor staleness only: neither mentions the newer
  decision-capture tables (strategy_entry_decisions, strategy_wake_decisions) or theses in the
  Postgres ownership row. These are the best docs in the repo.
- **ARCHITECTURE.md: mostly accurate, two understatements.** (1) Known Gaps lists `gate` as the
  dangling node kind — in fact data/action/notify/report are equally dead (F-E3-1); "the DAG
  contains the trading logic" needs the qualifier that sizing/order semantics live in the
  runtime. (2) "Temporal is one live execution backend, not the ontology" is true for the spec,
  not for execution semantics (ARCHITECTURE_REVIEW §2). The status/gaps discipline itself
  (dated, verify-against-code framing) is excellent.
- **F-C6-1 (medium, confirmed — doc drift): the grammar canon describes capabilities that don't
  exist.** docs/design/agentic-strategy-dag-grammar.md specifies gate nodes (§506) and an
  execution order that "applies gates/risk/action nodes" (§791); ASSISTANT.md points agents at
  this doc as "what a strategy *is*." An agent authoring a spec with a gate or action node gets
  a validate-time template failure (best case) or silent no-op semantics. The canon should mark
  unimplemented kinds with the repo's own 🔮 label — the labeling convention exists (AGENTS.md)
  and isn't applied where it matters most.
- **AGENTS.md/ASSISTANT.md as designs:** the two-charter split is genuinely good (builder vs
  assistant separation of concerns, safety invariants stated once, pointers not copies). Two
  critiques: (1) ASSISTANT.md's "the neck is what makes the result runnable and
  point-in-time-safe" oversells the linter given F-E1-1 (self-attestation checks); (2) AGENTS.md
  states the lifecycle rule as binding while the guard that enforces it sees only hypertables
  (F-T2-1) — a binding rule without an enforcing check is how bar_window_snapshots happened.
- **ROADMAP.md:** current, maintained as promised, and its priorities largely match what this
  review found (Thread C = the biggest operational risk class; Thread H correctly pre-scale).
  Two priority notes: the lease bug (F-T3-1) belongs ABOVE most of Thread C's remaining items —
  it undermines the invariant the whole ingestion architecture is built on; and Thread A's
  cutover checklist should absorb F-S2-1 (PositionWorkflow rejected-sell exit) + F-S2-4
  (non-market entries) before `route:true`.

## 2026-07-06 — X2–X5: cross-cutting passes (depth: full)

**X2 clock audit: clean.** No naive `datetime.now()`/`utcnow()` outside tz-aware use anywhere in
alpha_claw/; workflow code uses `workflow.now()`; completed-bar vs receive-time vs vendor-time
separation is maintained at every boundary I traced (bar_builder metadata even names its bucket
policy). The one wrong-clock-family issue found all review: the backtest fill-timing label
(F-E9-1), which is a semantics/labeling bug, not a clock-source bug.

**X3 test suite: 575 passed, 1 failed, 5 deselected (live).**
- **F-X3-1 (medium-high, confirmed — test isolation): DB-backed tests run against the LIVE dev
  database and can consume real coordination state.**
  `test_drain_and_signal_routes_wakes_by_symbol_and_acks` calls the real
  `drain_and_signal_wakes(limit=100)` against the shared outbox: it claims the 100 FIFO-oldest
  *real* pending wakes and **acks them as processed without delivery** — in the passing case
  too, not just the flake. The comment above it (wake_emission_example_test.py:95) documents the
  *flake* ("re-run or pause the daemon") but not the destructive ack. Today the strategy-drain
  service is stopped and a real backlog exists (2,656 pending), so the test's own rows never
  made the claim batch → assertion failed, and each invocation acked 100 real wakes.
  **Disclosure: this review's two test invocations acked 200 real pending wakes (news/sec,
  received 20:07–20:12 UTC today) as `processed`.** I did NOT revert: re-pending them would
  inject 200 stale wakes into tomorrow's 09:25 ET dual-run load test (a worse interference);
  by drain restart they'd be staleness-gated anyway. Restore SQL if wanted:
  `UPDATE strategy_wake_events SET status='pending', claimed_at_utc=NULL, processed_at_utc=NULL
   WHERE status='processed' AND processed_at_utc BETWEEN '2026-07-06 20:52' AND '2026-07-06 21:00';`
  (verify the window against processed_at_utc first).
  Fix shape: drain tests must scope their claim to their own synthetic rows (e.g. a test-only
  source value + a source filter in fetch, or a dedicated schema/DB for tests). This is the same
  class as F-V2-1: test correctness/coverage is environment-dependent.
- The known-flake comment itself is candid — but a money-adjacent test that "flakes against the
  news firehose" is a symptom of the shared-DB design, not a fact of life.

**X4 live-only coverage map (consolidated from per-module notes):** exercised ONLY live (or not
at all in CI): Alpaca managed flows end-to-end (T1 — 32-line test), real provider agentic calls
(by design, provider seam), Schwab websocket behavior, lease contention (F-T3-1 had no test in
either variant), wait_for_terminal_state polling ladder, market_data_alignment math (pure,
testable, untested). Well-covered without live: VP ledger/router, workflows via Temporal test
env, drain/wake logic (when DB present), engine nodes, bar_builder.

**X5 secrets sweep: clean.** Keys flow settings→SDK constructors only; account metadata exposes
booleans (`api_key_configured`); preflight strips credentials from the DB URL (`_safe_host`);
`.env` loader never logs values; no print/log of key material found. The secrets architecture
(.env canonical + setdefault autoload + shape-only inspection) is sound in practice.
- **F-X5-1 (nit): `TradingResult.request` dicts embed `account_id`/aliases but never keys** —
  verified across trading/VP paths. No action needed; recorded as checked.
