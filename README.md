# Monte Carlo Pricing and Risk Engine: Option Valuation & Portfolio Risk Analysis

This repository implements an end-to-end quantitative finance engine using Monte Carlo simulation to price European options, estimate option sensitivities (Greeks), and measure portfolio risk through Value at Risk (VaR) and Expected Shortfall (ES).

Rather than relying exclusively on simulation outputs, the project validates its numerical methods against analytical financial models, including Black–Scholes option pricing, analytical Greeks, and put-call parity. It also compares parametric Monte Carlo risk estimates with historical risk measures and evaluates their performance through out-of-sample backtesting.

The main challenges addressed in this project are stochastic price simulation, Monte Carlo estimation error, variance reduction, numerical sensitivity estimation, correlated asset returns, and the limitations of normal-distribution assumptions in financial risk modelling.

## Results

### 1. European Option Pricing

Evaluated using AAPL market data, a one-year maturity, a 4% assumed risk-free rate, and historical annualized volatility of approximately 30.5%.

| Metric | Black-Scholes | Monte Carlo |
|---|---:|---:|
| European Call Price | 40.4428 | 40.5114 |
| European Put Price | 43.4864 | 43.4297 |
| Call Standard Error | — | 0.0728 |
| Put Standard Error | — | 0.0525 |

**Call price validation:**
- Monte Carlo simulations: 1,000,000
- 95% confidence interval: [40.3687, 40.6541]
- Black-Scholes price: 40.4428
- The analytical price falls inside the Monte Carlo confidence interval.

**Put-call parity validation:**

| Metric | Value |
|---|---:|
| Monte Carlo Call - Put | -2.9183 |
| Theoretical S₀ - Ke^(-rT) | -3.0435 |
| Absolute Difference | 0.1252 |
| Standard Error of Difference | 0.0898 |

The difference is approximately 1.4 standard errors, consistent with ordinary Monte Carlo sampling variation.

### 2. Greeks Estimation

Greeks are estimated numerically using the bump-and-reprice method with common random numbers and compared against analytical Black-Scholes values.

| Greek | Monte Carlo | Black-Scholes |
|---|---:|---:|
| Delta | 0.5499 | 0.5492 |
| Vega | 136.6167 | 136.3352 |

Both estimates closely match their analytical counterparts.

### 3. Variance Reduction

Antithetic variates were evaluated over 300 independent repetitions, using 10,000 payoff evaluations per estimate.

| Method | Standard Deviation of Estimates |
|---|---:|
| Plain Monte Carlo | 0.65931 |
| Antithetic Variates | 0.64636 |

**Variance reduction factor: 1.04x**

Antithetic sampling provides a modest improvement in this particular experiment. It pairs each normally distributed random shock with its negative counterpart and averages the corresponding payoffs.

### 4. Portfolio Risk: VaR and Expected Shortfall

Portfolio assumptions:
- Initial portfolio value: 100,000 (arbitrary monetary units).
- Assets: AAPL, MSFT, JPM.
- Portfolio allocation: Equal weights (approximately 33.33% per asset).
- Risk horizon: One trading day.
- Monte Carlo simulations: 500,000.

**Monte Carlo vs Historical Risk Estimates**

| Confidence | Method | VaR | Expected Shortfall |
|---|---|---:|---:|
| 95% | Monte Carlo | 2,384 | 3,021 |
| 95% | Historical | 2,218 | 3,517 |
| 99% | Monte Carlo | 3,416 | 3,916 |
| 99% | Historical | 4,100 | 6,141 |

The normal-based Monte Carlo model produces substantially lower tail-risk estimates at the 99% confidence level than the historical method. This highlights the limitations of normal-distribution assumptions when modelling extreme market movements.

### 5. Out-of-Sample VaR Backtesting

The historical return series is divided chronologically:
- Training period: First 70% of observations.
- Testing period: Remaining 30% (591 trading days).

VaR thresholds are estimated using the training period and evaluated against realized test-period losses.

| Confidence | Expected Exceedance | Monte Carlo | Historical |
|---|---:|---:|---:|
| 95% | 5.0% | 2.2% | 3.4% |
| 99% | 1.0% | 0.5% | 0.3% |

Both methods produce fewer exceedances than theoretically expected, indicating conservative risk estimates during the test period.

However, the 99% backtest contains very few exceedances, making strong statistical conclusions difficult.

## Workflow Breakdown

1. **Market Data Collection and Exploration**

   Historical adjusted closing prices are downloaded using `yfinance` for four financial instruments:
   - SPY (S&P 500 ETF)
   - AAPL (Apple)
   - MSFT (Microsoft)
   - JPM (JPMorgan Chase)

   Data covers January 2019 through October 2026, containing 1,969 trading-day observations across four assets.

   Daily log returns are calculated as:

   `r_t = ln(P_t / P_(t-1))`

   Annualized historical mean returns and volatilities are estimated using 252 trading days per year.

   Observed annualized volatility:
   - SPY: 19.2%
   - AAPL: 30.5%
   - MSFT: 29.1%
   - JPM: 29.1%

   The diversified SPY index exhibits lower volatility than individual stocks over the observed period.

2. **Geometric Brownian Motion (GBM)**

   A stochastic price simulator is implemented to generate future asset-price trajectories under the risk-neutral measure.

   The model follows:

   `dS_t = rS_t dt + σS_t dW_t`

   Its discrete-time implementation is:

   `S_(t+dt) = S_t × exp((r - σ²/2)dt + σ√dt × Z)`

   Where:
   - S₀: Initial asset price.
   - r: Risk-free rate.
   - σ: Annualized volatility.
   - dt: Time increment.
   - Z: Standard normally distributed random variable.

   Simulation configuration:
   - Initial asset: AAPL.
   - Initial price: 344.37.
   - Risk-free rate: 4% (assumption).
   - Annualized volatility: 30.5%.
   - Time horizon: 1 year.
   - Time steps: 252.
   - Number of simulated paths: 10,000.

   **Model validation:**
   - Simulated expected terminal price: 359.44.
   - Theoretical expected terminal price: 358.42.
   - Simulated standard deviation of log returns: 0.3051.
   - Theoretical standard deviation: 0.3052.

   The simulated statistics closely match their theoretical GBM counterparts.

3. **European Option Pricing: Monte Carlo vs Black-Scholes**

   Two pricing approaches are implemented and compared.

   - **Black-Scholes:** Analytical pricing for European call and put options.
   - **Monte Carlo:** Simulates terminal asset prices using the risk-neutral GBM distribution, calculates option payoffs, and discounts their average to present value.

   The Monte Carlo implementation supports both call and put payoffs:

   `Call Payoff = max(S_T - K, 0)`

   `Put Payoff = max(K - S_T, 0)`

   The estimator is:

   `Option Price = exp(-rT) × Average(Payoff)`

   Standard errors and 95% confidence intervals are calculated to quantify simulation uncertainty.

   The project also evaluates convergence using simulation counts of 100, 1,000, 10,000, 100,000, and 1,000,000.

   The results demonstrate the expected Monte Carlo convergence rate:

   `Standard Error ∝ 1 / √N`

   Increasing the number of simulations improves estimation accuracy, although the improvement is proportional to the square root of the simulation count.

4. **Variance Reduction: Antithetic Variates**

   Antithetic variates are implemented to reduce Monte Carlo estimation variance.

   Instead of generating independent random shocks, the method pairs each shock `Z` with `-Z`:

   `Payoff_Anti = (Payoff(Z) + Payoff(-Z)) / 2`

   The paired payoffs are averaged before calculating the discounted option price.

   Performance is evaluated over 300 repetitions, comparing the variability of plain Monte Carlo estimates against antithetic estimates.

   The observed variance reduction factor is approximately 1.04x, indicating a relatively modest improvement for this particular option and simulation setup.

5. **Greeks Estimation**

   The project estimates option sensitivities using the finite-difference bump-and-reprice technique.

   - **Delta:** Sensitivity of the option price to changes in the underlying asset price.
   - **Vega:** Sensitivity of the option price to changes in volatility.

   Central finite differences are used:

   `Delta ≈ [C(S₀ + h) - C(S₀ - h)] / (2h)`

   `Vega ≈ [C(σ + dσ) - C(σ - dσ)] / (2dσ)`

   Common random numbers are used for both pricing calculations to reduce simulation noise when estimating small price differences.

   The resulting estimates are compared against analytical Black-Scholes Greeks to validate numerical accuracy.

6. **Portfolio Construction and Correlation Analysis**

   A hypothetical portfolio is constructed using AAPL, MSFT, and JPM.

   - Total portfolio value: 100,000.
   - Equal allocation across all three assets.
   - Daily log returns are used as an approximation of daily returns.

   Historical correlations are calculated to examine diversification effects.

   Observed correlations:
   - AAPL–MSFT: 0.64
   - AAPL–JPM: 0.41
   - MSFT–JPM: 0.39

   JPM exhibits the lowest pairwise correlations with the two technology stocks in this sample, providing greater diversification potential within this portfolio.

7. **Value at Risk (VaR) and Expected Shortfall (ES)**

   Two risk estimation approaches are implemented:

   **Parametric Monte Carlo VaR and ES**
   - Estimates the historical mean vector and covariance matrix of asset log returns.
   - Generates 500,000 correlated multivariate normal return scenarios.
   - Calculates portfolio losses using the portfolio weights.
   - Estimates VaR using the selected loss quantile.
   - Calculates Expected Shortfall as the average loss exceeding the VaR threshold.

   **Historical VaR and ES**
   - Uses observed historical portfolio returns directly.
   - Calculates the empirical loss quantile.
   - Estimates Expected Shortfall from losses exceeding the historical VaR threshold.

   The two methods are evaluated at 95% and 99% confidence levels.

   VaR measures a loss threshold at a specified confidence level, while Expected Shortfall measures the average loss in the tail beyond that threshold.

8. **Out-of-Sample Backtesting**

   The final stage evaluates the reliability of the risk estimates using a chronological train/test split.

   - The first 70% of historical returns are used to estimate risk parameters and VaR thresholds.
   - The remaining 30% are reserved for out-of-sample evaluation.
   - Realized test-period losses are compared against the estimated VaR thresholds.
   - Actual exceedance frequencies are compared with theoretically expected exceedance rates.

   The backtest indicates that both models were conservative during the testing period.

   This result also demonstrates why risk estimates should be evaluated against realized outcomes rather than relying exclusively on theoretical distributions.

## Key Findings and Methodological Considerations

- **GBM validation:** Simulated terminal-price statistics closely match theoretical expectations, supporting the correctness of the implemented stochastic simulator.
- **Option pricing accuracy:** Monte Carlo call and put prices closely match Black-Scholes analytical prices.
- **Statistical convergence:** The convergence experiment demonstrates how Monte Carlo estimation uncertainty decreases as the number of simulations increases.
- **Variance reduction:** Antithetic variates provide a modest 1.04x variance reduction in the executed experiment.
- **Numerical Greeks:** Delta and Vega estimates closely reproduce their analytical Black-Scholes counterparts.
- **Diversification:** JPM has lower historical correlations with AAPL and MSFT than the two technology stocks have with each other.
- **Tail-risk estimation:** Historical Expected Shortfall is substantially higher than normal-based Monte Carlo Expected Shortfall, particularly at the 99% confidence level.
- **Backtesting:** Both risk estimation methods produce lower-than-expected exceedance frequencies during the out-of-sample period.

## Limitations

- **GBM assumptions:** The model assumes constant volatility and normally distributed Brownian shocks, which cannot fully capture volatility clustering, jumps, and fat-tailed market returns.
- **Historical volatility:** Option pricing uses historical volatility rather than implied volatility extracted from market option prices.
- **Risk-free rate:** The risk-free rate is fixed at 4% rather than dynamically linked to market interest rates.
- **European options only:** The pricing implementation focuses on European call and put options and does not model early exercise.
- **Variance reduction:** Antithetic variates provide only a modest improvement in the tested configuration. A control variate is discussed in the notebook's explanatory text but is not implemented in the executed code.
- **Portfolio return approximation:** Log returns are used as an approximation of simple portfolio returns.
- **Normality assumption in VaR:** The parametric Monte Carlo risk engine uses a multivariate normal distribution, which can underestimate extreme tail losses.
- **Backtesting limitations:** A single chronological split is used rather than a rolling-window framework. The limited number of extreme exceedances makes 99% VaR validation statistically inconclusive.
- **Data dependency:** Historical prices are retrieved from Yahoo Finance. Results may change when the underlying dataset is refreshed.

## Technologies Used

- **Python 3** – Core implementation.
- **NumPy** – Random number generation, GBM simulation, vectorized calculations, multivariate normal sampling, and numerical estimation.
- **Pandas** – Financial time-series processing, return calculations, and portfolio analysis.
- **Matplotlib** – Visualization of simulated price paths, terminal price distributions, return distributions, and Monte Carlo convergence.
- **yfinance** – Retrieval of historical financial market data.
- **Mathematical and Statistical Methods:**
  - Geometric Brownian Motion (GBM)
  - Black-Scholes option pricing
  - Monte Carlo simulation
  - Antithetic variates
  - Finite-difference Greeks
  - Correlation and covariance analysis
  - Value at Risk (VaR)
  - Expected Shortfall (ES)
  - Out-of-sample backtesting

## Reproducibility

A fixed random seed (`42`) is used for reproducible random number generation.

Historical market data is retrieved dynamically through Yahoo Finance, so reproducing the exact numerical outputs requires using the same historical data snapshot and execution order.

To install the required dependencies:

```bash
pip install numpy pandas matplotlib yfinance
```

Run the notebook sequentially from the first cell to reproduce the analysis and results.

## Possible Future Improvements

- Implement control variates and benchmark their effectiveness against antithetic sampling.
- Introduce stochastic volatility models such as Heston.
- Incorporate implied volatility from actual option market data.
- Extend pricing to Asian, barrier, and other path-dependent options.
- Implement rolling-window VaR backtesting.
- Explore Student-t and other fat-tailed distributions for portfolio risk estimation.
- Add stress testing and scenario analysis for extreme market conditions.
