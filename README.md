# Value-for-Money (VFM) Index for Cars — From Messy Specs to Pricing Insight

> **TL;DR**
>
> * End-to-end pipeline to convert raw car specs into a **Value-for-Money (VFM)** score and **% over/under-priced** signal per variant.
> * Uses **peer-group standardization**, a transparent **performance composite**, and **robust price modeling**.
> * **Explainable** and **stable** under weight changes; ready for dashboards and stakeholder decisions.

---

## Why This Project?

* Car specs (hp, torque, 0–100, top speed) and price are usually shown **separately**.
* The question customers/dealers actually care about: **“Am I getting the best performance for the price I pay?”**
* This project computes a **single, defensible VFM metric** and a **peer-based pricing residual** to show **who’s a bargain** and **who’s overpriced**.

---

## What the System Delivers

* **VFM score** per variant (higher = more performance per dollar, within peer).
* **% Over/Under-priced** vs. peer expectation given performance.
* **Fair peer comparisons** using Fuel × Seats × Engine-size bands.
* **Explainable weights & robust stats** (median/IQR, winsorization).
* **Reproducible outputs** (CSV) + plots; easy to wire into **Power BI** or **Streamlit**.

---

## Data (Final Schema)

```
Company Names, Car Names, Engines, CC/Battery Capacity, HorsePower,
Total Speed, Performance(0 - 100 )KM/H, Cars Prices, Fuel Types, Seats, Torque, variant_index
```

### Cleaning & Normalization Highlights

* **Split multi-engine rows** into separate **variants** (`variant_index`).
* **Parse units & ranges → numerics**

  * Examples: “680 hp” → 680; “2.9 sec” → 2.9; “\$14k–\$16k” → midpoint.
* **Liters → cc** (e.g., 2.8L → **2800 cc**) to fix displacement after engine split.
* **Seats**: “2+2” → 4; seat ranges (“4–7”) → **max** (7) to reflect capacity.
* **Fuel canonicalization (7 buckets)**:

  * `Petrol, Diesel, Hybrid, PHEV, CNG, Electric, Hydrogen`
  * Rules: “plug” → **PHEV**; “Gas/Gasoline” → **Petrol**; “Hybrid” in engine → **Hybrid**; “Electric Motor” alone → **Electric**; “Bi-Fuel” → **CNG**.
* **De-duplication** and type checks for modeling reliability.

---

## Peer Groups (Fair Apples-to-Apples)

```
peer_id = Fuel | SeatsBand | EngineBand
```

* **Fuel**: {Petrol, Diesel, Hybrid, PHEV, CNG, Electric, Hydrogen}
* **SeatsBand**: {2, 4–5, 6–7, 8+}
* **EngineBand (ICE)**: small <1500 cc, mid 1500–3000 cc, large >3000 cc
* **Electric/Hydrogen**: use labels `EV` / `Hydrogen` (cc not meaningful)

> All standardization and pricing fits happen **within a peer** to ensure fairness.

---

## Features & Robust Scaling

* **Winsorization**: 1%/99% per peer to reduce outlier influence.
* **Acceleration**: invert seconds → use **−acc\_sec** so **faster = higher**.
* **Efficiency (ICE only)**: use **−cc** (smaller displacement = better proxy).
* **Price**: use **log(price)** for stability and percent interpretation.

**Performance Inputs**

* HorsePower (**hp**), Torque (**tq**), Top speed (**vmax**), Acceleration (**acc\_sec**, inverted), **Efficiency proxy** (−cc for ICE; 0 for EV/H2).

---

## Performance Composite (Transparent Weights)

```
perf = 0.35*z_hp + 0.20*z_tq + 0.20*z_vmax + 0.20*z_acc + 0.05*z_eff(ICE only)
```

* **EV/Hydrogen** rows use `z_eff = 0` (cc not meaningful); we **don’t renormalize** (sum = 0.95).
* **Rationale**:

  * hp carries most cross-segment signal;
  * torque & acceleration matter strongly;
  * top speed informative but often capped;
  * cc is a **light** efficiency proxy for ICE.

---

## VFM Scoring (Performance per Dollar)

* **Ratio form**

  * `price_index = exp(z_price)`
  * `VFM_ratio = perf / price_index`  → **higher is better**
* **Difference form**

  * `VFM_diff = perf − z_price`

> Use either in the dashboard; both are computed in outputs.

---

## Over/Under-Priced % (Peer Price Model)

* Fit within each peer:

  * `log(price) ≈ intercept + slope * perf`
  * Robust regression (Huber) when available; else OLS (winsorized inputs).
* Residual (log space):

  * `price_resid = log_price_actual − log_price_pred`
* Convert to percent:

  * `overpriced_% = (exp(price_resid) − 1) × 100`
  * **Positive** = overpriced; **Negative** = underpriced.
  * **Small peers (<3)**: skip fit; show 0% as “not estimated”.

---

## Example Insight (Same Peer)

**Peer**: `Diesel | 6–7 | mid`

| Model               |  perf |  Actual |  Expected | Residual (log) | Overpriced % |
| ------------------- | ----: | ------: | --------: | -------------: | -----------: |
| **MAHINDRA XUV500** | 0.125 | \$18.4k | \~\$33.4k |         −0.596 |   **−44.9%** |
| **Ford Everest**    | 0.781 | \$52.0k | \~\$44.1k |         +0.166 |   **+18.1%** |

* **Everest** performs better but is priced **even higher than expected** → \~**18% overpriced**.
* **XUV500** is modestly above-median on perf but **priced far below expected** → \~**45% under-priced**.
* A **peer scatter** (log-price vs `perf`) makes this obvious to pricing teams.

---

## Sensitivity & Ablation Results (Robustness)

### What We Tested

* **Sensitivity**: ±10% weight tweaks (one feature at a time) with renormalization; recompute VFM and compare to baseline via:

  * **Spearman rank correlation** (global rank stability)
  * **Top-20 overlap & Jaccard** (leaderboard stability)
  * **Avg |Δ rank|** within Top-20
* **Ablation**: drop one feature (set weight to 0), redistribute the remainder, re-rank, compute same metrics.

### Key Takeaways

* **Rankings are very stable** under ±10% tweaks.

  * High **Spearman**, near-complete **Top-20 overlap**, minimal **Top-20 churn**.
* **Importance order matches domain logic**:

  * **Drop hp** → biggest churn;
  * **Drop acceleration/torque** → noticeable churn (segment-dependent);
  * **Drop top speed** → small–moderate churn;
  * **Drop efficiency (cc)** → least change.

### Implication

* The VFM index is **explainable and resilient**—safe for **pricing**, **product**, and **marketing** decisions.
* Optional **segment-aware tuning**:

  * Diesel/7-seater peers → +5–10% **torque**;
  * Sporty petrol peers → +5–10% **acceleration**;
  * Keep **cc weight = 0.05** for ICE (until real economy metrics are added), and **0** for EV/H2.

---

## Tools & Technologies

* **Python 3.x**
* **Pandas, NumPy** — data wrangling, grouping, winsorization, robust z
* **scikit-learn** — `HuberRegressor` for robust fits (fallback OLS via `numpy.polyfit`)
* **Matplotlib** — EDA & diagnostics
* **Regex parsing** — unit/range normalization
* **Jupyter/Colab** — exploration & reproducibility
* *(Optional)* **Power BI** / **Streamlit** — dashboards for VFM, peer scatter, price residuals

---

## Interpreting the Outputs (Cheat Sheet)

* **perf**: peer-standardized performance (higher = better).
* **z\_price / price\_index**: how pricey vs peer (higher = more expensive).
* **VFM\_ratio / VFM\_diff**: performance adjusted for price (higher = better value).
* **overpriced\_%**: price vs peer expectation at your performance.

  * Positive → **over-priced**
  * Negative → **under-priced**

---

## Business Impact

* **Pricing**: flag trims ±10–25% vs peers; adjust list prices or run targeted promotions.
* **Product**: identify **portfolio gaps** (e.g., no high-VFM mid-range EV with 4–5 seats).
* **Marketing**: honest, data-backed claims (“best hp-per-dollar in segment”).
* **Sales**: peer scatter & residuals for transparent, persuasive conversations.

---

## Next Steps

* Add **real efficiency** metrics (mpg, L/100km, kWh/100km) for all fuels.
* Learn **segment-specific weights** from price/sales (constrained regression) or regularize.
* Regionalize with **market effects** (taxes, incentives, brand equity).
* Attach **confidence intervals** to over/under-price estimates (bootstrap).
* **Time-series** tracking of VFM and residuals.
