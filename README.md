# Fin-452 QTP — Momentum-Based Sector Rotation

A quantitative trading project implementing a monthly-rebalanced, dollar-neutral long-short strategy across the eleven SPDR S&P 500 sector ETFs. Built for Fin-452 at the University of Alberta.

---

## Strategy Overview

The strategy attempts to exploit sector-level momentum by rotating capital into relative outperformers and shorting relative underperformers each month.

**Pipeline:**

1. **Feature Engineering** — Rolling moving averages, volatility, Sharpe ratio, momentum, SPY correlation, and max drawdown are computed daily per sector using a configurable global window.

2. **Kalman Smoothing** — Each feature series is passed through a local-level Kalman filter (random-walk state + Gaussian noise) to reduce noise before clustering. Series are normalised to zero mean/unit variance before filtering so that a single `dW` parameter controls smoothing intensity uniformly across all features.

3. **Hierarchical Clustering** — At each month-end, the eleven sector ETFs are grouped into K clusters using Ward's D2 linkage on the smoothed feature vectors. An expanding-window normalisation prevents look-ahead bias.

4. **Signal Construction** — A composite signal is computed per sector as:

   ```
   composite = MA_signal × cluster_confirmation × (1 / volatility)
   ```

   where `MA_signal` is the normalised short/long MA crossover strength, `cluster_confirmation` is the fraction of cluster peers sharing the same directional signal, and inverse volatility down-weights noisy sectors.

5. **Weight Allocation** — Sectors with positive composite scores go long; negative scores go short. Weights are proportional to composite magnitude within each side. The portfolio is dollar-neutral at every rebalance (long weights sum to +1, short weights sum to −1). A median split is applied when all signals share the same direction to ensure the portfolio always makes a relative bet.

6. **Execution** — A 1-day execution lag is applied throughout: signals observed at month-end close are executed at the next trading day's open. Between rebalances, sector positions drift naturally with daily price changes rather than being reset to target weights each day.

**Baseline parameters:** window = 126 days, dW = 0.01, K = 3 clusters.

---

## Results Summary

The strategy was evaluated on two non-overlapping periods using baseline parameters:

| Period | Strategy Cumulative Return | SPY Cumulative Return |
|---|---|---|
| Training (Jun 2000 – Dec 2024) | Negative | Substantially positive |
| Testing (Jan 2025 – Feb 2026) | Negative | Negative |

The strategy did not generate a positive edge in either period. A parameter grid search over ~70 (window, dW) combinations on the training set failed to identify a robust positive-return region, suggesting the problem is structural rather than a matter of misconfiguration. The write-up discusses likely causes and next steps.

---

## Assumptions

The backtest does **not** model:
- Transaction costs or commissions
- Bid-ask spread
- Short-selling borrow fees
- Market impact
- Taxes
- ETF expense ratios (partially captured in adjusted price data)

Results should be interpreted as gross, frictionless performance. See the Assumptions section of the report for a full discussion.

---

## Reproducing the Report

### Requirements

- R ≥ 4.3
- Quarto ≥ 1.4
- Internet access (data is pulled live from Yahoo Finance via `tidyquant`)

### R Packages

```r
install.packages(c(
  "tidyverse", "tidyquant", "PerformanceAnalytics",
  "cluster", "gt", "slider", "doParallel",
  "dlm", "purrr", "patchwork", "timetk", "plotly"
))
```

### Rendering

Open `QTP.qmd` in RStudio and click **Render**, or from the terminal:

```bash
quarto render QTP.qmd
```

This produces `QTP.html` — a fully self-contained file with all charts and tables embedded. The full pipeline (data pull, feature engineering, Kalman smoothing, backtest, grid search) runs on render. On first run, expect ~10–20 minutes depending on machine speed due to the parallelised grid search.

> **Note:** The data pull requires a live internet connection. Results may differ slightly from the committed `QTP.html` if Yahoo Finance has updated historical adjusted prices since the last render.

---

## Project Structure

```
Fin-452-QTP/
├── QTP.qmd          # Source document — all code and write-up
├── QTP.html         # Rendered output (self-contained)
├── TODO.md          # Original project planning notes
└── Fin-452-QTP.Rproj
```

All R code lives in `QTP.qmd`. There are no separate script files.

---

## Key Functions

| Function | Purpose |
|---|---|
| `add_rollers()` | Computes all rolling statistics per sector |
| `kalman_smooth_all()` | Applies local-level Kalman filter to each feature column |
| `get_monthly_snapshots()` | Extracts last trading day of each month per sector |
| `evaluate_k()` | Elbow/silhouette diagnostics for K selection |
| `cluster_sectors()` | Hierarchical clustering at a single rebalance date |
| `generate_signals()` | Computes composite signal per sector |
| `construct_weights()` | Converts signals to dollar-neutral weights |
| `compute_portfolio_returns()` | Simulates portfolio P&L with proper intra-month position drift |
| `run_backtest()` | Runs the full pipeline end-to-end for a given parameter set |

---

## Author

Alex Hirtle — University of Alberta, Fin-452
