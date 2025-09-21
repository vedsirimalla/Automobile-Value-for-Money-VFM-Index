Automotive Value-for-Money (VFM) Index— From Messy Specs to Pricing Insight

TL;DR: I built an end-to-end pipeline that turns raw car specs into a Value-for-Money (VFM) score and a % over/under-priced signal per variant, using peer-group standardization, a transparent performance composite, and robust price modeling. It’s explainable, resilient to weight changes, and ready for dashboards.

Why this project?

In a crowded auto market, buyers and dealers ask the same question:

“Am I getting the best performance for the price I pay?”

Specs (hp, torque, 0–100, top speed) and price are usually shown separately. This project answers with a single, defensible metric: a VFM Index that quantifies performance-per-dollar and flags over/under-pricing vs. true peers.

What the system delivers

VFM score per variant (higher = more performance per dollar, within peer)

% Over/Under-priced vs. peer expectation given performance

Fair peer comparisons using Fuel × Seats × Engine-size bands

Explainable weights & robust stats; stability validated via Sensitivity & Ablation tests

Exportable CSV + ready-to-plot diagnostics; simple to wire into Power BI or Streamlit

Data (final schema)
Company Names, Car Names, Engines, CC/Battery Capacity, HorsePower,
Total Speed, Performance(0 - 100 )KM/H, Cars Prices, Fuel Types, Seats, Torque, variant_index

Cleaning & normalization highlights

Split multi-engine rows into variants (variant_index)

Parse units & ranges → clean numerics (e.g., “680 hp”, “2.9 sec”, “$14k–$16k”)

Convert liters to cc (e.g., 2.8L → 2800 cc)

Seats: "2+2" → 4; ranges (4–7) → max (7)

Fuel canonicalization (7 buckets): Petrol, Diesel, Hybrid, PHEV, CNG, Electric, Hydrogen with rules (e.g., “plug” → PHEV, “Gas/Gasoline” → Petrol, “Hybrid” in engine → Hybrid, “Electric Motor” alone → Electric, “Bi-Fuel” → CNG)

Deduplication and type checks for modeling reliability

Peer groups (fair apples-to-apples)
peer_id = Fuel | SeatsBand | EngineBand


Fuel: {Petrol, Diesel, Hybrid, PHEV, CNG, Electric, Hydrogen}

SeatsBand: {2, 4–5, 6–7, 8+}

EngineBand (ICE): small <1500 cc, mid 1500–3000 cc, large >3000 cc

Electric/Hydrogen: use labels EV / Hydrogen (cc not meaningful)

All standardization and pricing fits happen within a peer to ensure fairness.

Features & robust scaling

We use winsorization (1%/99%) and robust z-scores (median/IQR) within each peer to control outliers and skew.

Performance inputs

HorsePower (hp)

Torque (tq)

Top speed (vmax)

Acceleration 0–100 km/h in seconds (acc_sec) → we invert (lower is better)

Efficiency proxy for ICE: −cc (smaller is better). For EV/H2, efficiency is set to 0 by design.

Price input

Cars Prices in USD (ranges → midpoint), transformed via log for modeling stability

Performance composite (transparent weights)
perf = 0.35*z_hp + 0.20*z_tq + 0.20*z_vmax + 0.20*z_acc + 0.05*z_eff(ICE only)


EV/Hydrogen rows use z_eff = 0 (cc not meaningful); we don’t renormalize (sum = 0.95)

Rationale: hp carries most signal; torque & acceleration matter strongly; top speed is informative but capped in many peers; cc is a light nudge toward efficiency for ICE.

VFM scoring (performance per dollar)

Two complementary forms (use either; both are computed):

Ratio:
price_index = exp(z_price)
VFM_ratio = perf / price_index → higher is better

Difference:
VFM_diff = perf − z_price

Over/Under-priced % (peer price model)

Within each peer, fit:

log(price) ≈ intercept + slope * perf


Fit robustly (Huber) when possible; else OLS (winsorized inputs)

Residual in log space: price_resid = log_price_actual − log_price_pred

Convert to %:

overpriced_% = (exp(price_resid) − 1) × 100


Positive = over-priced; negative = under-priced
(Small peers <3 models → skip fit; % shows 0 as “not estimated”)

Example insight (same peer)

Peer: Diesel | 6–7 | mid

Model	perf	Actual	Expected	Residual (log)	Overpriced %
MAHINDRA XUV500	0.125	$18.4k	~$33.4k	−0.596	−44.9%
Ford Everest	0.781	$52.0k	~$44.1k	+0.166	+18.1%

Everest performs better but is priced even higher than expected → ~18% overpriced

XUV500 is modestly above-median on perf but priced far below expected → ~45% under-priced

A peer scatter (log-price vs perf) makes this immediately visible to pricing teams.

Sensitivity & Ablation results (robustness)

What we tested

Sensitivity: ±10% weight tweaks (one feature at a time) with renormalization; compare rankings vs. baseline using Spearman, Top-20 overlap/Jaccard, and avg |Δ rank|.

Ablation: drop one feature (set weight to 0), redistribute the remainder, re-rank, and compute the same stability metrics.

Key takeaways

Rankings are very stable. Across ±10% tweaks, Spearman correlations stay high and Top-20 overlap is near complete → leaders don’t reshuffle under reasonable weight changes.

Importance order aligns with domain logic:
HP removal causes the most churn, then Acceleration/Torque, then Top speed (moderate), and cc efficiency has the smallest impact (as expected with a 0.05 weight, 0 for EV/H2).

Implication: the VFM index is explainable and resilient—safe for pricing, product, and marketing decisions.

Tools & technologies

Python 3.x

Pandas, NumPy — data wrangling, grouping, winsorization, robust z

scikit-learn — HuberRegressor (robust price fit); fallback OLS via numpy.polyfit

Matplotlib — EDA & diagnostic charts

Regex parsing — unit/range normalization

Jupyter/Colab — exploration & reproducibility

(Optional) Power BI / Streamlit — dashboards for VFM leaderboards, peer scatter, and price residuals
