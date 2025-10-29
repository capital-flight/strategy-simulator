# Trading Strategy Simulator

A single-file, browser-based simulator for testing simple betting/trading processes. It tracks equity, performance stats, drawdowns, and year-by-year results. Two payoff generators are supported:

- **Binary model:** coin-flip style outcome with user-set win rate, average win/loss, and caps.
- **Multinomial model:** outcomes drawn from six buckets (outlier/big/small wins and losses) with editable weights and payoff ranges.

---

## Quick Start

1. Adjust inputs in the left panel.
2. Use **Auto-bet** controls (Start / Stop / Restart) to run simulations at **Fast** or **Ultra-Fast** speeds.  
3. After a run, press **Start** again to continue progressing.

> The page uses Chart.js from a CDN.

---

## UI Overview

**Left panel (Settings)**
- **Bankroll & Risk**
  - *Starting Tokens* — initial equity.
  - *Risk Type* — `Fixed Tokens` or `% of Total` (position size = fixed amount or equity × percent).
  - *Tokens / % Risked per Bet* — position size control.

- **Pacing**
  - *Trades per Year* — used to translate trade count into “years” for CAGR and to segment yearly stats.

- **Model Selection**
  - **Binary Model inputs** (win %, avg win/loss %, max win/loss %).
  - **Multinomial Model inputs** (weights for each bucket + min/max payoff ranges per bucket).
  - **Switch button** toggles between Binary and Multinomial inputs.

- **History**
  - Rolling list of the latest outcomes with severity coloring:
    - Wins: small (white), big/outlier (lime).
    - Losses: small (light red), big/outlier (red).

**Main panel**
- **Auto-bet controls** — count, speed (Fast or Ultra-Fast), Start/Stop/Restart.
- **Stats** — live metrics (see “Metrics” below).
- **Equity chart** — cumulative tokens vs. bet number (Y-axis on the right).
- **Yearly table** — one row per simulated “year” (every *Trades per Year* bets).
- **Yearly chart** — in-year cumulative return % for each year (separate colored lines).

---

## How Position Sizing Works

- If *Risk Type* = `% of Total`:  `position_size = current_tokens × (risk_percent / 100)`
- If *Risk Type* = `Fixed Tokens`:  `position_size = fixed_amount` (capped by current tokens)

P&L per bet is `position_size × (pct_move / 100)`.

---

## Model 1 — Binary Payoff Model

**Inputs**
- `Win %`
- `Avg Winning Trade (%)`, `Largest Winning Trade (%)`
- `Avg Losing Trade (%)`, `Largest Losing Trade (%)` (enter as positive numbers in the UI; losses are applied as negative)

**Process**
1. **Direction**: A trade is a win with probability `Win %`, else a loss.
2. **Magnitude**:
   - For wins, draw a positive percentage from a **Beta-shaped** distribution on `[0, max_win]` with mean near `avg_win`.
   - For losses, draw a negative percentage with magnitude drawn similarly on `[0, max_loss]` with mean near `avg_loss`.

**Distribution Details**
- The simulator maps your `avg` and `max` to a Beta(a, b) on `[0, 1]` then scales by `max`.
- Mean `μ = avg / max`, clamped to (0, 1).
- Concentration `k = 6` (tunable constant).
- `a = μ × k`, `b = (1 − μ) × k`.
- Sample `X ~ Beta(a, b)`, payoff% = `+X×max_win` (win) or `−X×max_loss` (loss).

This gives a unimodal draw centered near your desired average while respecting an absolute cap.

---

## Model 2 — Multinomial Payoff Model

**Inputs**
- **Weights** for each bucket (must be ≥ 0; they get normalized internally):
  - `Outlier Win`, `Big Win`, `Small Win`, `Small Loss`, `Big Loss`, `Outlier Loss`
- **Payoff Ranges (%)** for each bucket:
  - `outlierWin [min, max]` (positive)
  - `bigWin [min, max]` (positive)
  - `smallWin [min, max]` (positive, can start near 0)
  - `smallLoss [min, max]` (negative)
  - `bigLoss [min, max]` (negative)
  - `outlierLoss [min, max]` (negative)

**Process**
1. **Bucket selection**: Draw one bucket according to the provided weights.
2. **Magnitude**: Draw **uniformly** within the selected bucket’s `[min, max]` range.
   - Positive buckets yield positive %.
   - Loss buckets yield negative %.

Use this model when you want explicit control over the tail shape and the mix of small vs. large outcomes.

---

## Metrics

All metrics update after each bet:

- **Bets** — number of completed bets.
- **Annualized Return (CAGR)** — computed from starting equity and current equity using the “years” proxy:
  ```
  years = max(1, bets / trades_per_year)
  CAGR = (current_tokens / starting_tokens)**(1 / years) - 1
  ```
  Shown as a percent. Green if ≥ 0, red if < 0.

- **Profit Factor (PF)** — `gross_profit / gross_loss`. Shown as `∞` if there are wins but no losses yet.

- **Max Drawdown (%)** — peak-to-trough decline relative to the running peak of equity.

- **Avg Win Trade (%)** — average of win magnitudes (absolute percent).
- **Avg Loss Trade (%)** — average of loss magnitudes (absolute percent).

- **Win %** — `wins / bets × 100`.

- **Longest Losing Streak** — maximum consecutive losses so far.

**Yearly Table (per “year”)**
- Computed every `Trades per Year` bets:
  - `Return%` — year-over-year percent return.
  - `Max DD` — intra-year max drawdown.
  - `PF` — intra-year profit factor.
  - `AvgW%`, `AvgL%` — intra-year average win/loss magnitudes.
  - `LLS` — intra-year longest losing streak.
- Yearly chart shows in-year cumulative return % across trades 1..N for each year.

---

## Controls & Behavior

- **Start**: begins auto-bets for the specified count.
- **Stop**: halts auto-betting.
- **Restart**: stops, resets the sim, and clears charts/tables.
- **Speed**
  - *Fast*: runs on an interval.
  - *Ultra-Fast*: runs in batches using `requestAnimationFrame` for very large runs.

- **History**
  - Shows the most recent outcomes (newest on top).
  - Color coding reflects the dynamic classification (based on your Multinomial bucket thresholds) even when running the Binary model, so outliers and big moves stand out.

---

## Implementation Notes

- **Charts**
  - Chart.js line charts with Y-axis on the right.
  - Equity labels are simple bet indices.
  - Yearly lines are colored by HSL rotation for quick differentiation.

- **Random Sources**
  - Binary model uses a Beta sampler via Gamma draws (Marsaglia & Tsang) to hit your target average and cap.
  - Multinomial model uses a categorical draw (weights) and uniform sampling within bucket ranges.

- **CAGR Caution**
  - “Years” are derived from your `Trades per Year` setting. If you change *Trades per Year* mid-run, interpret the CAGR accordingly.

---

## Common Tweaks

- **Heavier tails in Multinomial model**  
  Shift weight from “small” buckets to “big/outlier” buckets and widen those ranges.

- **Risk tight/loose**  
  Lower or raise `% of Total` or `Fixed Tokens` to control path volatility and drawdowns.

---

## File Layout

Single file:
```
index.html
```
Includes:
- Inline CSS
- Inline JS
- CDN for Chart.js

No build, no dependencies beyond the CDN.

---

## Troubleshooting

- **Nothing happens on Start**: Ensure `Tokens / % Risked per Bet` is > 0 and, for fixed mode, ≤ current tokens.
- **Yearly rows don’t show**: They render every `Trades per Year` bets. Increase auto-bet count or lower `Trades per Year`.
- **CAGR looks odd**: Verify `Trades per Year` matches your intended cadence. CAGR assumes `years = bets / trades_per_year` (min 1).

---

## Why Two Models?

- **Binary**: When you think in terms of “hit rate + average payoffs with a cap,” this is fast and intuitive while still adding distributional realism via Beta sampling.
- **Multinomial**: When your edge comes from asymmetric tails or distinct regimes (lots of small losses with rare large wins, or vice versa), explicit buckets give you precise control.
