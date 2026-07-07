# Alpha-Claw design review (run 2026-07-06)

First-class deliverable: judges the design itself, not just drift from docs. We are not
starting over — each flaw is stated plainly first, then an incremental migration path.
Seeded with the questions to answer; observations deposited here as module reviews proceed;
consolidated at the end with keep/evolve/rework verdicts.

## Questions to answer

### 1. The node ontology (12 kinds: trigger/data/feature/agentic/condition/signal/risk/action/gate/notify/report/evaluator)
- Are these the right cuts? Which kinds are overloaded, underused, redundant, or dead?
- `gate` is declared but unimplemented (known). What else is dangling?
- Would a different taxonomy make specs simpler to write/read/verify?
- Observations:
  - (E3) **Template census: feature 29, evaluator 18, signal 11, condition 9, risk 9, trigger 2,
    agentic 1 — data/action/notify/report/gate: 0.** Five of twelve kinds are aspirational, not
    just `gate` as the docs admit (F-E3-1).
  - (E3/S1) **The most consequential absence is `action`.** The DAG's output is
    `desired_weights_raw`; everything from there to a broker order — sizing (`_buy_qty`),
    reconciliation against holdings, order construction, routing — is Python in
    temporal_execution.py, invisible to the spec, the linter, and the catalog. "A strategy is a
    typed, inspectable DAG" is true for judgment, false for action. Adding a market (options:
    contracts/legs; crypto: fractional/24h) means teaching the *runtime*, not the *language* —
    the opposite of the stated design goal.
  - (E1) `data` is likewise absent: bars/evidence/portfolio state are runtime *seeds*
    (well-known output handles like `portfolio.*`, `wake_events`), not declared nodes. That's
    actually a defensible design (data as ambient context), but then the kind shouldn't exist.
  - Verdict direction: shrink the declared ontology to what exists (7 kinds) OR make `action`
    real. The current in-between quietly relocates the highest-stakes semantics (sizing) into
    backend code.
  - (E1) The per-node policy surface (artifact/cost/timeout) is real and enforced for `agentic`
    (E6 harness) — the one place it matters most. `retry_policy`/`state_policy`/`safety_policy`
    are schema without readers.

### 2. Layering & dependency direction
- Claim: Temporal is one backend, not the ontology. Does the code deliver the separation?
  Is the intent itself well-conceived?
- Where do Temporal types / retry semantics / scheduling assumptions leak into "neutral" layers?
- Observations:
  - (S1/S2) The *mechanical* separation holds: workflow bodies are dict-orchestration; heavy
    compute/I-O is in activities; spec.py imports nothing Temporal; temporalio is an optional
    import with in-process fallbacks. Credit where due.
  - (S1/E3) The *semantic* separation does not: order sizing, desired-vs-held reconciliation,
    in-flight guards, dual-run capture, and the entry/position sub-DAG mechanics live in
    temporal_execution.py (2,335 lines). A hypothetical second backend would reimplement the
    money semantics, not just the scheduling. The "StrategySpec should be runnable by another
    backend" test would fail today on sizing alone.
  - (S6) The one artifact that LOOKS like a backend-neutral seam — durable_execution.py's
    step store — is dormant (F-S6-1): registered nowhere on the live path. Two
    durable-idempotency mechanisms, one dead, is worse than one.
  - Is the intent well-conceived? Partially. Backend-neutrality of *judgment* (the DAG) is
    cheap and achieved. Backend-neutrality of *execution* is expensive and unachieved — and
    arguably not worth pursuing until a second backend is real. The honest posture would be:
    "the spec is backend-neutral; the execution semantics are Temporal-native by choice,"
    and then invest in making the execution layer smaller, not more portable.
  - Positive: clock discipline in the runtime is exemplary (workflow.now(), completed-bar
    as_of computed deterministically, receive-time vs bar-time separation maintained).

### 3. State placement
- Postgres vs strategy YAML vs ledgers vs in-memory vs Temporal workflow state — principled or
  accidental? Where is the same fact stored twice (and which copy wins)?
- Observations:
  - (T1/V1/M1) **Derived financial state is computed in two languages**: cost basis exists as
    Python `fold_ledger` (lot-aware average cost) and as SQL views (lifetime average — the
    tracked Thread-H divergence). The deeper point: any basis logic expressed twice will
    re-diverge as accounting rules evolve. Recommend the views expose only raw aggregates
    (quantity, gross flows) and every basis/PnL number come from one fold implementation.
  - (M1) `strategy_runs` = files + DB index row is a good pattern (big artifacts out of the DB),
    but the pairing is unenforced — orphan risk both directions (F-M1-1).
  - (V1) The **account↔portfolio relationship is many-to-many by hash policy**
    (`_select_execution_account(portfolio_hash)`) while several call sites quietly assume 1:1
    (close_and_confirm sells the whole account position for one book — F-T1-2). The pooled-book
    model is real (tests exercise it); the trading/ layer hasn't caught up to it.
  - (T2/V1) Escalations (`alpaca_trade_escalations`) are the system's human-attention queue, but
    writing one is discretionary per call site: managed flows escalate, the VP router does not
    (F-V1-4). An invariant like "any money-path failure produces exactly one escalation" wants a
    single chokepoint, not per-function discipline.

### 4. Subsystem proportions
- `strategy/` (esp. engine + temporal_execution) dominates. Essential complexity or a smell?
  What would decompose it?
- Observations:
  - (S1/S2) temporal_execution.py is the de-facto runtime kernel: input dataclasses, 13
    activities, 6 workflow classes, order derivation/sizing, payload guards, exit-check glue.
    It's *well-written* but it's one file where five modules should be (inputs / activities /
    workflows / order-derivation / payload-guard). Decomposition is mechanical and low-risk.
  - (E1–E12) The engine package itself is proportionate: many small, single-purpose modules
    with colocated tests. The size of strategy/ is mostly essential complexity of the domain,
    plus ~4 retained legacy paths (live_worker, durable_execution, module-level load_live_bars,
    the raw-L1 provider) that inflate the reasoning surface (F-S6-1, F-S7-2, F-S8-1).
  - (X) A real smell: **six copy-pasted lease modules** and **five near-identical stream daemon
    scaffolds** (tick loop, status upsert, signal handling) — the ingestion layer wants one
    shared daemon chassis; the lease bug (F-T3-1) is the cost of not having it.

### 5. Accidental vs essential complexity
- Where has the design made simple things hard? (e.g. how many files must change to add a
  node template? a data source? a provider?)
- Observations:
  - **Cheap where it should be cheap:** adding a node template = one registry entry + impl +
    colocated test; adding an agentic provider = one adapter behind the uniform factory;
    adding an indicator = folder-per-capability. The extension seams the docs advertise are
    real for judgment-layer capabilities.
  - **Accidental complexity concentrates in duplication, not abstraction:** 6 lease copies
    (one wrong — F-T3-1/F-T3-2), 5 daemon chassis (F-D2-1), 2 durable-order mechanisms (one
    dormant — F-S6-1), 2 warmup paths with different source policies (F-S7-2), 2 cost-basis
    computations (Thread H), buy_and_confirm/close_and_confirm ~250-line twins (T1), a
    deprecated in-process live worker (F-S8-1). The repo's failure mode is not
    over-engineering; it's **retention** — superseded mechanisms stay resident and
    copy-paste divergence does the damage.
  - **The lifecycle rule is binding prose with a partial enforcer** — the guard sees
    hypertables only, so new append-only tables (snapshots, payload store, decisions, run
    artifacts on disk) accumulate unbounded until they become the next incident (F-S9-1,
    F-S11-1, F-T2-1). 2.4 GB of run artifacts + 250 MB payload store + 68 MB snapshots
    already exist.

### 6. What's missing
- What would adding a second market (options/crypto/Kalshi) or a genuinely new data source
  actually require? Which requirements have no seam today?
- Known hard-coded assumptions to watch: US-equities/RTH market hours, single-symbol
  positions, USD, whole-share vs fractional, "symbol" as the universal entity key.
- Observations:
  - **A new data source is genuinely cheap** — the seams exist and have been exercised five
    times: daemon + lease + status + source table + in-transaction wake emit + event_trigger
    source filter. The earnings-calendar stream (newest) proves the recipe. Cost: the chassis
    copy-paste (§5); benefit: nothing else needs to change — wakes are source-agnostic.
  - **A new market has no seam.** The blockers, concretely:
    1. `symbol: text` (uppercased US ticker) is the entity key in ~15 tables and every fold —
       an options contract (OCC symbol + expiry + strike + right) or a Kalshi market id has
       nowhere to live; `entity.is_tradeable` is SEC-registry-backed.
    2. Sizing/order construction is equity-shaped Python (whole shares, `_buy_qty`,
       market/limit only — F-E3-1): no order-type vocabulary in the spec, so no place to even
       *declare* multi-leg or contract semantics.
    3. Market-hours/bar assumptions: completed-1m-bar cadence, RTH grace, extended_hours as a
       boolean special case — 24/7 crypto or event-resolution markets (Kalshi) break the
       trigger model's clock, not just data.
    4. The broker layer is Alpaca-paper-specific with no venue interface (`make_trading_client`
       hardcodes; venue_policy selects *accounts*, not venues).
    5. USD is a column default everywhere but at least it IS a column — currency is the
       easiest of the five.
  - What SHOULD exist and doesn't: an instrument/venue abstraction (instrument id + venue +
    quantization rules + session calendar) under the ledger and the router. That's the one
    seam worth cutting *before* more markets are attempted; everything else can stay
    equities-specific until then.
  - Also missing (smaller): a single escalation chokepoint (§3), a required-DB CI lane
    (F-V2-1/F-X3-1), GC for the three unbounded artifact stores (§5).

### 7. Testing philosophy
- `_example_test.py` colocated tests-as-demonstrations: does it correlate with weak
  adversarial/edge coverage? How much of the money path is live-gated only?
- Observations:
  - (T1 vs V1) Coverage is **wildly uneven across the money path**: virtual_portfolios has
    real adversarial tests (guard blocks, replay reconciliation, pooled-account independence);
    trading/services.py — the layer that actually talks to the broker — has 32 lines testing two
    argument validations. The managed flows, recovery ladder, alignment math, and
    marketable-limit pricing (pure functions!) have zero non-live coverage.
  - (V2) DB-backed tests `pytest.skip` when Postgres/migrations are absent — a green
    `-m "not live"` run does not certify the ledger/router unless the environment happened to
    have the DB (F-V2-1). Coverage is environment-dependent and silently so.
  - (X3) Worse than skipping: DB-backed tests that DO run share the **live** dev database —
    the drain test acks real pending wakes as a side effect (F-X3-1). The `_example_test`
    philosophy is fine for pure modules (the engine proves it); the un-examined decision was
    pointing integration tests at production state.
  - **Verdict on the testing philosophy:** tests-as-demonstrations does NOT inherently produce
    weak coverage — virtual_portfolios and the workflow tests are adversarial and excellent.
    The correlation is with *module age and layer*: the oldest, most manually-verified layer
    (trading/) has the weakest tests. The philosophy needs two amendments: an isolation rule
    (tests never touch shared live state) and a floor rule (money-path modules get the same
    adversarial standard as virtual_portfolios).

### 8. Cross-cutting invariants
- Clock discipline (received_at_utc vs completed-bar vs vendor time) — consistent?
- Idempotency of order/ledger writes; one-connection-per-account lease enforcement.
- Secrets architecture (.env as canon) — sound?
- Observations:
  - (T3) **The one firm invariant is not enforced.** The lease upsert in 4 of 6 daemons
    (incl. Schwab) lets any contender steal an unexpired lease and reports success to both
    (F-T3-1, reproduced in SQL). The two newest daemons have the correct guarded upsert —
    evidence the fix exists in-repo but the concept was copy-pasted six times instead of shared
    (F-T3-2). Design lesson: an invariant this central deserves one implementation + one
    contention test, not a convention.
  - (T1/V1) Ledger idempotency is strong (locks + unique indexes + mismatch-raise); *broker*
    idempotency is delegated to client_order_id stability, which the router honors
    (pre-submit reconcile) but callers must supply — verify at S1/S3.

## Keep / evolve / rework verdicts

| Subsystem | Verdict | Justification |
| --- | --- | --- |
| StrategySpec language (`spec.py`, compiler, engine registry) | **Keep** | Typed, forward-compatible, well-linted; the judgment layer genuinely is inspectable and extensible. Shrink or implement the 5 dead kinds (F-E3-1) — a pruning, not a rework. |
| Virtual-portfolio ledger (`virtual_portfolios/`) | **Keep** | Decimal end-to-end, structurally idempotent, best test coverage in the repo. Fix Thread H by deleting the SQL basis columns (one fold implementation), add instance-scoped sell guard (F-V1-1). |
| Temporal runtime (`temporal_execution.py` + spawn/drain/launch) | **Keep, decompose** | The strongest engineering in the repo (idempotency, replay-safety, clock discipline) but it has absorbed order semantics that belong to the language (F-E3-1) and it's one 2,335-line file. Split mechanically; move sizing toward the spec. |
| Wake outbox + drain (`wake_events`, `live_dispatch`, `drain_*`) | **Keep** | Textbook transactional outbox with honest at-least-once semantics and layered money-side idempotency. Fix F-S5-1 (per-target signal isolation). |
| Trading managed flows (`trading/services.py`) | **Evolve** | Sound skeleton, but: account-level position aliasing (F-T1-2), retry-hostile result contract (F-T1-3), near-duplicate buy/close flows, and ~zero test coverage. Extract a shared managed-order pipeline and test the pure helpers. |
| Stream daemons + leases (`*_stream/`) | **Rework the chassis** | The per-source adapters are fine; the six-copy lease (one broken — F-T3-1, the review's only critical) and five-copy daemon scaffold must collapse into one shared implementation with a contention test. |
| Backtest runner (`runner.py`) | **Evolve** | Honest simulation shape, but fills-at-signal-close mislabeled as next-bar (F-E9-1) matters for daily-frequency credibility; validation layer needs behavioral (not declared) lookahead probes (F-E1-1). |
| Data lifecycle (data_gc + lifecycle check) | **Evolve** | Right philosophy (preserve-by-default, explicit opt-in deletion); the enforcement net has table-shaped holes (F-S9-1, F-S11-1, F-T2-1). Make the checker demand a verdict per CREATE TABLE. |
| Docs/charters/roadmap | **Keep** | Unusually honest and current; the trust-order + living-roadmap discipline works. Apply the repo's own status labels to the grammar canon's unimplemented kinds (F-C6-1). |
| Legacy/dormant paths (live_worker, durable_execution, module-level load_live_bars) | **Rework (delete or quarantine)** | Retention of superseded mechanisms is this codebase's main accidental-complexity generator (§5). |
