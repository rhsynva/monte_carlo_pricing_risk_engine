# Monte Carlo Pricing and Risk Engine

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/<your-username>/monte-carlo-pricing-risk-engine/blob/main/monte_carlo_pricing_risk_engine.ipynb)

I wanted to build Monte Carlo pricing and risk tools and be able to show that they are right, not only that they produce numbers. So every result in this project is checked against something known: option prices against Black-Scholes, Greeks against their closed-form values, put-call parity, and VaR against what actually happened in an out-of-sample test.

Everything is in one notebook: [`monte_carlo_pricing_risk_engine.ipynb`](monte_carlo_pricing_risk_engine.ipynb). All outputs are saved, so you can read it on GitHub without running anything.

## Results at a glance

| Check | Result |
|---|---|
| Monte Carlo call price vs Black-Scholes | 40.51 vs 40.44, Black-Scholes is inside the 95% confidence interval |
| Put-call parity | off by 1.4 standard errors, which is normal sampling noise |
| Delta and Vega vs analytical | 0.5495 vs 0.5492 and 136.32 vs 136.34 |
| Variance reduction (same budget) | antithetic 1.41x, control variate 4.96x |
| 99% Expected Shortfall / VaR | 1.15 under a normal model, 1.50 in the real data |
| Out-of-sample VaR | both methods were too cautious, only 95% Monte Carlo is rejected by a Kupiec test |

## 1. Option pricing

Setup: AAPL, last price S₀ = 344.37, European options with strike K = 361.59 (5% above the price), one year to maturity, risk-free rate 4% (an assumption), volatility 30.5% (AAPL's historical volatility).

| | Black-Scholes | Monte Carlo (1,000,000 paths) | Standard error |
|---|---|---|---|
| Call | 40.4428 | 40.5114 | 0.0728 |
| Put | 43.4863 | 43.4297 | 0.0525 |

The 95% confidence interval for the call is [40.3687, 40.6541], and the Black-Scholes price falls inside it. For put-call parity, Monte Carlo gives C − P = −2.9183 and theory gives −3.0435. The gap of 0.1252 is about 1.4 standard errors (SE 0.0898).

The convergence plot shows the error shrinking like 1/√N: 100 times more paths gives only 10 times less error.

![Monte Carlo convergence to Black-Scholes](images/convergence.png)

## 2. Variance reduction

I compared three estimators with the same budget of 10,000 payoff evaluations, repeated 2,000 times, and looked at how much the estimates spread.

| Method | Std of the estimates | Variance reduction |
|---|---|---|
| Plain Monte Carlo | 0.72030 | 1x |
| Antithetic variates | 0.60570 | 1.41x |
| Control variate | 0.32356 | 4.96x |

Antithetic variates pair each random shock Z with −Z. The control variate uses the terminal price S_T, whose expected value is known exactly (S₀·e^(rT)). A call payoff moves strongly with S_T, so subtracting the part explained by S_T removes most of the noise. That is why it wins by so much here.

My first run used only 300 repetitions and gave 1.04x for antithetic variates. That number was noise, so I repeated the experiment with 2,000 repetitions.

## 3. Greeks

Delta and Vega come from bump-and-reprice with the same random numbers in both prices. With fresh random numbers, the Monte Carlo noise would be larger than the small price change I am trying to measure.

| Greek | Monte Carlo | Black-Scholes |
|---|---|---|
| Delta | 0.5495 | 0.5492 |
| Vega | 136.3247 | 136.3352 |

## 4. Portfolio risk

One-day VaR and Expected Shortfall for 100,000 split equally between AAPL, MSFT and JPM. The Monte Carlo version draws 500,000 correlated normal scenarios from the sample mean and covariance. The historical version uses the real daily returns directly.

| Confidence | Method | VaR | Expected Shortfall |
|---|---|---|---|
| 95% | Monte Carlo | 2,381 | 3,010 |
| 95% | Historical | 2,218 | 3,517 |
| 99% | Monte Carlo | 3,400 | 3,915 |
| 99% | Historical | 4,100 | 6,141 |

At 95% the two methods are close. At 99% they are not. For a normal distribution, Expected Shortfall is about 1.15 times VaR at that level, and the Monte Carlo result matches that (3,915 / 3,400). In the historical data the ratio is 1.50. Real bad days are worse than the normal model expects, the same fat tails visible in the AAPL return histogram.

Sample correlations: AAPL-MSFT 0.64, AAPL-JPM 0.41, MSFT-JPM 0.39. JPM is the least correlated with the two tech stocks.

## 5. Backtest

The data is split by time. VaR is estimated on 2019-01-03 to 2024-06-24 (1,377 days) and tested on 2024-06-25 to 2026-10-30 (591 days). A correct 95% VaR should be exceeded on about 5% of days and a 99% VaR on about 1%. I also ran a Kupiec test, which asks whether the observed rate is statistically different from the expected one.

| Confidence | Method | Exceedances | Kupiec p-value |
|---|---|---|---|
| 95% | Monte Carlo | 13 / 591 (2.2%) | < 0.001 |
| 95% | Historical | 20 / 591 (3.4%) | 0.056 |
| 99% | Monte Carlo | 3 / 591 (0.5%) | 0.184 |
| 99% | Historical | 2 / 591 (0.3%) | 0.061 |

Both models were too cautious in the test period, and the test rejects only the 95% Monte Carlo VaR. A likely reason is that the training window contains the 2020 crash, which makes the estimated risk higher than what the calmer test period needed. I have not tested that explanation. With only 2 or 3 exceedances at 99%, those tests have little power, so I do not draw a strong conclusion from them.

## How it works

- **Simulation:** geometric Brownian motion under the risk-neutral measure, `S(t+dt) = S(t) · exp((r − σ²/2)·dt + σ·√dt·Z)`. As a check, the simulated mean terminal price is 359.44 against 358.42 in theory, and the standard deviation of log returns is 0.3051 against 0.3052.
- **Pricing:** discounted average payoff, `price = e^(−rT) · mean(payoff)`, with a standard error and confidence interval.
- **Control variate:** `y = x − b·(S_T − S₀e^(rT))`, with `b = Cov(x, S_T) / Var(S_T)`.
- **Risk:** VaR is a loss quantile and Expected Shortfall is the average loss beyond it.

## Limitations

- GBM assumes constant volatility and normal shocks, so it misses volatility clustering and fat tails.
- Volatility is historical, not implied from market option prices, and the 4% risk-free rate is fixed.
- Only European options on one stock and one strike are priced.
- The Monte Carlo VaR assumes normal returns, and daily log returns stand in for simple returns in the portfolio.
- The backtest uses a single split rather than a rolling window, and the test period is short.
- Prices come from Yahoo Finance and can be revised, so a fresh run may differ slightly from the saved outputs.
- Pricing, simulation and risk code is written with NumPy and SciPy, with no pricing library. The correlated normal draws use NumPy's `multivariate_normal`.

## Run it

```
pip install -r requirements.txt
```

Open the notebook and run it from the first cell to the last. The random seed is fixed at 42.

```
.
├── monte_carlo_pricing_risk_engine.ipynb
├── images/
│   └── convergence.png
├── requirements.txt
└── README.md
```

## How this was built

I ran the notebook end to end, checked each result against theory, and edited the write-up so the numbers match the outputs.
