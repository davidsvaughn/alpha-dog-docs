# The plain-English companion: what the review actually found, and whether it's all connected

Written 2026-07-06 as a companion to [`REMEDIATION_PLAN.md`](REMEDIATION_PLAN.md) (the technical
fix plan) and [`REVIEW_SUMMARY.md`](REVIEW_SUMMARY.md) (the ranked findings). This doc trades
precision for clarity: no code, no jargon where avoidable, every claim linked back to the
technical docs if you want the exact details. Finding IDs like **F-T3-1** are unique — grep
[`REVIEW_NOTES.md`](REVIEW_NOTES.md) for any of them to see the full evidence.

---

## The whole review in one paragraph

The architecture is good and most of the code is careful — the ledger math, the wake-delivery
pipeline, the workflow engine integration, and the clock discipline all reviewed *well*, and
the newest code is consistently the best code. The problems are real but they are not "the
design is wrong." They cluster into **four recurring habits** (ways of working that keep
producing the same kind of bug in different places) plus **one genuine structural gap**. One
bug is critical and live today (the lease); a handful are time bombs that today's simple
deployment hasn't armed yet but the roadmap's next steps will. Nothing here says "start over."
Several things say "stop doing X everywhere, do it once."

---

## Your actual question first: is there connecting tissue?

**Yes — but it's mostly habits, not architecture.** When I lay all ~60 findings side by side,
about three quarters of them are instances of four repeating patterns. The remaining quarter
are genuinely isolated bugs (every codebase has those). And there is exactly one finding I'd
call a true *design* gap rather than a habit. Here they are, roughly in order of how much pain
they're causing.

---

### Theme 1 — Same job, many copies (and a fix that never traveled)

**The habit:** when the codebase needs the same machinery again — a daemon, a lock, an order
flow — it copies an existing working version and adapts it, rather than sharing one
implementation. Copies then drift apart, and a bug fixed (or avoided) in one copy stays alive
in the others.

**The poster child is the review's only critical bug.** Every data-stream daemon uses a
"lease" — a database lock meaning *only one process may hold this connection*. This protects
the one rule the project calls firm: one connection per brokerage account. The lease logic was
copy-pasted six times. Four of the six copies have a subtle SQL bug that makes the lock a
polite fiction: **whoever asks for the lock is told "yes, it's yours" — even if someone else
already holds it.** Both processes then happily run at once, which for the Schwab connection is
exactly the disaster the lock exists to prevent. The two *newest* daemons have a correct
version — someone got it right later and the fix never traveled back to the older copies.
That's the theme in one sentence: the fix existed in the repo and copy-paste kept it from
spreading. *(F-T3-1, F-T3-2 → plan WS1. I proved the bug against a real Postgres before
calling it confirmed.)*

**Other members of this family:**
- Five daemons share ~the same scaffold (startup loop, status reporting, shutdown), each a
  hand-copy. *(F-D2-1 → WS9)*
- The "buy and confirm" and "sell everything and confirm" flows are ~250-line near-twins
  maintained separately — which is why one of them checks whether the account is blocked and
  the other forgot to. *(F-T1-6 → WS6)*
- Cost basis (what did we pay for this position?) is computed twice, once in Python and once in
  SQL, and the two disagree — that's your already-tracked Thread H bug. Two implementations of
  one financial truth will always re-diverge. *(M1 notes → WS9, last row)*
- There are two complete "make order submission crash-safe" mechanisms; one is the real one,
  the other is dead code that still looks load-bearing. *(F-S6-1 → WS9)*

**Does one fix cover this?** Largely yes, and it's the clearest "redesign that solves several
at once" in the whole review: **one shared lease module, one shared daemon chassis, one shared
order pipeline, one cost-basis implementation.** That's workstreams 1 and 9 — mostly deletion,
not new invention.

---

### Theme 2 — Rules without referees (the paperwork says more than the machine does)

**The habit:** the project is unusually good at *writing down* rules, contracts, and schemas —
and repeatedly stops short of building the robot that checks them. Where there's a written rule
with an automated check behind it, things are in great shape. Where the rule is prose- or
schema-only, reality has quietly drifted from the paperwork.

You've actually diagnosed this disease yourself already — it's the same "prose drifts from
executable behavior" problem behind `docs/policy/knowledge-promotion-loop.md`. The review found
it living in more places than docs:

- **The storage rule.** AGENTS.md says every fast-growing table must have a cleanup story. The
  automated checker only inspects *one category* of table. Result: three data stores nobody is
  cleaning — including **2.4 GB** of run artifacts on disk and an eval-snapshot table growing
  with every live minute. This is the exact disease that caused your 12 GB and 8.7 GB incidents;
  the checker just couldn't see these tables. *(F-S9-1, F-S11-1, F-T2-1 → WS3)*
- **The safety linter.** The validation check named `no_lookahead` — the thing both charters
  point to as proof a strategy can't cheat by peeking at the future — only fires if a strategy
  *confesses* (`lookahead: true` in its own config). It's an honor-system checkbox wearing a
  guarantee's name. The real protections exist elsewhere (in the data layer), but the label
  overpromises. *(F-E1-1 → WS8b)*
- **The backtest label.** Backtest reports stamp every trade "executed next bar after the
  signal," but the math actually fills at the *signal bar's own closing price* — a small,
  systematic optimism that matters for daily-frequency strategies (it silently pockets the
  overnight gap). The label says one thing; the arithmetic does another. *(F-E9-1 → WS8c)*
- **The strategy language's menu.** The spec system declares twelve kinds of building block;
  five of them (**including `action` — placing orders!**) have zero implementations. The
  design docs describe them as if they work. An agent reading the canon would design a strategy
  the runtime can't execute. *(F-E3-1, F-C6-1 → WS8d)*
- **A promise in a docstring.** The ledger code promises "a strategy sells only the positions
  it opened." The actual guard checks the whole shared book, not the strategy's slice — the
  promise has no referee. *(F-V1-1 → WS7b)*

**Does one fix cover this?** Not one *fix*, but one *policy*, applied a few times: **if it's
declared, something must enforce it — or it gets deleted/relabeled.** Concretely: make the
storage checker demand a verdict for every table (WS3); rename the honor-system checks to what
they really do and add one real behavioral probe (WS8b); relabel the backtest timing or fix the
math (WS8c); mark the five phantom building blocks as future-work in the canon docs (WS8d).
Same move, four places.

---

### Theme 3 — A three-part event squeezed into one yes/no

**The habit:** executing a trade is really three separate facts — *did we ask the broker?*,
*did the broker fill it?*, *did our books record it?* Several bugs come from compressing those
three facts into a single ok/not-ok flag, which then lies in one direction or the other.

- **Lies toward "success":** the process that babysits each open position asks the router to
  sell, and the router replies "ok" whenever the order reached *any* final state — including
  **rejected with zero shares sold**. The babysitter hears "ok," closes its books, and retires.
  The position is still there; nothing is watching it anymore. (The entry-side code gets this
  right — it checks whether a ledger entry actually exists. The exit side doesn't.) This is the
  #2 issue in the whole review and must be fixed before the cutover flips real routing on.
  *(F-S2-1 → WS2a)*
- **Lies toward "failure":** if the broker fills a buy but the *bookkeeping copy* fails, the
  whole operation reports failure. Anything that reasonably retries a "failed" buy would buy
  twice. The truthful answer is "broker: yes; books: not yet — fix the books, don't re-buy."
  *(F-T1-3 → WS6c)*
- **Same shape, smaller stakes:** a "was this symbol tradeable?" lookup caches three possible
  truths (yes / no / *couldn't reach the broker just now*) into one bit — so a single network
  hiccup gets remembered as "this stock doesn't exist," permanently, until restart. Real
  entries get silently skipped. *(F-T1-1 → WS6a)* And the live loop's order log marks any
  attempt "routed" whether or not it succeeded, which blocks re-entry on a symbol for half an
  hour after a buy that never happened. *(F-S2-3 → WS2c)*

**Does one fix cover this?** Mostly yes: **give money operations a result that carries all
three facts** (asked / filled / booked) instead of one boolean, and make every caller read the
fact it actually cares about. That's one contract change that fixes the exit bug, the
double-buy hazard, and the re-entry poisoning together, plus makes "who writes the alert row?"
systematic instead of per-function judgment. *(WS2a + WS6c + WS7c share this spine.)*

---

### Theme 4 — Correct for today's deployment, armed by tomorrow's roadmap

**The observation:** a lot of guards are only correct under simplifications that happen to be
true right now — one strategy, effectively one book per account, one copy of each daemon, no
concurrency, paper money. The design *documents* promise the general version (pooled books,
many strategies, parallel workflows), and the roadmap is actively walking toward it. Each step
of that walk arms one of these dormant bugs:

| The simplification that protects you today | The bug that arms when it ends | Roadmap step that ends it |
|---|---|---|
| One book per brokerage account | "Close position" sells *everyone's* shares and books them all to one portfolio *(F-T1-2)* | Second strategy/book on a shared account |
| One strategy instance per book | Book-level sell guard lets strategy B sell strategy A's shares *(F-V1-1)* | Multi-strategy books |
| One replica of each daemon | The lease bug *(F-T3-1)* | Any second process — even a debugging one-off |
| No concurrent orders | Check-then-submit races (the known Thread-H race + its account-level sibling F-T1-5) | Dual-run / parallel entry workflows |
| Low volume | A fresh database connection per tiny operation *(F-D4-1)* | Cutover-scale workflow traffic |
| Simple strategy graphs | Strategy steps run in file order, not dependency order *(F-S1-1)* | First multi-step research pipeline spec |

This isn't a new category for you — **Thread H already exists because you sensed exactly this**
("pre-scale correctness fixes"). The honest summary is that the review found Thread H's list is
about three times longer than what's written on it. The plan's sequencing is built around this
table: fix each row *before* the roadmap step that arms it, and treat those fixes as gates
(most explicitly: WS2 gates the cutover).

---

### Theme 5 — the one true *structural* gap: the strategy language stops at the decision

Everything above is habits. This one is architecture.

The whole system is built around a beautiful idea: a strategy is a typed, inspectable graph —
you can read it, validate it, replay it, reason about it. And that's true right up to the
moment the strategy decides *what it wants*. Everything after that — how many shares that
means, how the order is constructed, how it reconciles against what's already held — is plain
Python inside the runtime, invisible to the spec, the validator, and the catalog. **The most
safety-critical stretch of the pipeline is the least inspectable one.** It's also why adding a
new market (options, crypto, Kalshi) has no natural seam: there's no place in the *language*
where "acting" lives, so a new market means teaching the runtime's internals, not extending the
language. *(F-E3-1; full argument in [`ARCHITECTURE_REVIEW.md`](ARCHITECTURE_REVIEW.md) §1–§2)*

This is not the hidden parent of the small bugs — the lease bug has nothing to do with it. But
it *is* the one place where I'd say the design, not the habits, needs a decision from you:
either make "acting" a first-class part of the spec (the plan recommends starting small —
pull position-sizing policy into the spec, defer a full `action` node until a second market
forces the interface), or honestly shrink the language's claims to "the DAG owns judgment; the
runtime owns action." Both are defensible. The current in-between is what's not. *(WS8d)*

---

## What is genuinely NOT connected

For honesty's sake: some findings are just bugs, and forcing them into a theme would be
storytelling. The drain daemon dying when one message target has vanished *(F-S5-1)*, a
sort-key that would crash on data that can't currently occur *(F-V1-3)*, the eval-recorder
being able to kill the live actor *(F-S2-2)* — these are ordinary missed edge cases in a
codebase that otherwise handles failure isolation well. The test suite quietly consuming real
queue messages *(F-X3-1 — this is the one that acked 200 of your real pending wakes during the
review)* is its own small category: tests pointed at the live database. Important, quick to
fix, not deep.

## One more pattern — and it's good news

**Code quality tracks age, in the right direction.** The newest layers are the best ones: the
two newest daemons have the *correct* lease; the workflow layer (newest big subsystem) has the
strongest engineering and the best tests in the repo; the entry/position workflow tests are
genuinely adversarial. The debt is concentrated in the oldest layer (`trading/` — first thing
built, 2,335 lines, 32 lines of tests) and in things that were copied early and never
revisited. In other words: **your practices already improved — most of this plan is
retrofitting the old layers to the standard the new layers already meet.** That's a much
better position than the reverse.

---

## So: is there a larger redesign that solves several at once?

Direct answer: **no grand redesign is warranted, and I'd advise against looking for one** —
the fundamentals (event pipeline, ledger, spec language, workflow split) reviewed well and
the "keep" column dominates the verdict table. But there ARE exactly three medium-sized
consolidations that each kill a whole cluster rather than one bug, plus one discipline and one
decision:

1. **Share the machinery** (Theme 1): one lease implementation, one daemon chassis, one order
   pipeline, one cost-basis computation. Mostly deletion. → WS1 + WS9
2. **One truthful result for money operations** (Theme 3): asked/filled/booked as separate
   facts, one alert chokepoint. → the spine of WS2 + WS6 + WS7
3. **Total enforcement** (Theme 2): every declared rule gets a checker or gets deleted —
   storage checker covers every table, safety checks renamed or made real, phantom node kinds
   labeled. → WS3 + WS8
4. **The discipline** (Theme 4): before each roadmap step that removes a simplification, run
   the corresponding pre-scale batch. Thread H, made systematic.
5. **The decision** (Theme 5): where does "acting" live — in the language or in the runtime?
   Yours to make; the plan recommends the incremental path. → WS8d

If you internalize just three sentences from this whole review: *The architecture is sound and
the newest code proves the practices have already matured. The bugs come from copies that
drifted, promises without checkers, and one-bit answers to three-part questions. Fix by
sharing, enforcing, and un-flattening — not by rebuilding.*
