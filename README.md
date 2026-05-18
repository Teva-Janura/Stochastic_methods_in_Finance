# Floating-Strike Asian Call Option Pricer — AAPL

A numerical pricing study of a European floating-strike Asian call option on AAPL,
implemented via three complementary methods: a binomial tree, a normal approximation,
and Monte Carlo simulation.

---

## Overview

The option contract is defined as follows:

| Parameter | Value |
|-----------|-------|
| Underlying | AAPL (auto-adjusted close) |
| Option type | European floating-strike Asian call |
| Payoff | max(S_T − Ā_T, 0), where Ā_T is the arithmetic average of all daily closes |
| Maturity (T) | 0.5 years (6 months) |
| Binomial steps (n) | 25 |
| Risk-free rate (r) | 1% p.a. |
| Spot S₀ | $270.46 (AAPL close, 28 April 2026) |
| Volatility (σ) | 31.30% p.a. (calibrated from daily log-returns, 2020–2026) |

---

## Project Structure

```
.
├── data/
│   ├── AAPL_log_returns.csv       # Daily log-returns (2020–2026)
│   └── robustness_grid.csv        # 5×6 (σ, r) sensitivity grid prices
├── figures/
│   ├── robustness_check.png       # Sensitivity plots (σ and r)
│   └── normal_approx_convergence.png
└── notebook.ipynb                 # Main analysis notebook
```

---

## Methods

### 1. Binomial Tree (`price_asian_floating`)

A Cox-Ross-Rubinstein (CRR) tree with the standard parameterisation:

```
u = exp(σ √Δt),   d = 1/u,   q = (exp(r Δt) − d) / (u − d)
```

Because the Asian payoff depends on the path average, the tree is **non-recombining**:
each node tracks the cumulative price sum C_t alongside the current stock price S_t.
Backward induction discounts under the risk-neutral measure Q.

A vectorized, chunked variant (`price_asian_floating_vectorized`) enumerates all 2ⁿ
paths in batches to stay within RAM limits while exploiting NumPy broadcasting.

**Result: $14.05**

### 2. Normal Approximation (`price_asian_normal_approx`)

Under Q, D_n = S_n − Ā_n is approximated as Gaussian. Exact first and second moments
are computed in closed form using the GBM covariance structure:

```
E^Q[S_j S_k] = S₀² · exp(r(j+k)Δt + σ² min(j,k) Δt)
```

The expected payoff then has the closed-form solution for a half-normal:

```
E[max(D, 0)] = μ_D · Φ(z) + σ_D · φ(z),   z = μ_D / σ_D
```

**Result: $14.27** (+1.55% vs binomial; error shrinks as n → ∞ by the CLT)

### 3. Monte Carlo Simulation

GBM paths are simulated under Q with M paths and n time steps per path. The
floating-strike payoff is evaluated per path and discounted:

```
Price ≈ e^{−rT} · (1/M) Σ max(S_T^(i) − Ā_T^(i), 0)
```

A 25×22 convergence grid (n from 10 to 600, M from 500 to 60,000) confirms the
estimate stabilises around **$14.05–14.10** for large (n, M).

---

## Key Results

| Method | Price (USD) |
|--------|-------------|
| Binomial tree (n=25) | **14.05** |
| Normal approximation (n=25) | 14.27 (+1.55%) |
| Monte Carlo (n=600, M=60,000) | ~14.08 |

---

## Volatility Calibration

Daily log-returns on AAPL (2020-01-01 to 2026-04-28) were computed and analysed:

| Statistic | Value |
|-----------|-------|
| Daily mean | 0.000831 |
| Daily std (σ̂_daily) | 0.019798 |
| Annualized σ̂ | **31.30%** |
| Skewness | 0.0272 |
| Excess kurtosis | 6.40 |
| Jarque-Bera p-value | ≈ 0 |

Five daily returns exceeded ±10% (COVID crash March 2020, earnings July 2020,
tariff reversal April 2025). All were confirmed as legitimate market events and
retained in the calibration sample.

---

## Robustness / Sensitivity

Prices were computed across a 5×6 grid of (σ, r) values:

| | r=0% | r=0.5% | r=1% | r=2% | r=3% | r=4% |
|---|---|---|---|---|---|---|
| 0.70σ̂ | 9.62 | 9.78 | 9.95 | 10.28 | 10.62 | 10.96 |
| 0.85σ̂ | 11.68 | 11.84 | 12.00 | 12.33 | 12.66 | 12.99 |
| **1.00σ̂** | **13.73** | **13.89** | **14.05** | **14.37** | **14.70** | **15.03** |
| 1.15σ̂ | 15.79 | 15.94 | 16.10 | 16.42 | 16.74 | 17.06 |
| 1.30σ̂ | 17.84 | 17.99 | 18.15 | 18.46 | 18.78 | 19.10 |

Key findings: the price is **approximately linear in σ** (vega ≈ $43.7 per unit)
and only mildly sensitive to r (rho ≈ $31.98 per unit).

---

## Greeks (Finite Difference)

All Greeks were computed via central finite differences at the baseline parameters.

| Greek | Value | Interpretation |
|-------|-------|----------------|
| Delta (∂V/∂S₀) | 0.0519 | +$1 in S₀ → +$0.052 in price |
| Gamma (∂²V/∂S₀²) | ≈ 0 | Linear payoff in S₀ → zero gamma ✓ |
| Vega (∂V/∂σ) | 43.68 | +1 vol pt → +$0.44 |
| Volga (∂²V/∂σ²) | −0.93 | Slight concavity in σ |
| Vanna (∂²V/∂S₀∂σ) | 0.1614 | = Vega/S₀ ✓ (linearity sanity check) |
| Rho (∂V/∂r) | 31.98 | +1pp in r → +$0.32 |
| Theta (−∂V/∂T) | −14.31/yr | ≈ −$0.039 per calendar day |

The zero gamma and the identity Vanna = Vega/S₀ are exact theoretical results for
a floating-strike payoff linear in S₀, and serve as sanity checks on the implementation.

---

## Dependencies

```
yfinance
pandas
numpy
scipy
matplotlib
pandas_market_calendars
plotly
```

Install with:

```bash
pip install yfinance pandas numpy scipy matplotlib pandas-market-calendars plotly
```

---

## Notes

- All computations use a **250-day trading year**. Missing data (weekends and NYSE
  holidays) were dropped rather than interpolated to preserve path-dependency integrity.
- The binomial tree uses the **real-world probability p = 1/2** for tree construction
  and the **risk-neutral probability q ≈ 0.491** for pricing.
- The Monte Carlo convergence surface is saved as an interactive Plotly HTML figure.
