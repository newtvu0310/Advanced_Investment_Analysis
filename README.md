# VN Portfolio Screener — Correlation Clustering Approach

> Automated pipeline to select a **6-stock diversified portfolio** from 130+ HOSE-listed stocks using liquidity filtering, Sharpe ranking, and hierarchical clustering.

---

## Pipeline Overview

### Step 1 — Define Universe
- 130+ stocks manually curated across **5 sectors**: Công nghiệp & Logistics, Tài chính Ngân hàng, Tiêu dùng, Xây dựng Hạ tầng, Công nghệ
- Daily price data fetched via `vnstock` API (sources: VCI, TCBS)
- Period: `2023-06-01` → `2025-12-31`
- Built-in rate limiter: batch of 18 requests / 62s to avoid API throttling

### Step 2 — Liquidity Filter
- Compute average daily turnover (VND) over the **most recent 26 weeks (~6 months)**
- Apply **sector-specific minimum thresholds** (e.g. 50B for Banking, 2–3B for others)
- Stocks below threshold → excluded

### Step 3 — Return Filter
- Resample daily prices → **weekly closing prices**
- Compute weekly returns and filter out stocks where `mean_return ≤ risk-free rate`
  - Risk-free: 2.73%/year → converted to weekly equivalent
  - Ensures only stocks with positive excess return enter the portfolio

### Step 4 — Sharpe Ratio Calculation
- Formula: `Sharpe = (mean_weekly_return − rf_weekly) / std_weekly`
- All metrics kept in **weekly units** (no annualization)
- Ranks each stock by risk-adjusted performance

### Step 5 — Pearson Correlation Matrix
- Build a correlation matrix from weekly return series of all qualified stocks
- Handle missing data: drop stocks with < 50% valid weeks; impute remaining NaNs with off-diagonal median
- Enforce matrix symmetry

### Step 6 — Hierarchical Clustering (Ward Linkage)
- Convert correlation → distance matrix: `dist = 1 − correlation`
- Apply **Ward linkage** via `scipy.cluster.hierarchy`
- Cut dendrogram into exactly **6 clusters**

### Step 7 — Select Portfolio (1 stock / cluster)
- From each cluster, pick the stock with the **highest Sharpe ratio**
- Result: **6 stocks** with low pairwise correlation + strong risk-adjusted returns

### Step 8 — Quality Check & Visualization
- Compare average pairwise correlation: full universe vs. selected 6 (should be lower)
- Outputs:
  - Correlation heatmap (full universe + selected 6)
  - Scatter plot: Mean Return vs. Std (bubble size ∝ Sharpe)

### Step 9 — Export
- Results saved to `vn_6stock_screener.xlsx` with 5 sheets:
  - `Selected` — final 6 stocks with stats
  - `All Qualified` — all stocks that passed filters
  - `Clusters` — full cluster membership
  - `Corr (6 stocks)` — correlation matrix of selected portfolio
  - `Skipped` — stocks that failed to fetch

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| `vnstock` | Fetch historical price data from HOSE |
| `pandas` / `numpy` | Data processing & return calculation |
| `scipy` | Hierarchical clustering, distance matrix |
| `matplotlib` / `seaborn` | Heatmaps, scatter plots |
| `openpyxl` | Excel export |

---

## Key Parameters

| Parameter | Value |
|-----------|-------|
| Universe | 130+ stocks, 5 sectors |
| Data period | 2023-06-01 → 2025-12-31 |
| Return frequency | Weekly (Friday close) |
| Risk-free rate | 2.73%/year (SBV reference) |
| Clustering method | Ward linkage |
| Output size | 6 stocks (1 per cluster) |

---

## Output Example

```
FINAL_SYMBOLS = [
    'XXX',   # Tài chính ngân hàng  mean_r=+0.xxxx%/week  Sharpe=x.xxx
    ...
]
```
