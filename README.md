# PCA-Based Statistical Arbitrage in European Equities

This repository contains a Python implementation of a PCA-based statistical
arbitrage strategy applied to Euro Stoxx 50 equities.

The methodology is inspired by the PCA approach developed by Avellaneda and Lee
for equity statistical arbitrage.

The project combines:

- rolling Principal Component Analysis;
- statistical factor modelling;
- residual return extraction;
- AR(1) estimation;
- Ornstein-Uhlenbeck mean-reversion modelling;
- modified s-score trading signals;
- long-short portfolio construction;
- transaction-cost analysis;
- parameter sensitivity tests;
- optional volume-based signal refinement.

The strategy is rebalanced daily and uses total return indices denominated in
EUR.

## Project Overview

The objective is to construct a market-relative equity strategy that identifies
temporary deviations of individual stock returns from a small set of common
statistical factors.

The workflow is:

1. estimate the main equity-market factors using PCA;
2. regress each stock return on the selected factors;
3. construct cumulative residual processes;
4. estimate their mean-reversion properties;
5. select sufficiently fast mean-reverting stocks;
6. convert residual deviations into trading signals;
7. construct an equal-weighted long-short portfolio;
8. evaluate the strategy before and after transaction costs;
9. test the robustness of the main modelling choices.

## Investment Universe

The investment universe consists of Euro Stoxx 50 constituents.

The project uses:

- total return indices in EUR;
- simple daily returns;
- daily signal generation;
- daily position updates;
- no risk-free rate.

The course-provided datasets are:

```text
sx5e_underlyings.csv
ticker_details.csv
volume.csv
```

The baseline strategy uses price data and ticker information.

Volume data are used only in the optional extension.

## Data Preparation

Simple daily returns are calculated from total return indices.

```text
return_t =
    total_return_index_t
    / total_return_index_t_minus_1
    - 1
```

Before constructing the strategy, the data are:

- sorted chronologically;
- aligned across securities;
- checked for missing observations;
- restricted to stocks with sufficient historical information;
- matched with ticker and sector metadata.

All information used at a trading date must be available before the corresponding
position is opened.

## Avoiding Look-Ahead Bias

Signals are computed using data available at the close of day `t`.

Positions based on those signals are opened or closed at the close of the
following trading day.

The timing convention is therefore:

```text
signal computed at close of day t

trade executed at close of day t + 1

position return starts after execution
```

This lag is essential to avoid using information before it could have been
implemented in practice.

## Rolling PCA Framework

At each trading date, PCA is performed using the previous 252 trading days.

```text
pca_lookback = 252 trading days
```

The baseline strategy retains the first four principal components.

```text
number_of_factors = 4
```

PCA is applied to the sample correlation matrix rather than directly to the
covariance matrix.

Using the correlation matrix standardizes the stocks and prevents the
highest-volatility securities from dominating factor extraction only because of
their scale.

## Return Standardization

Within each estimation window, stock returns are standardized before applying
PCA.

A generic standardization is:

```text
standardized_return_i_t =
    (
        return_i_t
        - mean_return_i
    )
    / volatility_i
```

PCA is then applied to the correlation matrix of the standardized returns.

The standardization parameters are estimated only from the current rolling
window.

## Principal Component Analysis

The sample correlation matrix is decomposed into eigenvalues and eigenvectors.

```text
correlation_matrix
    * eigenvector_j
    =
eigenvalue_j
    * eigenvector_j
```

The eigenvectors define portfolios representing common statistical factors.

The eigenvalues measure the variance associated with each factor.

The components are ordered from the largest eigenvalue to the smallest.

## Explained Variance

The proportion of variance explained by component `j` is:

```text
explained_variance_ratio_j =
    eigenvalue_j
    / sum(all_eigenvalues)
```

For the baseline strategy, the first four components are retained regardless of
the percentage of variance that they explain.

The sensitivity analysis later replaces the fixed number of factors with a
time-varying number selected through explained-variance targets.

## Eigenportfolio Returns

The PCA loadings are used to construct factor-mimicking portfolios.

A generic factor return can be represented as:

```text
factor_return_j_t =
    sum(
        factor_weight_i_j
        * stock_return_i_t
    )
```

When PCA is performed on standardized returns, the eigenvector loadings must be
translated consistently into exposures to the original return series.

The implementation should document the exact normalization used for the
eigenportfolio weights.

## Statistical Factor Model

For each stock, returns are modelled as a linear combination of the selected PCA
factors plus an idiosyncratic residual.

```text
stock_return_i_t =
    intercept_i
    + beta_i_1 * factor_1_t
    + beta_i_2 * factor_2_t
    + beta_i_3 * factor_3_t
    + beta_i_4 * factor_4_t
    + residual_i_t
```

The residual represents the component of the stock return that is not explained
by the common PCA factors.

These residuals form the basis of the mean-reversion strategy.

## Residual Process

The daily regression residuals are accumulated through time to construct a
residual price-like process.

```text
cumulative_residual_i_t =
    cumulative_residual_i_t_minus_1
    + residual_i_t
```

The strategy assumes that sufficiently stationary cumulative residuals fluctuate
around a long-run equilibrium level.

Temporary deviations from this level create potential long and short signals.

## Ornstein-Uhlenbeck Interpretation

Each cumulative residual process is interpreted through a univariate
Ornstein-Uhlenbeck model.

The continuous-time intuition is:

```text
change_in_residual =
    mean_reversion_speed
    * (
        long_run_mean
        - current_residual
    )
    * time_step
    + volatility
    * random_shock
```

The process tends to return toward its long-run mean.

The strength of this tendency is determined by the mean-reversion speed.

## AR(1) Estimation

The OU parameters are estimated through a discrete AR(1) representation using a
60-day rolling window.

```text
ou_lookback = 60 trading days
```

The discrete model is:

```text
residual_process_t_plus_1 =
    ar_intercept
    + ar_coefficient
      * residual_process_t
    + error_t_plus_1
```

The assignment specifies that the residual means must not be de-meaned before
estimating the AR(1) model.

The intercept is therefore retained and contributes to the estimated equilibrium
level and modified s-score.

## AR(1) Validity Checks

A mean-reverting AR(1) process generally requires:

```text
0 < ar_coefficient < 1
```

Values close to one indicate slow mean reversion.

Values well below one indicate faster mean reversion.

Invalid or economically implausible estimates should be excluded from signal
generation.

Possible exclusion criteria include:

- non-positive AR coefficient;
- AR coefficient greater than or equal to one;
- insufficient observations;
- near-zero residual variance;
- unstable numerical estimates.

## Mean-Reversion Speed

The discrete AR coefficient is converted into an annualized mean-reversion speed.

A generic conversion is:

```text
mean_reversion_speed =
    -log(ar_coefficient)
    / time_step
```

For daily observations:

```text
time_step approximately equals 1 / 252
```

The exact annualization convention should match the implementation used for the
assignment.

## Stock Selection

Only stocks with sufficiently fast estimated mean reversion are eligible for
trading.

The selection rule is:

```text
mean_reversion_speed > 8.4
```

Stocks that fail this condition receive no active signal.

This filter excludes residual processes that revert too slowly to support the
short-horizon statistical arbitrage strategy.

## Interpretation of the Speed Threshold

A high mean-reversion speed implies a short characteristic reversion time.

A generic mean-reversion time measure is:

```text
mean_reversion_time =
    1 / mean_reversion_speed
```

A half-life measure can be calculated as:

```text
half_life =
    log(2)
    / mean_reversion_speed
```

The speed threshold therefore restricts the strategy to residual deviations
expected to normalize relatively quickly.

## Equilibrium Volatility

The fitted AR(1) model is used to estimate the long-run volatility of the
residual process.

A generic discrete-time expression is:

```text
equilibrium_variance =
    innovation_variance
    / (
        1 - ar_coefficient^2
    )
```

The equilibrium standard deviation is:

```text
equilibrium_volatility =
    sqrt(equilibrium_variance)
```

This volatility is used to normalize the distance of the current residual from
its estimated equilibrium level.

## Modified S-Score

The trading signal is based on a modified s-score.

The score measures how far the current cumulative residual lies from its
estimated equilibrium, in units of equilibrium volatility.

Unlike a simplified zero-drift score, the modified score includes the drift or
intercept estimated from the AR(1) model.

A generic representation is:

```text
modified_s_score =
    drift_adjusted_residual_deviation
    / equilibrium_volatility
```

The exact drift adjustment follows the implementation prescribed by the
assignment and the reference methodology.

Large positive values indicate that the residual is high relative to its
estimated equilibrium.

Large negative values indicate that the residual is low relative to its
estimated equilibrium.

## Baseline Trading Rules

The baseline strategy uses symmetric opening and closing thresholds.

### Open a Long Position

```text
modified_s_score < -1.25
```

A strongly negative residual is interpreted as temporarily undervalued relative
to the common factors.

### Open a Short Position

```text
modified_s_score > 1.25
```

A strongly positive residual is interpreted as temporarily overvalued relative
to the common factors.

### Close a Long Position

```text
modified_s_score > -0.50
```

The long position is closed when the residual has moved sufficiently close to
equilibrium.

### Close a Short Position

```text
modified_s_score < 0.50
```

The short position is closed when the residual has moved sufficiently close to
equilibrium.

## Position Persistence

Positions remain open until their respective closing conditions are satisfied.

Signals should therefore be implemented through a stateful position-management
process.

For each stock, the state may be:

```text
-1 = short position
 0 = no position
+1 = long position
```

The next position depends on:

- the current position;
- the latest modified s-score;
- the opening threshold;
- the closing threshold.

## Position Update Logic

A representative position rule is:

```text
if current_position == 0:
    if score < negative_open_threshold:
        next_position = 1
    elif score > positive_open_threshold:
        next_position = -1
    else:
        next_position = 0

elif current_position == 1:
    if score > negative_close_threshold:
        next_position = 0
    else:
        next_position = 1

elif current_position == -1:
    if score < positive_close_threshold:
        next_position = 0
    else:
        next_position = -1
```

The position generated on day `t` is executed on the following trading day.

## Portfolio Construction

Active signals are converted into equal-weighted positions.

The target gross exposure is 100%.

```text
gross_exposure =
    sum(abs(portfolio_weights))
    = 1
```

If there are `N_active` active positions:

```text
absolute_weight_per_position =
    1 / N_active
```

Long signals receive positive weights.

Short signals receive negative weights.

If no stocks are active, the strategy remains in cash with zero risky exposure.

## Net Exposure

Equal weighting of active positions does not guarantee market neutrality.

The net exposure is:

```text
net_exposure =
    sum(portfolio_weights)
```

Depending on the number of long and short signals, the strategy may have:

- positive net exposure;
- negative net exposure;
- approximately zero net exposure.

The project should report both gross and net exposure through time.

## Factor Exposure

Although signals are generated from regression residuals, equal weighting of the
selected stocks does not automatically guarantee zero exposure to the PCA
factors.

A useful diagnostic is:

```text
factor_exposure =
    factor_loading_matrix_transpose
    * portfolio_weights
```

This helps determine whether the resulting strategy remains approximately
factor-neutral or develops unintended systematic exposures.

## Daily Portfolio Returns

The gross strategy return is computed using the lagged portfolio weights.

```text
gross_portfolio_return_t =
    sum(
        weight_i_t_minus_1
        * stock_return_i_t
    )
```

The weights must correspond to positions that were actually executable before
the return was realized.

## Transaction Costs

The strategy is evaluated under two scenarios:

1. no transaction costs;
2. all-in transaction costs of 5 basis points.

The cost rate is:

```text
transaction_cost_rate = 0.0005
```

Turnover is calculated from changes in portfolio weights.

A simple implementation is:

```text
turnover_t =
    sum(
        abs(
            target_weight_i_t
            - previous_weight_i_t
        )
    )
```

Daily transaction costs are:

```text
transaction_cost_t =
    transaction_cost_rate
    * turnover_t
```

The net strategy return is:

```text
net_portfolio_return_t =
    gross_portfolio_return_t
    - transaction_cost_t
```

## Drifted Weights

A more precise turnover calculation compares new target weights with the
portfolio weights after market movements but before rebalancing.

The pre-trade drifted weight is approximately:

```text
drifted_weight_i_t =
    previous_weight_i
    * (
        1 + asset_return_i_t
    )
    / (
        1 + previous_portfolio_return_t
    )
```

Turnover is then:

```text
turnover_t =
    sum(
        abs(
            new_target_weight_i_t
            - drifted_weight_i_t
        )
    )
```

The README and code should state which turnover convention is used.

## Performance Metrics

The strategy is evaluated using:

- cumulative performance;
- annualized return;
- annualized volatility;
- Sharpe ratio;
- maximum drawdown;
- turnover;
- gross exposure;
- net exposure;
- number of active positions;
- long and short counts;
- average holding period;
- percentage of profitable days;
- transaction-cost impact.

## Cumulative Performance

```text
cumulative_wealth_t =
    product(
        1 + strategy_return_s
        for all s up to t
    )
```

Separate cumulative wealth series are calculated for:

- gross returns;
- net returns after transaction costs.

## Annualized Return

A geometric annualized return can be calculated as:

```text
annualized_return =
    final_wealth
    ** (
        252 / number_of_trading_days
    )
    - 1
```

## Annualized Volatility

```text
annualized_volatility =
    standard_deviation(daily_strategy_returns)
    * sqrt(252)
```

## Sharpe Ratio

Since the risk-free rate is ignored:

```text
sharpe_ratio =
    mean(daily_strategy_returns)
    / standard_deviation(daily_strategy_returns)
    * sqrt(252)
```

## Drawdown

```text
running_peak_t =
    maximum cumulative wealth observed up to t

drawdown_t =
    cumulative_wealth_t
    / running_peak_t
    - 1
```

The maximum drawdown is the minimum value of the drawdown series.

## Turnover Analysis

Statistical arbitrage strategies may generate substantial turnover because:

- signals are updated daily;
- positions are closed when residuals revert;
- PCA factors and regression coefficients change through time;
- stocks enter and leave the eligible universe;
- portfolio weights are rescaled when the number of active signals changes.

The comparison between gross and net performance is therefore a central part of
the project.

## Baseline Backtest

The baseline backtest uses:

```text
PCA lookback = 252 trading days

number of PCA factors = 4

OU estimation lookback = 60 trading days

minimum mean-reversion speed = 8.4

opening threshold = 1.25

closing threshold = 0.50

gross exposure target = 100%

transaction cost scenarios = 0 bps and 5 bps
```

## Opening-Threshold Sensitivity

The robustness of the strategy is tested using alternative symmetric opening
thresholds.

```text
opening_threshold = 1.00

opening_threshold = 1.50

opening_threshold = 2.00
```

The closing thresholds remain unchanged unless the implementation explicitly
states otherwise.

## Expected Threshold Effects

A lower opening threshold generally produces:

- more trading signals;
- more active positions;
- higher turnover;
- shorter waiting time between trades;
- weaker average signal strength;
- greater transaction-cost exposure.

A higher opening threshold generally produces:

- fewer trades;
- lower turnover;
- stronger residual deviations at entry;
- more concentrated portfolios;
- potentially longer waiting periods;
- greater estimation uncertainty because of fewer observations.

The sensitivity analysis determines whether performance is driven by one
particular threshold choice.

## Variable Number of PCA Factors

The baseline model always retains four principal components.

However, the variance explained by four components changes through time.

The sensitivity analysis therefore selects the number of factors dynamically.

The target cumulative explained-variance levels are:

```text
40%

55%

65%

75%
```

At each trading date, the minimum number of components required to reach the
selected target is retained.

## Dynamic Factor Selection

For an explained-variance target `target`, the number of factors is:

```text
number_of_factors_t =
    smallest k such that
    cumulative_explained_variance_k >= target
```

The number of factors may therefore vary from one date to another.

This allows the model to adapt to changes in market correlation.

## Market Conditions and Factor Count

During stressed markets:

- correlations often increase;
- the first components explain more variance;
- fewer factors may be required to reach a given target.

During calmer markets:

- idiosyncratic and sector effects become more relevant;
- variance is distributed across more components;
- more factors may be required.

Dynamic factor selection can therefore make the strategy more responsive to
changes in market structure.

## Fixed vs Variable Factor Count

### Fixed Number of Factors

Advantages:

- simple implementation;
- stable model dimension;
- easier comparison across dates;
- lower computational complexity.

Limitations:

- explained variance changes through time;
- may underfit or overfit in different market conditions.

### Variable Number of Factors

Advantages:

- adapts to the current covariance structure;
- maintains a more consistent explained-variance target;
- can respond to changing market integration.

Limitations:

- introduces discontinuities in model dimension;
- may increase signal instability;
- can change residual definitions abruptly;
- may increase turnover.

## Optional Volume-Based Extension

The optional extension incorporates trading volume following the liquidity
refinement discussed in the reference methodology.

Possible uses of volume include:

- excluding illiquid stocks;
- scaling signals by relative volume;
- restricting position sizes;
- ranking signals by liquidity;
- reducing exposure when volume is unusually low.

The exact volume transformation should follow the implementation chosen for the
assignment.

## Relative Volume

A representative relative-volume measure is:

```text
relative_volume_i_t =
    current_volume_i_t
    / rolling_average_volume_i_t
```

A liquidity filter might require:

```text
relative_volume_i_t > minimum_volume_threshold
```

Alternatively, signal strength may be scaled by a bounded function of relative
volume.

## Why Volume May Matter

A residual deviation in an illiquid stock may reflect:

- stale prices;
- temporary market frictions;
- large bid-ask spreads;
- non-synchronous trading;
- limited execution capacity.

Volume information may help distinguish economically meaningful dislocations
from signals that are difficult to trade.

However, volume filters may also reduce diversification and eliminate profitable
signals.

## Statistical Diagnostics

The strategy should include diagnostics for:

- PCA explained variance;
- number of retained factors;
- residual autocorrelation;
- AR coefficients;
- mean-reversion speeds;
- residual equilibrium volatility;
- distribution of modified s-scores;
- number of eligible stocks;
- long and short signal counts;
- holding periods;
- factor exposures;
- gross and net exposure.

## PCA Stability

Because PCA is re-estimated daily, component loadings can change through time.

Useful diagnostics include:

- cosine similarity between consecutive components;
- changes in explained variance;
- changes in the number of retained factors;
- sector composition of factor loadings.

Eigenvector signs are arbitrary and should be aligned before comparing
components through time.

## Residual Diagnostics

The residual processes should be checked for:

- mean reversion;
- stationarity;
- outliers;
- changing volatility;
- sensitivity to the estimation window;
- instability of AR parameters.

A high estimated mean-reversion speed alone does not guarantee that a residual
process is economically stable.

## Backtest Validation

The implementation should include the following checks.

### No Look-Ahead Bias

Signals estimated on day `t` must not affect returns before the execution date.

### Gross Exposure

```text
sum(abs(weights_t))
    approximately equals
1
```

whenever at least one position is active.

### Position Values

```text
position_i_t belongs to {-1, 0, 1}
```

before conversion into portfolio weights.

### AR Parameter Validity

Only valid mean-reverting AR estimates should be used.

### Speed Filter

```text
mean_reversion_speed > 8.4
```

for all stocks eligible to trade.

### Cost Consistency

When transaction costs are positive:

```text
net_return_t
    less than or equal to
gross_return_t
```

before considering any unrelated numerical differences.

### Weight Timing

Returns must be multiplied by lagged executable weights.

### Factor Count

Under dynamic selection, the retained number of components must be the smallest
number reaching the target explained variance.

### Reproducibility

Results should be reproducible using fixed random seeds whenever random
procedures are used.

The baseline PCA and regression workflow itself should be deterministic.

## Common Implementation Risks

Potential sources of error include:

- using future returns in the PCA window;
- applying positions on the same day as signal estimation;
- mixing covariance and correlation PCA normalizations;
- failing to transform eigenvectors into correct factor portfolios;
- de-meaning residual means despite the assignment instruction;
- using an incorrect annualization for mean-reversion speed;
- ignoring the eigenvector sign ambiguity;
- opening new positions before closing existing ones;
- calculating transaction costs from unlagged weights;
- failing to account for disappearing or newly available stocks;
- introducing survivorship bias;
- using current-day volume information before it becomes tradable.

## Suggested Repository Structure

```text
pca-statistical-arbitrage-eurostoxx/
|
|-- README.md
|-- requirements.txt
|
|-- src/
|   |-- data_loader.py
|   |-- returns.py
|   |-- rolling_pca.py
|   |-- eigenportfolios.py
|   |-- factor_regression.py
|   |-- residual_process.py
|   |-- ou_estimation.py
|   |-- s_scores.py
|   |-- signal_generation.py
|   |-- position_management.py
|   |-- portfolio_construction.py
|   |-- transaction_costs.py
|   |-- backtest.py
|   |-- performance_metrics.py
|   |-- sensitivity_analysis.py
|   |-- volume_filter.py
|   `-- validation.py
|
|-- notebooks/
|   `-- statistical_arbitrage_analysis.ipynb
|
|-- scripts/
|   `-- run_backtest.py
|
|-- data/
|   `-- README.md
|
|-- results/
|   |-- baseline_performance.csv
|   |-- daily_positions.csv
|   |-- daily_weights.csv
|   |-- daily_s_scores.csv
|   |-- factor_exposures.csv
|   |-- threshold_sensitivity.csv
|   |-- explained_variance_sensitivity.csv
|   `-- figures/
|       |-- cumulative_performance.png
|       |-- gross_vs_net_performance.png
|       |-- drawdown.png
|       |-- rolling_volatility.png
|       |-- turnover.png
|       |-- active_positions.png
|       |-- gross_and_net_exposure.png
|       |-- pca_explained_variance.png
|       |-- retained_factor_count.png
|       |-- mean_reversion_speed.png
|       |-- s_score_distribution.png
|       |-- threshold_comparison.png
|       `-- variance_target_comparison.png
|
`-- report/
    `-- assignment_report.pdf
```

The file and folder names can be adapted to the structure of the actual Python
implementation.

## Requirements

A representative Python environment may include:

```text
numpy
pandas
scipy
scikit-learn
statsmodels
matplotlib
```

Install the dependencies with:

```bash
pip install -r requirements.txt
```

## Running the Project

A possible execution command is:

```bash
python scripts/run_backtest.py
```

Alternatively, the complete workflow can be executed from:

```text
notebooks/statistical_arbitrage_analysis.ipynb
```

## Main Outputs

The project reports:

- rolling PCA eigenvalues and eigenvectors;
- percentage of variance explained by each factor;
- number of retained factors;
- rolling factor returns;
- stock-specific factor loadings;
- residual return series;
- cumulative residual processes;
- AR(1) coefficients;
- mean-reversion speeds;
- equilibrium residual volatilities;
- modified s-scores;
- long and short signals;
- daily portfolio weights;
- gross and net exposure;
- portfolio turnover;
- transaction costs;
- cumulative gross performance;
- cumulative net performance;
- annualized return and volatility;
- Sharpe ratio;
- maximum drawdown;
- opening-threshold sensitivity results;
- explained-variance target sensitivity results;
- optional volume-filter results.

## Technologies

- Python
- NumPy
- pandas
- SciPy
- scikit-learn
- statsmodels
- Matplotlib
- Principal Component Analysis
- Ornstein-Uhlenbeck processes
- Statistical arbitrage
- Quantitative equity strategies
- Backtesting

## Data

The project uses course-provided Euro Stoxx 50 data.

Expected files include:

```text
sx5e_underlyings.csv
ticker_details.csv
volume.csv
```

Course-provided or proprietary datasets should not be included in a public
repository unless redistribution is explicitly permitted.

When the original datasets cannot be published, the `data` folder should contain
a description of:

- required files;
- expected columns;
- date formats;
- ticker identifiers;
- total return index units;
- sector metadata;
- volume units;
- missing-data conventions.

## Academic Context

This project was developed as part of the Buy Side section of the Financial
Engineering course at Politecnico di Milano.

The repository presents the Python implementation of a PCA-based statistical
arbitrage strategy, including factor extraction, residual mean-reversion
modelling, trading signals, transaction costs, and robustness analysis.
