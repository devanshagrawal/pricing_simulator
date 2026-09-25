# ILine Pricing Simulator

Interactive pricing tool + written analysis for ILine's 3-wheeler pricing (Noida), built from real completed-order data.

## Contents
- **index.html** — the interactive dashboard (open in any browser). Three tabs:
  - **Data** — the written analysis (earnings identity, decision rule, discount layer, competitor comparison).
  - **Simulator** — forward model: set the tariff (base + ₹/km), discounts and evening surge → see AOV, gross/net ₹/day, and position vs competitors.
  - **Target** — reverse model: set target net ₹/day + OPD + discount mix + surge → it solves the base + ₹/km tariff (move one lever, the other adjusts to hold the target), with a live competitor table and chart.
- **pricing_analysis.md** — the methodology and findings (Strategy B: price to a daily earnings target).

## Data basis
Real completed 3-wheeler orders in Noida (`iline_riderequests_v2`), last 15 days; competitor fares from the 31 Aug dummy-route scrape (Porter / Delhivery / Uncle Delivery). Comparisons are like-for-like at the scrape's average distance per bucket.

_Built with Claude Code._
