Car Value-for-Money (VFM) Index — End-to-End ML & Analytics

What if you could tell—in one number—whether a car is “worth it” for the price?
This project builds a production-grade pipeline and analytics stack to compute a Value-for-Money (VFM) Index across thousands of car models/variants, enabling apples-to-apples comparisons, pricing simulations, and portfolio gap analysis.

Business impact: Identify under/over-priced models, support pricing moves, guide SKU strategy, and surface consumer value “sweet spots.”

✨ Highlights

VFM Index: Composite score that normalizes performance & efficiency versus price to rank models fairly across fuel types and segments.

Robust data cleaning & normalization: Handles multi-engine rows, unit parsing, ranges (“$12k–$15k”), liter→cc mapping, and fuel inference (Hybrid/EV/Diesel/PHEV) with explicit rules.

Variant-level granularity: Splits rows into per-engine variants for precise comparisons.

Dashboards (optional): Brand/segment drill-downs, VFM leaders/laggards, and a pricing simulator for margin vs. volume trade-offs.

Reproducible pipeline: One command to clean, normalize, and export a modeling-ready dataset.

🧩 Problem & Solution (Business Lens)

Problem: Shoppers, dealers, and OEMs can’t easily judge value for money because specs and prices aren’t standardized across trims, fuel types, and segments.

Solution:
A VFM score = (standardized performance + efficiency) per dollar, computed at the variant level, benchmarked within peer segments.
This unlocks:

Pricing actions: Spot ±10–25% under/over-pricing vs. peer medians.

Portfolio strategy: Identify gaps (e.g., missing mid-range EVs with 4 seats & 300–400 hp).

Go-to-market: Position high-VFM models, justify premiums, steer promotions.
