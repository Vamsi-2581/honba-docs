# Adapting Tucker Balch's "10 Ways Backtests Lie" to Honba

In quantitative finance, the gap between backtested simulation and live execution is the single greatest cause of strategy failure. As Dr. Tucker Balch (Georgia Tech / Lucena Research / J.P. Morgan) highlighted in his QuantCon presentation *"10 Ways Backtests Lie"*, high-performing backtests frequently reflect developer errors, execution delusions, and statistical artifacts rather than genuine market alpha.

This document analyzes Balch's 10 pitfalls and defines how Honba systematically defends against them through architectural contracts, runtime simulators, and statistical validation gates.

---

## The 10 Pitfalls and Honba's Countermeasures

| # | Balch Pitfall | Core Mechanism of Failure | Honba Defense & Architectural Implementation |
|---|---|---|---|
| **1** | **In-Sample Backtesting** | Overfitting models to historical calibration periods. | Strict Out-of-Sample (OOS) splits, Walk-Forward Validation, and CI gates requiring OOS Sharpe $\ge 50\%$ of in-sample. |
| **2** | **Survivorship Bias** | Using modern index constituents retroactively, dropping delisted failures. | Point-in-time universe resolution (`resolve_universe(as_of=date)`), survivorship-bias-free Parquet catalogs, and constituent tracking. |
| **3** | **Observing & Trading the Close** | Lookahead bias: inspecting bar $t$'s close and filling at that same close. | **ADR 0016 (`NextOpenExecution`)**: Sells/buys fill at $t+1$ open; multi-day settlement cycle tracking (T+1/T+2). |
| **4** | **Ignoring Market Impact** | Assuming zero market friction and infinite liquidity at mid/last prints. | Non-bypassable statutory Indian cost packs (STT, GST, stamp duty) + non-linear square-root market impact / slippage models. |
| **5** | **Capacity Delusions** | Taking massive positions in illiquid names that cannot absorb volume. | **ADR 0018 (`RiskStage`)**: Volume participation constraints ($\le 2.5\%$ ADV), lot sizing, and minimum liquidity gates. |
| **6** | **Data Mining Fallacy** | Searching $10^4$ combinations until random noise passes a Sharpe hurdle. | Deflated Sharpe Ratio (DSR), trial count tracking ($N$), hypothesis-first AI research contracts, and Bonferroni/Holm corrections. |
| **7** | **Stateful Strategy Luck** | Path-dependent rules (trailing stops, regime flags) biased by start date. | Staggered start-date Monte Carlo perturbations and state-convergence invariance tests. |
| **8** | **Buying at the Open** | Assuming effortless fills at `Open[t]` despite opening auction volatility. | Pre-open auction modeling, execution spread buffers, and post-open TWAP/VWAP execution schedules. |
| **9** | **Trusting Complex Models** | High-capacity neural networks/ensembles memorizing non-stationary regimes. | Parameter surface plateau validation (flatness score $\ge 0.4$), BIC penalties, and Occam's razor preference for parsimonious factor logic. |
| **10** | **Failing to Forward Test** | Deploying directly to live capital without incubating against live order books. | Symmetric `ExecutionPort` / `Adapter` contract enabling unified promotion from Backtest $\to$ Paper Incubation $\to$ Live Production. |

---

## Detailed Architectural Integration in Honba

### 1. In-Sample Backtesting vs. Walk-Forward Validation
* **The Vulnerability:** Developing an algorithm by tweaking hyperparameters on the full dataset causes the strategy to overfit to idiosyncratic past noise.
* **Honba Implementation:**
  - Honba mandates the three-tier research pipeline:
    - **Tier 1 (Vectorized Pre-filter):** Fast screening across candidates; outputs are treated strictly as an unvalidated *upper bound*.
    - **Tier 2 (Event-Driven Simulation):** Out-of-sample execution with full cash, settlement, and lot size constraints.
    - **Tier 3 (Walk-Forward & Monte Carlo):** Automated walk-forward validation across rolling folds:
      ```python
      from honba.algo_analytics import walk_forward

      wf = walk_forward(
          node=node,
          strategy=StrategyClass,
          train_months=24,
          test_months=6,
          n_folds=8,
      )
      # Requirements: Mean OOS Sharpe > 0.8, fold Std < 0.5, >= 6/8 folds profitable
      ```

### 2. Eliminating Survivorship Bias with Point-in-Time Data
* **The Vulnerability:** Selecting a basket from modern constituents (e.g. today's Nifty 50 or Alpha-30) ignores companies that were delisted, went bankrupt, or dropped out during market distress.
* **Honba Implementation:**
  - Universe resolution accepts an explicit `as_of: datetime.date` parameter (`resolve_universe(name, as_of=...)`).
  - Parquet data catalogs track the lifecycle of delisted instruments and maintain exact historical index inclusion dates.

### 3. Causal Timing & Lookahead Prevention (ADR 0016)
* **The Vulnerability:** Standard vector backtests calculate signals on bar $t$'s close and assume instantaneous fill at that close. In live trading, once the close is observed, the market is closed.
* **Honba Implementation:**
  - In `NextOpenExecution`:
    1. An order generated by session $t$'s close is submitted to working order state.
    2. The order fills strictly at session $t+1$'s open (`open_session`).
    3. Sells execute before buys to simulate cash generation.
    4. Sells credit proceeds that remain *unsettled* for `settlement_days` (T+1 or T+2), preventing immediate reinvestment of unsettled cash unless margin borrowing is explicitly configured.

### 4. Non-Bypassable Transaction Costs & Market Impact
* **The Vulnerability:** Ignoring execution costs and slippage turns negative-expectancy strategies into illusory winners.
* **Honba Implementation:**
  - In Honba, transaction costs cannot be disabled in standard backtests. Statutory Indian charges (STT, exchange turnover fees, SEBI turnover fees, stamp duty, GST, and broker commission) are calculated on every fill.
  - Market impact is modeled using slippage models scaling with order size:
    $$\text{Impact} = \kappa \cdot \sigma_{\text{daily}} \cdot \sqrt{\frac{\text{Order Qty}}{\text{ADV}}}$$

### 5. Liquidity Caps and Capacity Modeling (ADR 0018)
* **The Vulnerability:** High apparent returns in microcap securities that would collapse if real capital attempted to execute.
* **Honba Implementation:**
  - `RiskStage` enforces volume participation limits (e.g., capping order size at $1\% - 2.5\%$ of 20-day ADV).
  - Equal-weight engines (`ewr.py`) explicitly exclude unaffordable unheld stocks whose single-share price exceeds target allocation, preventing cash lockup and unexecutable allocations.

### 6. Deflating the Sharpe Ratio Against Data Mining
* **The Vulnerability:** Evaluating thousands of variations produces false discoveries via selection bias (p-hacking).
* **Honba Implementation:**
  - Honba integrates the Deflated Sharpe Ratio (DSR), which discounts the observed Sharpe ratio $\widehat{\text{SR}}$ based on the number of trials $N$ and the variance of returns across all tested variations:
    $$\text{DSR} = P\left(\text{SR}^* > 0 \mid \widehat{\text{SR}}, N, \text{skew}, \text{kurtosis}\right)$$
  - Research runs record the trial ledger in `run.json` to keep an auditable log of total parameter combinations evaluated.

### 7. Stateful Strategy Stability via Monte Carlo Start Dates
* **The Vulnerability:** Path-dependent logic (e.g. trailing stops, rebalancing calendars) can produce lucky outcomes purely because of the chosen start date.
* **Honba Implementation:**
  - `honba.algo_analytics.monte_carlo` evaluates start-date sensitivity:
    - Runs simulations across staggered start dates $t_0, t_0+1, \dots, t_0+k$.
    - Verifies that strategy final equity and drawdowns exhibit low dispersion across initialization offsets.

### 8. Realistic Opening Execution & Auction Friction
* **The Vulnerability:** Opening prints represent single auction prices with discrete supply/demand imbalances. Fills cannot be assumed without slippage.
* **Honba Implementation:**
  - Orders executing at the open incorporate opening-spread buffers.
  - Sub-daily intraday strategies delay execution by 5–15 minutes post-open to avoid auction turbulence, or execute over continuous trading intervals.

### 9. Parameter Robustness & Complexity Penalties
* **The Vulnerability:** Dense multi-parameter algorithms fit historical sample noise rather than structural economic behavior.
* **Honba Implementation:**
  - Honba's statistical validation requires parameter surface heatmap evaluation:
    - Measures the plateau score (the flatness of performance across neighbouring parameter values).
    - If a parameter change from $(20, 50)$ to $(21, 50)$ causes Sharpe to plunge from $1.8$ to $0.3$, the strategy is flagged as overfit and rejected.

### 10. Seamless Forward Testing via Unified Adapter Architecture
* **The Vulnerability:** Strategies moving straight from backtests into live trading suffer from unseen operational and state discrepancies.
* **Honba Implementation:**
  - Honba strategies implement a single polymorphic contract (`ExecutionPortLike`).
  - The exact same code runs against:
    1. **`NextOpenExecution`**: Offline backtesting against historical Parquet bars.
    2. **`PaperExecution` / `FakeAdapter`**: Forward paper trading on live exchange WebSocket streams without capital risk.
    3. **Live Adapters (Dhan, Zerodha, NSE)**: Real capital execution.
  - A mandatory incubation period in paper mode verifies latency, fill rates, and margin accounting prior to live deployment.

---

## Conclusion

By baking Tucker Balch's principles directly into Honba's engine defaults—causal next-open fills, mandatory cost modeling, volume participation gates, point-in-time universe tracking, and automated out-of-sample statistical validation—Honba transforms backtesting from an exercise in curve-fitting into a rigorous, production-grade validation pipeline.
