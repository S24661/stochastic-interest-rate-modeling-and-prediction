# Stochastic Interest Rate Modelling — CIR Model

Implementation and extension of the **Cox-Ingersoll-Ross (CIR)** model for yield curve reconstruction from a single observable short rate.

---

## What This Project Does

Given only the **3-Month yield** for any day, this project reconstructs the entire yield curve (out to 2 years on the test set) using a calibrated stochastic interest rate model. It implements the base CIR model, calibrates it against ~8 years of historical data, and extends it with a **Two-Factor Longstaff-Schwartz model** that captures both the level and slope of the yield curve independently.

---

## The Model

The CIR short rate follows:

$$dr_t = \kappa(\theta - r_t)\,dt + \sigma\sqrt{r_t}\,dW_t$$

The square-root diffusion keeps rates non-negative (Feller condition: $2\kappa\theta \geq \sigma^2$). From the calibrated parameters, a zero-coupon bond price has a closed form:

$$P(t, T) = A(t,T)\,e^{-B(t,T)\,r_t}$$

which gives the yield at any maturity analytically — no simulation needed at inference time.

---

## Data

| Split | Period | Rows | Maturities |
|-------|--------|------|------------|
| Training | 2016-05-19 → 2024-04-26 | 1,976 days | 3M, 6M, 9M, 1Y, 2Y, 5Y, 10Y, 20Y, 30Y |
| Test | 2024-04-29 → 2026-04-29 | 495 days | 3M, 6M, 9M, 1Y, 2Y |

Raw data was noisy — cleaning involved IQR Winsorisation (1%–99%) followed by time-based linear interpolation, with forward/backward fill for edge gaps.

---

## Calibration

Two methods were implemented and compared:

- **MLE (time-series)** — fits the transition density of the CIR process (non-central chi-squared) to the 3M rate time series. OLS provides warm-start initial parameters.
- **Cross-sectional least squares** — directly minimises yield errors across all maturities on sampled training days, then takes the median across days for robustness. This is the primary method used, since it optimises for yield curve shape rather than short-rate dynamics alone.

---

## Results

### Base CIR (out-of-sample, 3M rate as sole input)

| Tenor | R² | RMSE |
|-------|----|------|
| 0.25Y | 0.9982 | 0.00036 |
| 0.50Y | 0.9961 | 0.00049 |
| 0.75Y | 0.9791 | 0.00104 |
| 1.00Y | 0.9409 | 0.00160 |
| 2.00Y | 0.6068 | 0.00293 |
| **Overall** | **0.9502** | **0.00159** |

The 2Y tenor is the weak point — a single factor cannot independently model both level and slope.

### Two-Factor CIR Extension (Longstaff-Schwartz 1992)

A second stochastic factor $y_t$ drives the slope. Its value is estimated from the 3M rate via kernel regression trained on historical data.

| Tenor | R² | R² improvement |
|-------|----|----------------|
| 0.25Y | 1.0000 | +0.0018 |
| 0.50Y | 0.9874 | −0.0087 |
| 0.75Y | 0.9659 | −0.0132 |
| 1.00Y | 0.9430 | +0.0021 |
| 2.00Y | 0.8478 | **+0.2410** |
| **Overall** | **0.9670** | **+0.0168** |
| **Overall RMSE** | **0.00129** | **−19%** |

The two-factor model resolves the 2Y problem and cuts overall RMSE by 19%.

---

## Limitations

**Base CIR**
- Single factor — cannot independently model yield curve level and slope
- Constant parameters drift with macro regimes; periodic recalibration is needed
- No jump component — misses sudden central bank hikes (e.g., 75bp Fed moves in 2022)
- Cannot perfectly fit an arbitrary initial term structure (3 parameters, 9 maturities)

**Two-Factor CIR**
- The second factor $y_t$ is unobserved; estimated via kernel regression from 3M, which degrades under regime shifts
- 6 parameters make calibration harder and more sensitive to initialisation
- Kernel weights need recalibration as new data accumulates

---

## Stack

```
numpy  pandas  scipy  matplotlib  sklearn
```

No external interest-rate libraries — all bond pricing math is implemented from scratch.

---

## Files

```
Stochastic_IR_modelling.ipynb   # full notebook: data → calibration → prediction → extension → analysis
train_data.csv               # historical yield data (9 maturities)
test_data.csv                # held-out yields (5 maturities)
test_data_3M.csv             # 3M rate only — the sole allowed input at inference
```
