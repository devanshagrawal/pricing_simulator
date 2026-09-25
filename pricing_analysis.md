# ILine Pricing Analysis

Working document for deriving ILine's 3-wheeler pricing in Noida.
Scope so far: **Strategy B — price to a daily earnings target.**

---

## Strategy B — Price to a daily earnings target

### Objective
Set the tariff so that an average vehicle clears a target daily earning
(₹1,500/day) at its breakeven order rate — then decide whether pricing should
move up or down.

### The core identity
```
Daily earning / vehicle  =  Real AOV  ×  OPD

Target: ₹1,500/day at breakeven OPD = 3   →   required AOV = ₹500 / order
```
- **AOV** = average order value = average `grossFare` per completed order.
- **OPD** = orders per vehicle per day (breakeven assumption = 3).

### Decision rule
```
Real AOV × OPD  >  Target   →  surplus → room to DECREASE price (be more competitive)
Real AOV × OPD  <  Target   →  shortfall → need to INCREASE price (or add surge)
Real AOV × OPD  ≈  Target   →  already priced at the floor
```

### AOV decomposition
AOV is broken into distance and rate so the lever is explicit:
```
AOV  =  avg trip distance (km)  ×  effective ₹ per km
```
- Effective ₹/km = `AOV / avg_distance`. It is **not** a tariff slab rate — it
  is the blended realised rate, and it falls with distance because the fixed
  base fare is amortised over more kilometres (short trips carry a high ₹/km,
  long trips a low one).
- To lift AOV by a target amount at constant distance, raise the effective rate
  (equivalently, nudge the base fare, which lifts every order equally).

---

### Data source & basis

| Item | Choice |
|---|---|
| Table | `iline_riderequests_v2` (Metabase DB 33) — V2, `is_current_status = true` |
| Population | `cityName = 'Noida'`, `numberOfVehicleType = 3` (3-wheeler) |
| Completed orders | `requestStatus IN ('COMPLETED','ENDED')` — both count as completed |
| Fare | `grossFare` (GMV realised, incl. surge/waiting/add-ons) |
| Distance | `distance` (last-mile / trip km), filtered `0 < distance ≤ 120` to drop bad rows & outliers |
| Window | Last 15 days (rolling) |
| Time basis | **All-day** for the earnings test — a vehicle earns across the whole day, so a per-day target must use all-day AOV, not morning-only |
| Morning / evening split | `HOUR(bookingCreatedAt) < 15` = morning; `≥ 15` = evening. `bookingCreatedAt` is stored as IST, so **no `CONVERT_TZ`** (also avoids the known double-shift bug) |

> **Why all-day, not morning-only:** morning trips are longer and higher-AOV.
> Using morning-only AOV overstates daily earning and can flip the decision
> (morning basis said "cut price"; all-day basis said "slight increase").

---

### Trip-distance buckets
Analysis buckets: `0–5, 5–10, 10–15, 15–20, 20–25, 25–40, 40+` km.
The old single `25+` bucket was split into `25–40` and `40+` because it averaged
46 km and mixed two very different trip types. Per bucket we track: order count,
order share, avg km, Real AOV, effective ₹/km, and the morning/evening split.

> Note: the order distribution and Real AOV come from **real completed orders**.
> The earlier 141-row "KM bucket wise orders" sheet was dummy pick/drop points
> used only to scrape competitor apps — it is **not** a real order distribution
> and must not be used to weight anything.

---

### The evening-surge lever
ILine currently runs **~0% evening surge** — at the same distance bucket,
evening AOV ≈ morning AOV (within ±3%, noise). The lower *overall* evening AOV
is purely a distance-mix effect (evening trips are shorter), not a pricing one.

So the chosen Strategy-B lever is an **evening surge on orders after 3 PM**
(~42% of orders), which lifts all-day AOV without touching morning pricing:
```
new all-day AOV(surge) = [ Σ(morning_orders × morning_AOV)
                         + Σ(evening_orders × evening_AOV × (1 + surge)) ] / total_orders

evening surge to hit target =
    ( Target/OPD × total_orders − morning_contribution ) / evening_contribution₀  − 1
```
Competitors already run **~17% evening surge**, so there is large headroom before
ILine would reach market evening prices.

---

### Worked result (last 15 days, all-day, Noida 3W)
- Real AOV ≈ **₹493/order** → ₹493 × 3 = **₹1,479/day** → **₹21 short** of ₹1,500.
- Decomposition: ₹493 = **13.5 km × ₹36.5/km**; target ₹500 needs **+₹0.5/km**
  (≈ **+₹7 base fare**).
- Because the real distance mix is short/mid-haul heavy, a cost tariff re-solved
  to hit ₹1,500 lands **≈ ILine's current tariff** — i.e. current pricing is
  already at the earnings floor.
- **Evening surge closes the gap on its own:** a **~3.8% surge** (or flat
  **+₹17 per evening order**) → all-day AOV ₹500 → **₹1,500/day** — with no
  change to morning pricing and still well below competitors' evening fares.

### Conclusion
Strategy B does **not** call for a morning price increase. It calls for
introducing a **modest evening surge** (currently zero) on the ~42% of orders
after 3 PM. A conservative 5–10% surge clears ₹1,500/day comfortably while
keeping ILine the cheapest player in the evening.

---

### Discount layer — gross vs net realisation

`grossFare` is what the **driver is paid** (`fareToDriver = grossFare`); the promo
(`promoPrice`) is a discount **ILine funds**, so it does **not** reduce the
vehicle's earning — only ILine's realisation. This means the ₹1,500/day question
has two answers: a **gross** one (driver earning) and a **net** one (ILine
realisation after promo).

Fare waterfall — Noida 3W, last 15 days (avg per order):
```
grossFare  (= fareToDriver)   ₹492    ← driver earning / GMV
− promoPrice                  ₹109    ← ILine-funded discount (~22%, on ~100% of rides)
= tripFare  (customer pays)   ₹383    ← net realisation
```

**Parameters:**
- New-customer share of rides `s_new`; retained share `s_ret = 1 − s_new`
- New-customer discount `d_new`; retained discount `d_ret`
- **Morning discount `d_morn`** — a time-based promo (before 1 PM only)
- New / retained = first-ever completed ride vs repeat (lifetime `ROW_NUMBER` per `passengerId`)

**Discount rule — higher wins, never stacked, morning fenced at 1 PM:**
- A ride created **before 1 PM** takes `max(segment discount, d_morn)`.
- A ride **1 PM onward** takes the segment discount only.
- Only the *discount* fence is at 1 PM — the **evening surge and the AOV split stay at 3 PM**.

**Calculations:**
```
seg_disc     =  s_new · d_new  +  s_ret · d_ret
morn_disc    =  s_new · max(d_new, d_morn)  +  s_ret · max(d_ret, d_morn)
blended disc =  0.365 · morn_disc  +  0.635 · seg_disc     # 36.5% of rides created before 1 PM
Net AOV      =  Gross AOV × (1 − blended disc)
Gross ₹/day  =  Gross AOV × OPD      (driver earning; discount does not reduce it)
Net ₹/day    =  Net AOV × OPD        (ILine realisation)
```

**Time fence (from coupon behaviour).** The live promos are two time-segmented
coupons: a deeper **morning coupon (~24%, concentrated 9 AM–12 PM, peak 11 AM)** and a
shallower **afternoon/evening coupon (~20%, 1 PM–8 PM, peak 4 PM)**. They hand over at
**~1 PM**, so the morning discount is fenced to **rides created before 1 PM = 36.5%**
of rides (vs 58.1% before the 3 PM boundary). ~2.6–3.4 morning-coupon rides/day
currently leak past 1 PM (~₹310–410/day of misapplied morning discount).

**Real values (last 15 days, Noida 3W):**

| Segment | Ride share | Gross AOV | Discount % |
|---|---|---|---|
| New | 5.5% | ₹524 | 21.1% |
| Retained | 94.5% | ₹490 | 22.2% |
| **Blended** | 100% | **₹492** | **~22.1%** |

**Finding.** Gross (driver) earning ≈ **₹1,476/day ≈ target** — the driver is fine.
But the ~22% promo pulls ILine's net realisation to ≈ **₹1,149/day → ₹351 short**.
Because **94.5% of rides are retained customers discounted ~22%**, the biggest net
lever is **trimming the retained discount** (they are already loyal), not
acquisition spend. Illustratively, cutting the retained discount 22% → 5% recovers
≈ **₹237/day** of net realisation with **no impact on driver pay**.

**Two levers, two gaps.** A tiny evening surge closes the small *gross* gap (₹21);
**discount discipline on retained riders** closes the much larger *net* gap (₹346).

---

### Competitor comparison — like-for-like at scrape distance

The dashboard compares ILine's **net** price (post-discount, + surge in evening)
against Porter / Delhivery / Uncle Delivery per distance bucket, benchmarked **vs the
cheapest competitor** and **vs Porter**, following the morning/evening toggle.

**Competitor fares** are bucket means of the 31 Aug scrape (dummy pick→drop routes,
3-wheeler quoted fare):
`avg = mean(competitor 3W quote over routes whose Actual KM ∈ bucket, matching session)`.
Sample is thin — ~5 routes for short/mid buckets, only **2–3 for 20–40 km**, single day.

**Distance basis (the like-for-like fix).** A competitor bucket mean reflects the
*scrape routes'* distances, which differ from ILine's *real* avg distance within a
bucket — negligible for 0–25 km, but large at 40+ (**53.4 km scrape vs 45.0 km real**).
To compare fairly, ILine is priced **at the scrape's avg km** in the comparison table
(not its real avg km), so both sides sit at the identical distance. This corrected the
40+ row from **−24% → −13%** vs the cheapest competitor; other buckets barely moved.

- AOV / earnings still use ILine's **real** avg km — only the *comparison table* uses
  the scrape distance.
- Competitor figures are **quoted** fares (their promos unknown), so ILine-net-vs-their-
  sticker is a conservative read of the true gap.

**Finding.** After the ~22% promo, ILine's net price sits **~10–35% below even the
cheapest branded competitor in every bucket** — the discount doesn't make ILine
competitive, it makes it drastically undercut a market it already led on price.

---

## Target-driven pricing (reverse solver)

The forward model sets the tariff and reads out the earnings. The **reverse** model
does the opposite: **you set the goal, it solves the tariff.**

### Inputs
- **Target NET revenue** per vehicle per day
- **OPD** (orders per vehicle per day)
- **Discount mix** — new-customer share, acquisition (new) %, retention %, morning %
  (before-1 PM fence) — the blended discount is *computed* from these
- **Evening surge** (after 3 PM)

### The chain
```
Target NET AOV      =  Target NET revenue ÷ OPD
blended discount    =  0.365 · [s_new·max(new,morn) + s_ret·max(ret,morn)]      (before 1 PM)
                     +  0.635 · [s_new·new          + s_ret·ret]                (1 PM onward)
Required GROSS AOV   =  Target NET AOV ÷ (1 − blended discount)
```

### The tariff constraint (with surge)
An all-day gross AOV for a linear tariff `fare = base + ₹/km × km`, with evening orders
lifted by the surge, is:
```
gross AOV  =  base · A  +  ₹/km · D
  A  =  1 + surge × (evening orders ÷ total)                 # base-fare multiplier
  D  =  ( morning_dist_sum + (1+surge) × evening_dist_sum ) ÷ total   # surge-adjusted avg km
```
At surge = 0: `A = 1`, `D = 13.5 km`, so `gross AOV = base + ₹/km × 13.5`.

### The linked solve (move one → the other adjusts)
Set `base·A + ₹/km·D = Required GROSS AOV`, then:
```
change base  →  ₹/km = ( Required GROSS AOV − base · A ) ÷ D          (clamped ≥ 0)
change ₹/km  →  base = ( Required GROSS AOV − ₹/km · D ) ÷ A          (clamped ≥ 0)
```
Every solved pair hits the same Target NET AOV but distributes price differently across
distance — so the competitor table/graph (same like-for-like scrape-distance basis)
shows a different competitive position for each.

### Worked result (net ₹1,500/day, OPD 3, 22% blended, no surge)
- Target NET AOV **₹500** → Required GROSS AOV **₹641** (= 500 ÷ 0.78).
- base ₹235 → **₹30.1/km** (vs today's ₹19); base ₹400 → ₹17.9/km — both net ₹1,500.
- At ₹235 + ₹30/km the tariff **crosses above the cheapest competitor at 10–15 km (+7%)
  and 15–20 km (+11%)** — i.e. hitting the net target *ends the deep undercutting*.
- Turning on a 15% evening surge drops the required ₹/km to **₹27.5** (surge does part
  of the work).

**Takeaway.** To net ₹1,500/day you must either **raise the tariff** (steeper base or
₹/km) or **cut the discount** — the reverse tool makes that trade-off, and its
competitive cost, explicit.

---

### Open items
- The **36.5% before-1 PM share** is hard-coded from the current 15 days; re-pull if the
  AM/PM volume mix drifts (or make it a live input).
- Tighten the "evening" window from "after 3 PM" to a true peak band (e.g. 6–9 PM);
  fewer orders would carry the surge, so the surge % to hit target would rise.
- Confirm whether ₹1,500 is **fare revenue (GMV)** or **driver take-home net of
  commission** — if the latter, divide the target by `(1 − commission)` first.
- Competitor scrape is thin (2–3 routes in 20–40 km buckets) and single-day; re-scrape
  more routes across several days, or use medians for long-haul, to firm up the benchmark.
- Strategy A (match the market) to be documented separately.
