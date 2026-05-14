# 0DTE Heston Calibration Notebook

This project contains a self-contained notebook for fitting Heston-style models to intraday SPX 0DTE option quotes.

Notebook:

- `Heston_0DTE_week_10strikes.ipynb`

The notebook loads one true Monday-Friday week of same-day-expiring SPX options, keeps the closest 10 strikes below and 10 strikes above the underlying at each quote time, calibrates several Heston pricing methods, and produces diagnostics and 3D visualizations.

## Data

The notebook expects local parquet files.

Expected filenames look like:

```text
spx-options-2022-05-16.gzip.parquet
```

Required columns:

- `quote_datetime`
- `expiration`
- `strike`
- `option_type`
- `bid`
- `ask`
- `mid`
- `active_underlying_price`
- `moneyness`

The current notebook uses calls from same-day expiration rows and converts the ITM side into a time-value / synthetic-OTM calibration target.

## Main Ideas

0DTE options are difficult to fit directly with raw call prices because near expiry:

- ITM calls are dominated by intrinsic value.
- tiny quote or spot timestamp errors can look like large model errors.
- bid/ask noise and discrete ticks matter.
- full Heston parameters are weakly identified at a single ultra-short maturity.

To address this, the notebook uses:

- OTM/time-value calibration target,
- bid/ask-band Huber loss,
- spread-normalized diagnostics,
- reduced 0DTE Heston mode near expiry,
- total-variance diagnostics,
- Carr-Madan vs COS pricer validation,
- Broadie-Kaya-style Monte Carlo as its own fitted diagnostic method.

## Models / Pricing Methods

The calibrated Heston parameter vector is:

```text
(v0, kappa, theta, sigma, rho)
```

The notebook includes three fitted methods:

- Carr-Madan Fourier inversion
- COS expansion
- Broadie-Kaya-style Monte Carlo diagnostic

Broadie-Kaya in this notebook is a diagnostic Monte Carlo implementation. It samples the CIR variance endpoint exactly but approximates integrated variance with a trapezoidal bridge, so it should not be interpreted as a production exact Broadie-Kaya implementation.

## Calibration Modes

The notebook supports two Heston calibration regimes:

- Full mode: fits `(v0, kappa, theta, sigma, rho)`.
- Reduced 0DTE mode: fits `(v0, sigma, rho)` and freezes `(kappa, theta)`.

Reduced mode is used when time to expiry is below:

```python
REDUCED_HESTON_TAU_MINUTES = 120.0
```

Final very-short snapshots are filtered by:

```python
CALIBRATION_MIN_TAU_MINUTES = 30.0
```

## Diagnostics

The notebook reports:

- raw RMSE to mid,
- time-value RMSE,
- spread-normalized RMSE,
- bid/ask hit rate,
- total-variance RMSE,
- error by tau bucket,
- error by moneyness bucket,
- parameter paths,
- Carr-Madan vs COS disagreement,
- bid/ask violation rows.

## Visualizations

The notebook creates 3D surfaces for:

- market mid prices,
- market time value,
- fitted prices,
- fitted time value,
- spread-normalized errors,
- raw errors,
- Greek fields.

The 3D surfaces include high-contrast side guides:

- blue/cyan sheet: synthetic put side, `K < spot`,
- red/yellow sheet: call side, `K > spot`,
- black sheet: underlying / ATM divider, `K = spot`.

Greek fields are shown as 3D surfaces over:

```text
strike x quote time x Greek value
```

with a dropdown for:

- delta
- gamma
- vega
- theta
- rho-rate
- vanna
- vomma
- speed
- charm
- colour
- zomma
- ultima

## Running

Open and run:

```text
Heston_0DTE_week_10strikes.ipynb
```

Run cells in order.

The full notebook can be computationally expensive because it calibrates Carr-Madan, COS, and Monte Carlo methods across all quote times in the selected week. For faster experimentation, reduce:

```python
CARR_MADAN_N
COS_N
BK_CALIBRATION_PATHS
BK_EVALUATION_PATHS
```

## Dependencies

Python packages used by the notebook include:

- numpy
- pandas
- scipy
- plotly
- pyarrow or fastparquet
- IPython / Jupyter

Install example:

```bash
pip install numpy pandas scipy plotly pyarrow jupyter
```

## Notes

This is a research notebook, not a production pricing library. 0DTE SPX options often require jump, local-volatility, or short-time asymptotic corrections beyond vanilla Heston. The included diagnostics are designed to show when Heston is fitting the quoted market and when divergence is likely caused by numerical instability, microstructure noise, or model misspecification.
