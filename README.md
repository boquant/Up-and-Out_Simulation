# Up-and-Out Barrier Breach Simulation using Heston Model

## 1. Data Collection

### `get_data(symbol, frequency)`

Retrieve minute-level or daily high and close prices for the given symbol.

- `frequency`: one of `['1m', '5m', '15m', '60m', '1d']`
- `high`: used to detect barrier breaches.
- `close`: used to estimate instantaneous volatility.

---

## 2. Instantaneous Volatility Estimation

### `instantaneous_volatility(close_prices)`

Estimate a proxy for instantaneous volatility using:

- **Realized Volatility**: defined as \( r^2 \), more accurate under high-frequency data.
- **ARCH/GARCH** models: used to remove noise and model volatility clustering.

#### Model Selection Logic:

1. Identify the minute with the highest realized volatility.
2. Use data *after* that minute to fit an ARCH or GARCH model.
3. Apply the **Ljung–Box test** on realized volatility for autocorrelation.
   - If **no significant autocorrelation**: use realized volatility as proxy.
   - Otherwise:
     - Fit ARCH/GARCH.
     - Plot ACF and Ljung–Box p-values (10 lags).
     - Use BIC to determine best fit.

Visualize both model fit and test results. State which volatility proxy is selected.

---

## 3. Heston Model Likelihood Fitting

### Heston Model Parameters:

- $\mu$: Drift of the stock
- $\kappa$: Speed of mean reversion
- $\theta$: Long-run variance
- $\sigma$: Volatility of volatility
- $\rho$: Correlation between stock and variance
- $v_0$: Initial variance

### `log_likelihood(params, r, vt, interval)`

- Transform the Heston model into log returns.
- Assume bivariate normality between asset returns and volatility.
- Compute the 2D Gaussian log-likelihood for MLE.

### `MLE_fit(r, vt, interval)`

- Define parameter bounds.
- Fit Heston parameters using maximum likelihood on return \( r \) and volatility \( v_t \).
- `interval`: number of time intervals per day (e.g., 390 for 1-minute bars).

---

## 4. Monte Carlo Simulation for Barrier Breach

### `barrier_breach_monte_carlo(params, s0, v0, v_min, barrier, interval, steps, n)`

Simulate barrier breach using Heston paths:

- `params`: fitted Heston parameters
- `s0`, `v0`: initial stock price and variance
- `barrier`: barrier price level
- `interval`: number of intervals per day
- `steps`: time steps per path
- `n`: number of Monte Carlo trials

#### Brownian Bridge Adjustment:

If a path does **not** breach the barrier at discrete points, compute the probability of breach **within the interval** using Brownian bridge:

```math
P_{\text{bridge}} = \exp\left( -\frac{2 ( \log B - \log S_t )( \log B - \log S_{t+1} ) }{ \sigma^2 \Delta t } \right)
```
Total probability of breach per path:

```math
P_{\text{total}} = 1 - \prod_t (1 - P_{\text{bridge}, t})
```

Final breach probability is estimated by:

```math
\hat{P} = \frac{\text{breach count} + \text{cumulative bridge breach}}{n}
```

---

### 5. Barrier Breach Probability Testing

### `barrier_breach_prob_test(symbol, frequency, adjustments, n)`

Evaluate model accuracy by comparing simulation with actual historical test data:

- `adjustments`: list of floats. Each defines a barrier above the historical max:  
  $\text{barrier} = \max(\text{training high}) + \text{adjustment} $
- Last 390 minutes (1 day) used as test data; earlier data used for training.
- For each barrier, compare:  
  $\text{Benchmark breach prob} = \frac{\text{num} \text{ of test minutes where high > barrier}}{390}$

**Plot:**
- Simulated Monte Carlo breach probability vs. benchmark.

---
<p align="center">
  <img src="https://github.com/user-attachments/assets/d2dbc90d-07a6-45d6-9ee4-56032d91755c" width="45%" />
  <img src="https://github.com/user-attachments/assets/7bf64c8d-716f-4a43-bbab-9397aa32c8db" width="45%" />

</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/d2eb45ad-deaf-484b-9f62-468f80f60b70" width="45%" />
  <img src="https://github.com/user-attachments/assets/d2df49eb-7879-4d91-a227-921607dffe40" width="45%" />
</p>

**Observation:**  
Model may **underestimate breach probability** in cases with sudden surges (not captured in training),  
but generally **tracks benchmark well** and converges quickly as actual breaches diminish.

**Future Plan:** 
Evaluate performance across a broader set of stocks to assess generalizability.

