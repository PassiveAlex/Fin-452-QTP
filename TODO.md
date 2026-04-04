# TODO — Fin-452 QTP: Sector Rotation via Kalman-Filtered Hierarchical Clustering

## Strategy Summary
Long/short sector rotation across 11 SPDR Select Sector ETFs. Kalman filter smooths each ETF's return series; agglomerative hierarchical clustering (Ward's D2 linkage) groups sectors monthly by filtered momentum/vol features; long the best cluster, short the worst. Benchmarked against SPY.

---

## 0. Setup
- [ ] Create `QTP.qmd` in repo root
- [ ] Add YAML header (self-contained, embed-resources, toc, code-fold)
- [ ] Load all required packages
- [ ] Define constants: tickers, date ranges, inception dates

**Packages needed:** `tidyquant`, `KFAS`, `cluster`, `factoextra`, `PerformanceAnalytics`, `plotly`, `ggplot2`, `dplyr`, `tidyr`, `gt`, `zoo`, `slider`, `foreach`, `doParallel`, `patchwork`, `lubridate`, `scales`, `timetk`

---

## 1. Data
- [ ] Pull daily OHLC for SPY + 11 sector ETFs via `tidyquant::tq_get()` (Yahoo Finance)
  - Download from: `1999-06-01` (buffer year for rolling lookbacks)
  - Through: `2026-03-31`
- [ ] Compute daily log returns and simple returns from adjusted close
- [ ] Handle dynamic universe: XLC joins June 2018, XLRE joins Oct 2015, all others Dec 1998
- [ ] Inspect data: missing dates, NA counts, universe coverage table
- [ ] Visualize raw price series and returns distributions

---

## 2. Kalman Filter
- [ ] Write `fit_kalman_local_level(y, H_mult, Q_mult)` using `KFAS`
  - Local-level model: state follows random walk, observations are noisy
  - **Use filtered state `att` — NOT smoother `alphahat` (would cause look-ahead bias)**
- [ ] Apply to all 11 sector ETFs' daily return series
- [ ] Plot: before/after comparison of raw vs. filtered returns for 2-3 ETFs
- [ ] Document `H_mult` / `Q_mult` parameter meaning (obs noise / state noise ratio)

---

## 3. Feature Engineering (Monthly)
- [ ] Write `compute_cluster_features(filtered_returns, rebal_date, tickers, horizons_mean, horizons_vol)`
  - Rolling mean filtered returns: 1m (21d), 3m (63d), 6m (126d), 12m (252d)
  - Rolling vol of filtered returns: 1m (21d), 3m (63d)
- [ ] Standardize feature matrix with `scale()` — check for constant columns (→ NaN)
- [ ] Only include ETFs with ≥252 days of history at each rebal date
- [ ] Write `get_dynamic_universe(date, inception_dates)` helper

---

## 4. Clustering
- [ ] Write `run_hclust_rotation(feature_matrix, n_clusters, linkage)` using base R `hclust()`
  - Compute distance matrix: `dist(feature_matrix, method = "euclidean")`
  - Fit: `hclust(d, method = linkage)` — default linkage: `"ward.D2"`
  - Cut tree: `cutree(h, k = n_clusters)` → named integer vector of assignments
  - Other linkage options to test in optimization: `"complete"`, `"average"`, `"single"`
- [ ] Write `label_clusters(assignments, feature_matrix)` — deterministic labeling by 1m momentum score
  - Compute each cluster's mean 1m-return feature across its members
  - Highest mean → "long", Lowest mean → "short", rest → "neutral"
  - This is more stable than using k-means centers since assignments are deterministic
- [ ] Plot dendrogram at a representative rebalance date using `factoextra::fviz_dend()`
  - Highlight clusters with colored branches and rectangles (`rect = TRUE`)
  - Include as a one-time illustration of the clustering structure
- [ ] Visualize cluster assignments over time as a sector allocation heatmap
  - X: rebal date, Y: sector ETF, color: long (green) / short (red) / neutral (gray)

---

## 5. Signal Generation & Strategy Logic
- [ ] Write `generate_signals(filtered_returns, rebal_dates, ...)` — loops over month-ends
  - Output: `rebal_date, ticker, signal, weight`
  - Weights: `+1/n_long` for long leg, `-1/n_short` for short leg
- [ ] Write `compute_portfolio_returns(signals, daily_returns, tc_bps = 10)`
  - Use raw simple returns (not filtered) for PnL
  - Apply turnover-based transaction cost on rebal days
- [ ] Document short-selling assumption (borrow cost not modeled — note as limitation)

---

## 6. Training Period (June 2000 – December 2024)
- [ ] Run backtest on training data
- [ ] Plot equity curve (strategy vs SPY, starting at 1.0)
- [ ] Report: annualized return, vol, Sharpe, max drawdown, Calmar, win rate, beta, alpha

---

## 7. Testing Period (January 2025 – March 2026)
- [ ] Run backtest on testing data using same parameters as baseline
- [ ] Plot equity curve vs SPY
- [ ] Report same metrics as training period

---

## 8. Combined Performance
- [ ] Combined equity curve (plotly) with train/test boundary line
- [ ] Drawdown chart (strategy vs SPY)
- [ ] Monthly returns calendar heatmap
- [ ] `gt` summary table: Strategy vs SPY × Train vs Test

---

## 9. Optimization
- [ ] Define parameter grid:
  - `n_clusters`: 2–5
  - `linkage`: `"ward.D2"`, `"complete"`, `"average"`
  - `H_mult`: 0.5, 1.0, 2.0, 5.0
  - `Q_mult`: 0.1, 0.5, 1.0, 2.0
  - Feature horizon sets: 3 variants
- [ ] Pre-compute 16 Kalman filter variants (4×4 H/Q) before parallel loop — hierarchical clustering is deterministic so no random seed concern
- [ ] Parallel grid search with `foreach` + `doParallel` — maximize Sharpe on training set
- [ ] Report top-10 parameter sets
- [ ] Sensitivity surface plots (heatmap: n_clusters vs linkage, H_mult vs Q_mult, etc.)
- [ ] Re-run with optimized parameters, compare to baseline

---

## 10. Write-up
- [ ] Intro
- [ ] Rationale (sector momentum / mean-reversion literature)
- [ ] ETF descriptions (analogous to "Contracts" in QTP-1)
- [ ] Strategy logic explanation
- [ ] Conclusion
- [ ] Lessons learned

---

## Notes / Pitfalls
- Log returns are not additive across assets — convert to simple returns before cross-sectional weighting
- Month-end rebalance dates: use `max(date)` per year-month from actual trading data (not calendar)
- SPY is benchmark only — never enters the cluster universe
- XLC/XLRE must not appear before their inception dates — verify in output
- If training Sharpe is suspiciously high (>5), check for look-ahead bias in Kalman usage
- Hierarchical clustering is deterministic (no random init) — a key advantage over k-means; no `set.seed()` needed per rebalance
- Ward's D2 linkage minimizes within-cluster variance (most analogous to k-means objective); good default
- With only 9–11 ETFs, cluster structure can be fragile — dendrogram cut height sensitivity matters more than with large datasets
