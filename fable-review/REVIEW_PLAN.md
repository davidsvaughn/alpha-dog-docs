# Alpha-Claw comprehensive review — plan (run 2026-07-06)

Reviewer: Claude (Fable 5), builder mode per AGENTS.md. Read-only review; no code changes, no
daemons, no orders. Working files: this plan, `REVIEW_NOTES.md` (append-only log),
`CROSS_FLAGS.md` (living cross-module flags), `ARCHITECTURE_REVIEW.md` (design review).

**Scale:** 256 py files, ~56,065 LOC in `alpha_claw/`; 27 migrations; docs tree.

**Already tracked — do not re-report as new** (note open/closed only):
- Thread H: `virtual_portfolio_positions` view cost-basis divergence from `fold_ledger`; router
  sell-guard read-then-submit race.
- ARCHITECTURE Known Gaps: no uniform control-plane API; registry unification partial; no
  real-money gate (by design); Langfuse unwired; `gate` node kind declared-but-dead;
  `entity_resolve` not a node; live regression coverage gated.
- Roadmap: over-eager entry rate (Thread I); dual-run cutover double-trade risk (Thread A);
  as-of retrieval missing for event-driven backtests (deferred list).

## Depth legend
`full` = read module + its test file, traced key paths. `skim` = read for shape + spot-checked
hot spots. `skipped` = not read (reason required). Recorded per item on completion.

## Priority order & checklist

### P0 — Money, order flow, ledger (highest consequence)
- [x] T1. `trading/services.py` (2335) — order routing/submission, sell guard, idempotency — depth: full
- [x] T2. `trading/audit.py` + migrations 010/011 (audit + escalations) — depth: full
- [x] T3. `trading/tools.py` + `trading/stream_daemon.py` + `stream_lease.py` + `stream_status.py` — depth: full (daemon/lease), skim (tools, stream_status)
- [x] V1. `virtual_portfolios/services.py` (1795) — fold_ledger, cost basis, PnL math — depth: full
- [x] V2. `virtual_portfolios/tools.py` + all three VP test files — depth: full
- [x] M1. Migration 015 (virtual_portfolios) + 016 (strategy_runs) schema review — depth: full
- [x] X1. Cross-cutting: idempotency & clock audit of the order→ledger path (trace one buy and
  one sell end-to-end from signal to ledger row) — depth: full (traced across T1/V1/S1–S3 notes:
  wake → drain → EntryWorkflow/cascade → route_virtual_trade → Alpaca → fill snapshot → ledger;
  idempotency chain sound when client_order_id is workflow-minted; gaps logged as F-T1-*/F-S2-*)

### P1 — Live strategy runtime
- [x] S1. `strategy/temporal_execution.py` part 1: activities (data/portfolio/routing) — depth: full
- [x] S2. `strategy/temporal_execution.py` part 2: LiveStrategyWorkflow + StrategyWorkflow + its test — depth: full (module; test file read in S3 pass)
- [x] S3. `strategy/entry_spawn.py` + `entry_decisions.py` + entry/position workflow tests — depth: full
- [x] S4. `strategy/wake_events.py` + `wake_decisions` + migrations 014/022/023/027 — depth: full
- [x] S5. `strategy/drain_loop.py` + `drain_gates.py` + `drain_shadow.py` + tests — depth: full (loop/dispatch/shadow), skim (gate internals beyond header)
- [x] S6. `strategy/durable_execution.py` + test + migration 013 — depth: full
- [x] S7. `strategy/live_bars.py` + `market_data.py` + `stream_data.py` + `market_bar_consumers.py` + migrations 017/018 — depth: full (live_bars), skim (market_data, stream_data, market_bar_consumers)
- [x] S8. `strategy/live_worker.py` + `live_dispatch.py` + `temporal_worker.py` + `temporal_launch.py` — depth: full
- [x] S9. `strategy/health_monitor.py` + `data_gc.py` + migration 026 — depth: full (data_gc, lifecycle check), skim (health_monitor)
- [x] S10. `strategy/reconcile_late_fills.py` + `reconcile_stream_subscriptions.py` — depth: full (late fills), skim (stream subscriptions)
- [x] S11. `strategy/temporal_payload_store.py` + migration 019 + `run_store.py` — depth: full
- [x] S12. `strategy/live_preflight.py` + `live_status.py` + `portfolio_status.py` + `portfolio_context.py` — depth: full (preflight, portfolio_context), skim (live_status, portfolio_status)
- [x] S13. `strategy/portfolio_displacer.py` + `displacement_shadow.py` (parked Thread J) — depth: full
- [x] S14. `strategy/thesis.py` + `engine/thesis_nodes.py` + migration 021 — depth: full

### P2 — Strategy language & engine
- [x] E1. `strategy/spec.py` + `validation.py` — the ontology itself (design review anchor) — depth: full
- [x] E2. `strategy/compiler.py` + test — depth: full (lint framework + topo order), skim (individual check bodies beyond safety-relevant ones)
- [x] E3. `strategy/engine/registry.py` + `engine/example_test.py` — depth: full (census/contracts), skim (template bodies)
- [x] E4. `strategy/engine/exits.py` + `allocation.py` + `selection.py` + `factors.py` + `primitives.py` — depth: full (exits, primitives), skim (allocation, selection, factors)
- [x] E5. `strategy/engine/triggers.py` + `branching.py` + `divergence.py` + `liquidity_screen.py` + `news_triage.py` — depth: full (triggers), skim (rest)
- [x] E6. Agentic stack: `engine/agentic.py` + `agentic_structuring.py` + `native_agentic_provider.py` — depth: full (agentic.py), skim (structuring, native provider)
- [x] E7. Agentic providers — depth: full (factory, deepseek structure), skim (pipeline, langgraph)
- [x] E8. `engine/company_prefill.py` + `evidence_features.py` + `evidence_resolution.py` — depth: full (evidence_resolution), skim (company_prefill, evidence_features)
- [x] E9. `strategy/runner.py` (backtest determinism / lookahead) + runner tests — depth: full (simulation/decisions), skim (metrics assembly)
- [x] E10. `strategy/library/registry.py` + indicators + `library/models.py` — depth: skim (thin pandas-ta wrappers)
- [x] E11. `strategy/skeletons/` + `bundles.py` + `skeleton_library.py` + `skeleton_coverage.py` — depth: skim (declarative, no money path)
- [x] E12. `strategy/cli.py` + `strategy/catalog.py` + `subdag.py` + `links.py` + `robustness.py` — depth: full (subdag, robustness), skim (cli, catalog, links)

### P3 — Ingestion & shared core
- [x] D1. `schwab_stream/` full pass — depth: full (bar_builder, lease), skim (daemon, events, tools, status, token_status)
- [x] D2. `news_stream/` full pass — depth: full (events/adapters emit path), skim (daemon, tools, status, import_legacy)
- [x] D3. `sec_stream/` + `market_events_stream/` + `earnings_calendar_stream/` — depth: skim (shared scaffold; emit paths verified)
- [x] D4. Core: `__init__.py` + `db.py` + `settings.py` + `market_cache.py` + `market_bars.py` + `market_halts.py` — depth: full (init/db/settings), skim (market_*)
- [x] D5. `entity.py` + `llm_costs.py` + rest — depth: full (llm_costs structure, entity entry points), skim (rest)

### P4 — Retrieval, research, capabilities
- [x] R1. `retrieval/services.py` + `providers.py` + `models.py` — depth: full (contract/fallback), skim (providers, models)
- [x] R2. `retrieval/registry.py` + `tools.py` + `volume_delta.py` — depth: skim (declarative metadata)
- [x] R3. `research/` — depth: skim (sandbox/research code, no money path)
- [x] R4. `capabilities/` — depth: skim (derived read-only index)

### Docs & schema (review targets, not just orientation)
- [x] C1. Charters: AGENTS.md + ASSISTANT.md — drift + design quality — depth: full
- [x] C2. docs/ARCHITECTURE.md — depth: full
- [x] C3. docs/RESOURCE-MODEL.md — depth: full
- [x] C4. docs/STRATEGY-LIFECYCLE.md — depth: full
- [x] C5. Migrations 001–027 as a whole — depth: full (015/016/010/011/013/014/017/018/019/020/022/023/026/027 read; earlier stream-core migrations skimmed via their consumers)
- [x] C6. docs/design/agentic-strategy-dag-grammar.md + strategy-runtime-architecture.md — depth: full (grammar structure/claims), skim (runtime-architecture)

### Cross-cutting passes (after module reads)
- [x] X2. Clock audit — depth: full (grep sweep + boundary tracing; clean)
- [x] X3. Test-suite run — depth: full (575 passed / 1 failed → F-X3-1; no lint/typecheck configured in pyproject)
- [x] X4. Live-only coverage map — depth: full (consolidated in X notes + ARCH_REVIEW §7)
- [x] X5. Secrets handling sweep — depth: full (clean)

### Final
- [x] F1. Consolidate ARCHITECTURE_REVIEW.md (keep/evolve/rework verdicts) — done
- [x] F2. Write REVIEW_SUMMARY.md — done

## Notes on ordering
Money path first (T/V/M), then the live runtime that drives it (S), then the language/engine
that decides trades (E), then ingestion that feeds it (D), then retrieval/capabilities (R),
docs interleaved near the end when code truth is established, cross-cutting passes last.
Design-review observations flow into ARCHITECTURE_REVIEW.md continuously.
