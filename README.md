# LevUp — Quadrant v5.3 Strategy
### Algorithmic Long-only Strategy on Indian Equity Markets
**KRITI '26 · Team No. 2617**

---

## Overview

Quadrant v5.3 is a fully systematic, regime-aware, long-only equity strategy backtested on Indian market data (NSE universe) over 2010–2020. It combines breadth-based macro regime detection, cross-sectional stock ranking, and a Genetic Algorithm portfolio optimizer to deliver consistent risk-adjusted outperformance against the Nifty 500 benchmark.

| Metric | Strategy | Nifty 500 |
|---|---|---|
| CAGR (2010–2020) | **24.41%** | 9.35% |
| Annualised Volatility | **15.30%** | 17.0% |
| Maximum Drawdown | **−16.98%** | −38.3% |
| Information Ratio | **1.0845** | — |
| Up-Capture Ratio | **106.26%** | — |
| Down-Capture Ratio | **25.19%** | — |

---

## Architecture

The pipeline runs end-of-day on **T** and executes at the open of **T+1**, in four phases:

```
Phase 1 (EOD T)  →  Data ingestion, risk pre-computation, regime detection
Phase 2 (EOD T)  →  Four-quadrant stock classification + scoring
Phase 3 (EOD T)  →  Genetic Algorithm portfolio optimization + exit evaluation
Phase 4 (Open T+1)  →  Delta execution (SELL → TRIM → ADD → BUY)
```

### Phase 1 — Regime Detection

Market regime is determined by **breadth analysis** — the fraction of the universe trading above their 200-day SMA — rather than index-level returns. A 30-day fast SMA and 150-day slow SMA of breadth are compared; the spread `D(t)` with a ±1σ hysteresis band determines the regime:

- `D(t) > +σD` → **NORMAL**
- `D(t) < −σD` → **DEFENSIVE**
- Otherwise → carry prior regime

This design prevents whipsaw flips during volatile sideways markets.

### Phase 2 — Quadrant Classification & Scoring

Each stock is classified into one of four quadrants daily using cross-sectional Market-Cap (MC) rank and Smart-Beta (SB) rank:

|  | High SB | Low SB |
|---|---|---|
| **High MC** | HH — Leaders | HL — Value/Cyclical |
| **Low MC** | LH — Emerging Stars | LL — Laggards |

- **NORMAL regime**: HH, HL, LH quadrants are eligible. Stocks are scored on a 63-day Price Inertia + Relative Strength composite (MAD-normalised to handle fat tails).
- **DEFENSIVE regime**: All quadrants eligible; stocks re-scored on Gain-to-Pain ratio minus Downside Deviation.

A **20% intra-sector cap** is applied before stocks enter the optimizer, preventing sector concentration.

### Phase 3 — Genetic Algorithm Portfolio Engine

The GA maximises a penalised multi-objective fitness function:

```
F(w) = f_Kelly(w) − λ₁·P_Sharpe(w) − λ₂·P_HRP(w)
```

- **Kelly objective**: Maximise geometric growth proxy `μₚ − ½σ²ₚ`
- **Sharpe penalty**: Fires when portfolio Sharpe < 60% of the MPT tangency Sharpe
- **HRP penalty**: Fires when any correlated cluster exceeds 35% of total weight

GA parameters:

| Parameter | Value |
|---|---|
| Population size | 100 |
| Generations | 40 |
| Selection | Tournament (size 5), top 20% élites preserved |
| Crossover | Uniform blend, β ~ U[0,1] |
| Mutation | Gaussian, ε ~ N(0, 0.02) |

### Phase 4 — Execution

Trades are executed at the **previous day's close** as a proxy for the next-day open (no same-day information leakage). Execution priority: `SELL → TRIM → ADD → BUY`. Positions use **fractional Kelly** sizing (`κ·f*`), with the remainder held as a cash buffer.

**Exit rules:**
- **Hard stop**: `P_current < P_entry − k·ATR14` (sector-adjusted multiplier `k ≈ 2.0–3.5`)
- **Multi-factor exit**: At least 2 of — EMA flip, OBV divergence, VPT negative trend

**Rebalance triggers:** start of quarter, regime switch, or emergency (< 15 active positions, subject to 15-day cooldown).

---

## Requirements

```
Python 3.10+
pandas
numpy
pyarrow
scipy
matplotlib
yfinance
```

Install dependencies:

```bash
pip install pandas numpy pyarrow scipy matplotlib yfinance
```

---

## Data Format

The script expects NSE price data in **Parquet format** with at minimum these columns:

| Column | Description |
|---|---|
| `tradedate` | Date of the trading day |
| `ticker` | Stock symbol |
| `open`, `high`, `low`, `close` | OHLC prices |
| `volume` | Daily traded volume |
| `mcap` | Market capitalisation |
| `sector` | GICS sector label |

---

## Configuration

Edit only the **USER CONFIGURATION** block at the top of the script:

```python
# Period 1 (required — e.g., 2010–2020)
PARQUET_FILE_P1 = "nse_prices_complete 1.parquet"

# Period 2 (optional — e.g., 2020–2025)
USE_SECOND_DATASET = False
PARQUET_FILE_P2 = ""

# Output root directory
OUTPUT_ROOT = os.getcwd()
```

To run a two-period backtest:

```python
USE_SECOND_DATASET = True
PARQUET_FILE_P2 = "nse_prices_2020_2025.parquet"
```

Period 1 history is used as warm-up for Period 2 indicator computation; trades only execute from the Period 2 start date.

---

## Running the Strategy

```bash
python 2617_CodeBase_QuantFinChallenge.py
```

Outputs are written into period-labelled subfolders (e.g., `2010_2020/`, `2020_2025/`) under `OUTPUT_ROOT`:

| Output File | Description |
|---|---|
| `quadrant_v4_1_results.png` | Equity curve, drawdown, breadth, and divergence charts |
| `trade_log_*.csv` | Full trade log (ticker, direction, qty, price, delta weight) |
| `exit_log_v4.csv` | All exit events with reason codes |
| `breadth_log_v4.csv` | Daily market breadth, regime state, and divergence scores |
| `*_perf_report.csv` | Consolidated performance metrics vs. Nifty 500 |
| `*_rolling_excess.png` | Rolling 1y / 3y / 5y excess return charts |

---

## Key Hyperparameters

| Parameter | Value | Role |
|---|---|---|
| SMA200 | 200 days | Long-term trend threshold for breadth |
| Breadth fast SMA | 30 days | Short-term breadth trend |
| Breadth slow SMA | 150 days | Long-term breadth trend |
| Regime band | ±1σ_D | Hysteresis threshold for regime flip |
| ATR lookback | 14 days | Stop-loss and volatility proxy |
| Momentum window | 63 days | Price Inertia and Relative Strength |
| Defensive scoring window | 252 days | GtP / Downside deviation lookback |
| GA population | 100 | Random portfolios per generation |
| GA generations | 40 | Max evolutionary cycles |
| Sharpe penalty threshold | 60% of MPT SR | Min acceptable Sharpe vs. tangency |
| HRP concentration limit | 35% | Max weight in largest correlated cluster |
| Sector cap | 20% | Max fraction of shortlist per GICS sector |
| Rebalance cooldown | 15 days | Min gap between emergency reshuffles |
| Min active positions | 15 | Emergency rebalance trigger |
| Target shortlist size | 25–100 stocks | Stocks entering GA optimization |

---

## Performance Summary (2010–2020)

### Rolling Outperformance vs. Nifty 500

| Window | Avg. Outperformance | Worst Underperformance |
|---|---|---|
| 1 Year | 24.71% | −10.12% |
| 3 Years | 27.14% | +8.24% |
| 5 Years | 27.47% | +15.47% |

The strategy never underperformed the Nifty 500 over any rolling 3-year or 5-year window in the backtest period.

### Capture Ratios

| Metric | Value |
|---|---|
| Up-Capture vs Nifty 500 | 106.26% |
| Down-Capture vs Nifty 500 | 25.19% |
| Capture Spread | 81.07pp |

The down-capture of ~25% means the strategy lost roughly **one-quarter** of what the benchmark lost during market downturns, while capturing all of the upside.

---

## Design Principles

Each major design choice directly addresses a documented failure mode from earlier approaches:

| Design Choice | Problem Solved |
|---|---|
| Breadth-based regime detection | RAAM equity correlation illusion |
| MAD Z-scoring | Gaussian fat-tail assumption failures |
| Walk-forward EMA (1-day lag) | In-sample look-ahead bias |
| Four-quadrant cross-sectional ranking | Intra-sector dispersion trap |
| 20% sector cap | Concentration risk |
| Multi-factor exits (2+ signals) | Single-signal false exits |
| GA with Kelly + Sharpe + HRP penalties | MPT fragility + Kelly over-betting |
| Sub-Kelly sizing with cash buffer | Catastrophic drawdown risk |
| 15-day rebalance cooldown | Crash-cascade over-trading |

---

## Note on Stochastic Reproducibility

The Genetic Algorithm uses stochastic operators (Dirichlet initialisation, uniform crossover, Gaussian mutation). A fixed seed (`np.random.seed(42)`) is set for reproducibility, but slight run-to-run variation is expected in a live environment. The performance metrics reported represent averages across multiple backtest runs, ensuring they reflect robust expectations rather than a single lucky draw.

---

## References

1. Kelly, J.L. (1956). *A New Interpretation of Information Rate.* Bell System Technical Journal, 35(4), 917–926.
2. Markowitz, H. (1952). *Portfolio Selection.* The Journal of Finance, 7(1), 77–91.
3. López de Prado, M. (2016). *Building Diversified Portfolios that Outperform Out of Sample.* The Journal of Portfolio Management, 42(4), 59–69.
4. Jegadeesh, N., & Titman, S. (1993). *Returns to Buying Winners and Selling Losers.* The Journal of Finance, 48(1), 65–91.
