# Risk-Aware Sector Investing on PSX

**Which Pakistan Stock Exchange sectors deliver reliable risk-adjusted returns — and how do they react to SBP policy rate decisions?**

Capstone project — AuratTech Data Analyst Track
Team: Samia  Zainab

---

## Table of Contents

- [Overview](#overview)
- [Business Problem](#business-problem)
- [Dataset](#dataset)
- [Repository Structure](#repository-structure)
- [Methodology](#methodology)
  - [MP1 — Return & Risk by Sector](#mp1--return--risk-by-sector)
  - [MP2 — Reaction to SBP Policy Events](#mp2--reaction-to-sbp-policy-events)
  - [Capstone — Sector Reliability Score](#capstone--sector-reliability-score)
- [Key Findings](#key-findings)
- [SQL Techniques Used](#sql-techniques-used)
- [Recommendations](#recommendations)
- [Limitations & Next Steps](#limitations--next-steps)
- [Tech Stack](#tech-stack)
- [Deliverables](#deliverables)

---

## Overview

Pakistani equity investors have no simple, data-driven way to compare PSX sectors on two things that actually matter for portfolio decisions: **how good is the risk-adjusted return**, and **how predictably does the sector react to State Bank of Pakistan (SBP) policy rate decisions**. This project builds that comparison from raw price data using SQL — window functions, CTEs, joins, and conditional aggregation — and ends with a single **Reliability Score** that ranks six PSX sectors on both dimensions at once.

The project runs in three stages:

| Stage | Question | Output |
|---|---|---|
| **MP1** | How does each sector perform on average, across the full period? | Return-to-risk ranking |
| **MP2** | Does that behavior change around SBP policy events? | Predictability (swing consistency) ranking |
| **Capstone** | Which sectors are good *and* predictable? | Combined Reliability Score |

## Business Problem

- PSX investors lack a clear, data-driven view of which sectors deliver reliable risk-adjusted returns.
- SBP policy rate changes are known to move markets, but their sector-by-sector impact is not well understood.
- Portfolio decisions are often guided by sector reputation rather than measured return, risk, and predictability.
- This analysis identifies which of six PSX sectors are genuinely reliable — versus which only look good on a temporary macro tailwind.

## Dataset

| Feature | Details |
|---|---|
| **Source** | PSX historical prices ([dps.psx.com.pk](https://dps.psx.com.pk/historical)) + SBP MPC policy announcements |
| **Coverage** | January 2023 – present |
| **Scope** | 23 tickers across 6 sectors |
| **Unit of analysis** | Daily closing price, per ticker |
| **Core tables** | `sector_prices`, `policy_events` |

### Sectors & tickers

| Sector | Tickers | Rate sensitivity |
|---|---|---|
| Banks | MEBL, UBL, HBL, MCB, BAHL | Direct — via net interest margins |
| Textile | NML, GATM, ILP, KTML | Idiosyncratic — export/FX-driven |
| Cement | LUCK, DGKC, MLCF, FCCL, CHCC | Mixed — construction financing + local demand |
| Fertilizer | FFC, ENGRO, FATIMA, EFERT | Defensive — subsidy/input-cost driven |
| Oil & Gas Exploration | OGDC, PPL, POL, MARI | Idiosyncratic — global oil prices |
| Autos | INDU, PSMC, HCAR, MTL | Mixed, lagged — auto financing + FX |

### `policy_events` schema

| Column | Type | Notes |
|---|---|---|
| `date` | DATE | Date of the MPC decision |
| `old_rate` / `new_rate` | NUMERIC | Policy rate before/after (%) |
| `direction` | TEXT | `hike` / `cut` / `hold` |
| `change_bps` | INTEGER | Size of the move, in basis points |

## Repository Structure

```
.
├── data/
│   ├── sector_prices_banks.csv
│   ├── sector_prices_textile.csv
│   ├── sector_prices_cement.csv
│   ├── sector_prices_fertilizer.csv
│   ├── sector_prices_oilgas.csv
│   ├── sector_prices_autos.csv
│   └── sbp_policy_events.csv
├── sql/
│   ├── mp1_return_risk.sql        # daily_returns view + return/risk queries
│   ├── mp2_policy_sensitivity.sql # before/after labeling, swing consistency
│   └── capstone_reliability.sql   # combined MP1 + MP2 reliability score
├── reports/
│   ├── MP2_Sector_Sensitivity_Results.docx
│   └── Capstone_Report_MP1_MP2.docx
├── presentation/
│   └── PSX_Sector_Reliability_Presentation.pptx
└── README.md
```

## Methodology

### MP1 — Return & Risk by Sector

Daily returns are computed per ticker with `LAG()` over closing price, then aggregated to sector level.

```sql
CREATE VIEW daily_returns AS
SELECT
    date, ticker, sector, close,
    LAG(close) OVER (PARTITION BY ticker ORDER BY date) AS prev_close,
    (close - LAG(close) OVER (PARTITION BY ticker ORDER BY date))
      / LAG(close) OVER (PARTITION BY ticker ORDER BY date) AS daily_return
FROM sector_prices
WHERE is_anomaly = FALSE;
```

From there: average daily return, standard deviation (risk), annualized return/volatility, and **return per unit of risk** (`avg_return / stddev`), ranked with `RANK()`.

### MP2 — Reaction to SBP Policy Events

Sector returns are joined to `policy_events` on a **±10-day window**, each trading day is labeled `before`/`after` the nearest event with `CASE WHEN`, and results are aggregated with a `WITH` CTE.

```sql
CASE
    WHEN r.trade_date < e.date  THEN 'before'
    WHEN r.trade_date >= e.date THEN 'after'
END AS period
```

**Swing consistency** — the core predictability metric — is the standard deviation of each sector's (after − before) return across many individual events, computed with `FILTER (WHERE …)`:

```sql
AVG(daily_return) FILTER (WHERE period = 'after')  AS after_avg,
AVG(daily_return) FILTER (WHERE period = 'before') AS before_avg
```

Lower swing consistency = a more repeatable, predictable reaction.

### Capstone — Sector Reliability Score

```
Reliability Score = MP1 return_per_unit_risk ÷ (1 + MP2 swing_consistency)
```

This keeps risk-adjusted return as the primary driver while discounting sectors whose reaction to policy events is less consistent.

## Key Findings

| # | Finding | Evidence |
|---|---|---|
| 1 | **Banks react sharply and directly to SBP rate cuts** | Average daily return flips from **-0.30% before** to **+0.41% after** a cut — a ~71 bps swing, the largest of any sector |
| 2 | **Fertilizer — not Banks — is PSX's most predictable sector** | Lowest swing consistency of all six sectors (**0.90%**), despite being expected as the "defensive control" |
| 3 | **Fertilizer and Banks lead on the combined Reliability Score** | Fertilizer **0.084**, Banks **0.082** — the top two of six; Autos (**0.022**) ranks lowest on every measure |

### Reliability Score — full ranking

| Rank | Sector | Return per unit of risk (MP1) | Swing consistency (MP2) | Reliability Score |
|---|---|---|---|---|
| 1 | Fertilizer | 0.0848 | 0.00895 | **0.084** |
| 2 | Banks | 0.0831 | 0.01271 | **0.082** |
| 3 | Cement | 0.0623 | 0.01702 | **0.061** |
| 4 | Oil & Gas Exploration | 0.0467 | 0.01812 | **0.046** |
| 5 | Textile | 0.0347 | 0.01810 | **0.034** |
| 6 | Autos | 0.0219 | 0.01365 | **0.022** |

## SQL Techniques Used

- Window functions — `LAG`, `RANK`
- Aggregation — `AVG`, `STDDEV`, `COUNT`
- CTEs (`WITH`) for multi-step, readable logic
- `LATERAL` joins + `BETWEEN` date-range joins
- `CASE WHEN` for before/after labeling and sector classification
- `FILTER (WHERE …)` for conditional aggregation

## Recommendations

1. **Prioritize Banks and Fertilizer** in a reliability-tilted PSX sector allocation — the top two on combined risk-adjusted return and policy-event predictability.
2. **Treat Banks' positioning around SBP MPC meetings as a tactical signal** — average return swings ~71 bps before vs. after a rate cut.
3. **Reduce or avoid Autos exposure** — the weakest risk-adjusted return with no offsetting predictability benefit.
4. **Investigate Textile and Cement's counter-intuitive post-event moves** before using them in a policy-driven strategy.
5. **Treat Oil & Gas Exploration as an oil-price play, not an SBP play** — pair with global crude data before drawing conclusions.

> Recommendations are framed as things to test or investigate further, not proven solutions — findings in this project are associative, not causal.

## Limitations & Next Steps

**Limitations**
- Findings are associative, not causal — they reflect historical correlation with SBP events, not a proven cause.
- The ±10-day symmetric event window may blend pre-event anticipation with post-event reaction.
- Uneven sample sizes across sectors (2,700–4,500 trading days) affect cross-sector confidence.
- Based on Jan 2023 – present only; may not generalize to other rate cycles.

**Next Steps**
- Extend the dataset to cover multiple full hike/cut cycles.
- Add global oil price and FX data to separate SBP effects from other macro drivers.
- Backtest a reliability-score-weighted portfolio against an equal-weight PSX benchmark.
- Test shorter or asymmetric event windows to validate the ±10-day choice.

## Tech Stack

- **PostgreSQL** — data modeling and analysis (views, CTEs, window functions)
- **PSX historical data** ([dps.psx.com.pk](https://dps.psx.com.pk/historical)) — price source
- **SBP MPC announcements** — policy event source
- **Excel / CSV** — raw data collection and staging
- **PowerPoint** — final stakeholder presentation

## Deliverables

- [`reports/MP2_Sector_Sensitivity_Results.docx`](reports/MP2_Sector_Sensitivity_Results.docx) — full MP2 query results and insights
- [`reports/Capstone_Report_MP1_MP2.docx`](reports/Capstone_Report_MP1_MP2.docx) — combined MP1 + MP2 write-up with the Reliability Score
- [`presentation/PSX_Sector_Reliability_Presentation.pptx`](presentation/PSX_Sector_Reliability_Presentation.pptx) — 12-slide stakeholder presentation

---

*AuratTech Data Analyst Track — Capstone Project*
