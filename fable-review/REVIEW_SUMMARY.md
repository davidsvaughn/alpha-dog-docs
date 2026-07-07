# Alpha-Claw comprehensive review — summary (run 2026-07-06)

Reviewer: Claude (Fable 5). Read-only review per `docs/meta/review/fable-review-prompt.md`.
Full trail: `REVIEW_PLAN.md` (54/54 items, per-item depth recorded), `REVIEW_NOTES.md`
(~60 findings, F-IDs cited below), `ARCHITECTURE_REVIEW.md` (design review + verdicts).

**Remediation plan** (added 2026-07-06, after the review): all major findings mapped to nine
sequenced workstreams with concrete fix designs → [`REMEDIATION_PLAN.md`](REMEDIATION_PLAN.md).
Plain-language companion (what's going on + the connecting themes): [`EXPLAINER.md`](EXPLAINER.md).

**⚠️ One side effect to know about first:** running the (explicitly allowed) `-m "not live"`
suite twice acked **200 real pending wake events** as `processed` without delivery — the drain
test drains the *shared live outbox* (F-X3-1). I deliberately did not revert (re-pending them
would inject 200 stale wakes into Monday's 09:25 ET load test); restore SQL is in the X3 notes
if you want them back. This is itself a top finding: DB-backed tests mutate live coordination
state.

---

## Top issues by severity

### Critical
1. **The stream lease does not mutually exclude — the one "firm invariant" is unenforced.**
   (F-T3-1, confirmed by SQL reproduction.) In `trading/stream_lease.py:8`,
   `schwab_stream/lease.py:9`, `news_stream/lease.py:6`, `sec_stream/lease.py:6`, the upsert
   sets `owner_id = EXCLUDED.owner_id` unconditionally and `RETURNING owner_id = %s` — any
   contender steals an unexpired lease and *both* daemons are told they acquired it (verified:
   A/B flip-flop forever, both streaming). A one-off `docker compose run schwab-stream` beside
   the service replica double-connects the Schwab account. The two newest daemons
   (market_events, earnings_calendar) carry the **correct** `DO UPDATE … WHERE owner-or-expired`
   form — the fix exists in-repo and was never back-ported (F-T3-2). Fix: one shared lease
   module with the guarded form + a contention test.

### High
2. **PositionWorkflow treats any terminal sell as a successful exit** — a rejected/expired
   zero-fill sell closes the thesis and retires the only workflow watching the position
   (`temporal_execution.py:1694` checks `ok` but not `ledger_entry`; the EntryWorkflow does this
   correctly at :1503). Post-cutover this abandons a live position on one rejected order.
   (F-S2-1, confirmed.) **Belongs on Thread A's pre-`route:true` checklist.**
3. **`is_tradeable_on_alpaca` cache-poisons on any transient error, per process, forever**
   (`trading/services.py:131`, used on the live route path at `temporal_execution.py:919`). One
   Alpaca 500 while routing NVDA → NVDA silently untradeable until worker restart.
   `drain_gates.py:11` names this hazard; it's unfixed at the layer that gates orders.
   (F-T1-1, confirmed.)
4. **`close_and_confirm` sells the whole *account* position and books it all to one book**
   (`trading/services.py:1403,1491`). Correct only while account↔portfolio is 1:1 — which the
   pooled-book design (hash venue policy, multi-book tests) explicitly doesn't promise.
   (F-T1-2, confirmed.)
5. **`bar_window_snapshots` has no lifecycle story at all** — one full-window JSONB row per live
   eval, no delete path in the repo, absent from retention expectations and the lifecycle
   checker; 68 MB/2,696 rows already (live-verified). Same class as the 8.7 GB wake and 12 GB L1
   incidents Thread C just fought. Siblings: `temporal_payload_store` 250 MB no-GC,
   `data/strategy_runs/` **2.4 GB**/2,166 files no-GC. (F-S9-1, F-S11-1, confirmed.)

### Medium (selected — full list in notes)
- **Drain daemon dies on one failed signal** and strands the claimed batch ~5 min
  (`live_dispatch.py:110`, no try/except in `drain_loop.py:201`) — the "task-not-found" class
  Thread A is watching. (F-S5-1)
- **Late-fill reconciler bypasses the ledger's structural dedupe** (no source_id → no unique
  index, no advisory lock; in-process set check only) — a stream/manual copy racing the
  reconciler double-books a fill. Also float math in the book of record.
  (F-S10-1/2, `reconcile_late_fills.py:110`)
- **`record_strategy_eval` failure kills the live actor** — the one unguarded activity in
  LiveStrategyWorkflow (`temporal_execution.py:1983`). Observability shouldn't be load-bearing.
  (F-S2-2)
- **Sell guard is book-scoped, never instance-scoped** — instance B can sell instance A's
  shares; per-instance folds then silently diverge while book totals stay clean
  (`virtual_portfolios/services.py:430`; contradicts the scope promise at :1313). (F-V1-1)
- **The flagship validation checks are self-attestation** — `check_no_lookahead` only fires if
  a node *declares* `lookahead: true` (`validation.py:60`); both charters oversell the linter as
  the point-in-time safety neck. (F-E1-1)
- **Backtest fills at signal-bar close while labeled "next_bar_after_signal"**
  (`runner.py:945–984`) — negligible on 1m bars; systematically credits the overnight gap for
  daily-frequency families. (F-E9-1)
- **I/O nodes execute in YAML declaration order, not the compiler's topo order**
  (`temporal_execution.py:2056`) — latent until the first data→agentic or agentic→agentic spec;
  validate-spec would pass a spec that fails at runtime. One-line fix. (F-S1-1)
- **Managed-flow result contract is retry-hostile**: `virtual_fill_copy_failed` reads as a
  failed trade after a real fill; a retrying caller double-buys (client_order_id regenerates
  per call). (F-T1-3)
- **Tests share the live DB** — drain test acks real wakes (F-X3-1, see banner);
  DB tests skip silently when the DB is absent, so green ≠ tested (F-V2-1).
- Known-tracked items verified still open, not re-reported: Thread H cost-basis view divergence
  + sell-guard read-then-submit race (both confirmed unchanged at the schema/code level).

---

## Design review (distinct from the bug list — full text in ARCHITECTURE_REVIEW.md)

1. **The ontology is 7 kinds pretending to be 12.** Census: feature 29, evaluator 18, signal 11,
   condition 9, risk 9, trigger 2, agentic 1 — `data`/`action`/`notify`/`report`/`gate`: **zero
   templates**. The docs admit only `gate`. The consequential one is `action`: orders never
   appear in the DAG — sizing and order construction live in `temporal_execution.py`
   (`_orders_from_decision`, `_buy_qty`). "A strategy is an inspectable typed DAG" is true for
   judgment, false for action. Either implement `action` or shrink the declared ontology and
   make sizing a first-class spec section. (F-E3-1)
2. **Temporal separation: mechanically delivered, semantically not.** Workflow bodies are
   clean orchestration (genuinely good), but a second backend would have to reimplement the
   money semantics, not just scheduling. The dormant `durable_execution.py` (F-S6-1) is the
   ghost of the neutral seam. Honest posture: spec is backend-neutral; execution is
   Temporal-native by choice — then make the execution layer smaller, not more portable.
3. **The repo's accidental-complexity generator is retention + copy-paste, not
   over-abstraction**: 6 lease copies (1 broken = the critical), 5 daemon chassis, 2 durable
   mechanisms (1 dead), 2 warmup paths, 2 cost-basis computations (Thread H), twin managed
   flows, a deprecated live worker. Deletion and consolidation would fix more risk than new
   code.
4. **State placement is mostly principled** (git=authored, Postgres=money/coordination,
   files+index=artifacts) with two structural leaks: derived financial state computed in two
   languages (fold vs SQL views — Thread H is a symptom; recommend views stop computing basis
   at all), and escalations written per-call-site discretion rather than through one chokepoint
   (router failures never escalate — F-V1-4).
5. **Testing philosophy verdict:** tests-as-demonstrations does not inherently under-test —
   virtual_portfolios and the workflow tests are adversarial and excellent. The failure
   correlates with layer age: `trading/services.py` has 32 test lines for 2,335 LOC of broker
   flow. Needed: an isolation rule (never touch shared live state) and a floor rule (money
   modules get the VP standard).
6. **Keep/evolve/rework:** Keep — spec language, VP ledger, wake outbox/drain, Temporal runtime
   (decomposed), docs discipline. Evolve — trading managed flows, backtest runner, data
   lifecycle enforcement. Rework — stream-daemon chassis + leases; delete/quarantine the
   dormant paths. (Table with justifications in ARCHITECTURE_REVIEW.md.)

## Cross-cutting patterns

- **Clock discipline is a strength**: no naive datetimes anywhere; workflow.now(); completed-bar
  as_of computed deterministically; receive-time vs bar-time consistently separated (X2 clean).
- **Idempotency is strong wherever client_order_id is workflow-minted** (durable wake-identity
  tokens are excellent design); it's weak at the edges: CLI-triggered flows regenerate ids,
  reconciliation entries skip the dedupe key.
- **Secrets: clean** (X5) — shape-only inspection is followed in practice; preflight strips
  DB credentials.
- **"Binding rule without an enforcing check" recurs**: lifecycle rule vs hypertable-only
  checker; instance-scope promise vs book-scoped guard; grammar canon vs unimplemented kinds.
  The knowledge-promotion instinct (encode lessons in checkers) is right — the checkers just
  need to cover what the rules govern.
- **Failure→escalation is inconsistent**: managed flows escalate; the router, drain, and
  workflows log/record instead. One chokepoint would make "no silent money failure" structural.

## If you fix nothing else (ordered)

1. Lease SQL → guarded form everywhere + shared module + contention test (F-T3-1) — *before*
   the next ad-hoc daemon run, not after.
2. PositionWorkflow exit: require `ledger_entry` before declaring "exited" (F-S2-1) — gate on
   Thread A cutover.
3. Retention for `bar_window_snapshots` + `temporal_payload_store` + `data/strategy_runs`, and
   make the lifecycle checker demand a policy-or-waiver per CREATE TABLE (F-S9-1/S11-1/T2-1).
4. `is_tradeable_on_alpaca`: don't cache transport errors (F-T1-1).
5. Drain: per-target signal try/except + ack-what-succeeded (F-S5-1).
6. Test isolation: drain/DB tests must not consume the live outbox (F-X3-1).

## Extensibility note

Adding a **data source** is cheap and proven (five daemons deep; wake path is source-agnostic) —
the cost is chassis copy-paste, which the rework above fixes. Adding a **market** has no seam:
`symbol text` is the entity key in ~15 tables and every fold; sizing/order construction is
equities-shaped Python outside the spec; the trigger/bar model assumes RTH sessions; the broker
layer is Alpaca-hardcoded with no venue interface. The one seam worth cutting before any
options/crypto/Kalshi attempt: an instrument/venue abstraction (instrument id + quantization +
session calendar) under the ledger and router — everything else can stay equities-specific
until a second market is real.

## Coverage statement (honest)

- **Full depth:** the entire money path (trading, virtual_portfolios, audit + their migrations),
  the live runtime (temporal_execution, spawn/drain/wake/dispatch, durable_execution,
  live_bars, launch/worker, payload store, run store, data_gc, reconcilers, preflight
  structure), the language core (spec, compiler-lint framework, validation safety checks,
  engine registry census, exits, agentic harness, runner simulation), bar_builder, db/settings/
  env core, and all five core docs + roadmap + grammar-doc claims.
- **Skim depth (spot-checked, not line-audited):** individual engine template bodies,
  indicator implementations, evidence_features internals, CLI arg wiring, status/reporting
  modules, news/sec/market-events/earnings daemon internals beyond the emit+lease paths,
  retrieval registry/providers detail, research sandboxes, capabilities catalog internals,
  skeletons/bundles. Per-item depth is recorded line-by-line in REVIEW_PLAN.md.
- **Not exercised:** live provider paths (by design — read-only review); the `-m live` suite;
  Temporal server behavior beyond code reading. Lease-bug SQL semantics were verified against
  Postgres (temp table, rolled back); table sizes verified read-only against the dev DB.
- One suspected-only finding shipped as suspected: F-S1-1's runtime reachability (mechanism
  confirmed, no current spec triggers it). Everything labeled confirmed was traced through the
  actual code path, and the two most consequential claims (lease, wake-ack side effect) were
  reproduced/measured against the running system.
