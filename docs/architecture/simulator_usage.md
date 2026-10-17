# Using the Honba Simulator

The Simulator is the browser workbench for turning a strategy — built in the
[Algo Designer](algo_designer_and_simulator.md) or picked from the Library —
into a historical backtest with charts, trade markers and institutional
analytics. It ships as its own entry point, `simulator.html`, and is covered
by Phase 6–8 of the [Algo Designer & Simulator plan](algo_designer_and_simulator.md);
this page is the user-facing walkthrough of what that plan shipped.

## Opening a strategy

There are three ways a strategy ends up loaded in the Simulator, tried in
this order:

1. **Hand-off from the Designer.** Clicking **Simulate →** in the Algo
   Designer stashes the active graph in `sessionStorage` and opens
   `simulator.html?strategy=<slug>&source=designer`; the Simulator reads and
   validates it on load.
2. **Library selection.** If there's no hand-off (or it's missing/invalid),
   the Simulator falls back to the last strategy you had open from the
   Strategy Library.
3. **Built-in template.** With nothing else available, it falls back to a
   bundled template. Only `sma_crossover` is wired to a strategy `honba`
   actually executes today — `buy_and_hold`, `rsi_reversal` and `sma_crossover`
   are the three Rust-registered strategies, and a compiled graph only runs if
   it carries one of those names. Other templates and hand-authored graphs
   report `no_rust_implementation` until `honba` grows a generic IR
   interpreter (tracked as an open question in the plan, §11.7).

A reload restores your last selection, so you don't lose your place.

## Configuring a run

The controls panel sets:

- **Symbol / benchmark** — symbol search reuses the Workbench's instrument
  picker; benchmark defaults to NIFTY 50.
- **Period** — `1M` / `3M` / `6M` / `1Y` / custom date range.
- **Capital, fill model** — `bar_fill` by default.
- **Commission / fill and slippage** — sent on the wire as
  `commission_per_fill` (flat cost per fill) and `slippage_bps` (adverse,
  basis points), validated against the simulator's own ceilings. There is no
  margin control — that gap is deliberate and tracked, not silently ignored.

Clicking **Run Backtest** compiles the active graph against the session
catalogue, submits it, and polls until the run reaches a terminal status.

## Reading the results

- **Chart.** Candles plus the strategy's *own* indicator studies — the chart
  draws whatever the graph actually uses, with the graph's own parameters, so
  the chart and the run can't silently disagree. Buy/sell markers are placed
  from paired round trips, snapped to the bar they executed on. Markets-closed
  gaps (holidays, weekends) are left as gaps, never filled with synthetic
  candles.
- **Metrics ribbon.** Total Return, Sharpe, Max Drawdown and Total Trades are
  the run's wire values, echoed exactly. CAGR, Calmar, win rate, profit
  factor and exposure are *derived* from the closed round trips and labelled
  as such. **Sortino is `n/a`** — the periodic return series it needs isn't
  on the wire, and it is not estimated.
- **Equity / drawdown.** A *realised* equity curve rebuilt from the journal's
  round trips, daily-sampled over the run window, with a NIFTY 50 overlay
  shown only when a daily benchmark series is actually available. This is not
  a mark-to-market curve — the wire doesn't carry one.
- **Trade journal.** Filterable/sortable, CSV-exportable (RFC-4180; what
  "Excel export" means here is a CSV that opens cleanly in Excel, not an
  `.xlsx` writer). The **slippage column is intentionally blank** — fills
  carry no signal price, so slippage vs. signal cannot be computed honestly.
- **Monthly heatmap.** Realised PnL bucketed by IST month; it reconciles with
  the realised total by construction. The wire's `total_return` is shown
  alongside, labelled, since it need not equal the realised figure.
- **Trust badges / run log.** DSR/PBO/holdout/walk-forward badges from the
  same `ValidationBlock` the Designer uses. The log drawer tails the run's
  SSE event stream when available and degrades to the polled log otherwise.

## Sending sweeps back to the Designer

**Parameter Sweeps** stashes the active graph and opens the Designer's own
sweeps drawer — this hand-off is **navigation only**. `SweepReportResponse`'s
`best`/`ranked` fields are untyped on the wire, so the Simulator cannot read
or apply an "optimal combination" on your behalf; running and applying a
sweep still happens inside the Designer, using its existing local engine.

## Known limitations

These are deliberate, documented gaps — not bugs to report:

- No pending/unfilled-order markers (no run event stream carries them; only
  the derived open-position tail is drawn, and it's labelled as derived).
- No slippage-vs-signal figure (journal has no signal price).
- Sortino and mark-to-market equity are unavailable (no periodic return
  series on the wire).
- Sweep "apply optimal" from the Simulator side is not possible until
  `SweepReportResponse` is typed.

See [Risks & Mitigations](algo_designer_and_simulator.md#9-risks-mitigations)
in the plan for the full list and their tracking status.
