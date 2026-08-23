# Forecasting NSW Emergency Department Demand

Monthly forecasting and COVID impact analysis on 9 years of NSW emergency department
presentations. Built to answer two questions a health service actually has to answer:
how many people will turn up next month, and how many did COVID keep away.

---

## The short version

- Adjusting for month length changed which months are seasonally quiet. February looked
  like the annual low in raw counts; it isn't. The real quiet season is **May–July**.
- **Seasonality is weakening.** Peak-to-trough fell from 9.5% of the level in 2016 to
  6.9% in 2019.
- Holt-Winters beat both baselines in **every** backtest fold — average MASE **0.57**
  against ~1.2 for the baselines.
- COVID cost NSW EDs roughly **655,000 presentations** between February 2020 and June
  2023. The gap has not closed; it has widened.
- Forecast demand for FY2023-24 implies **1,148–1,304 nurse-shifts per day**, a
  seasonal swing of ~156 shifts between July and December.

---

## The problem

Emergency departments roster staff months ahead. Under-roster and patients wait.
Over-roster and money is spent that could have gone elsewhere. Both errors are
expensive, and the second one is invisible, which makes it worse.

Two questions:

1. **Forecasting** — can next month's presentations be predicted well enough to plan
   rosters against?
2. **Counterfactual** — how many presentations did COVID remove? Predict what should
   have happened, then measure the gap against what did.

---

## Data

| | |
|---|---|
| Source | HealthStats NSW — Emergency department presentations, unplanned (monthly) |
| Publisher | Centre for Epidemiology and Evidence, NSW Ministry of Health |
| Coverage | July 2014 – June 2023, 108 monthly observations |
| Scope | NSW, all public hospital EDs, Persons (males and females combined) |
| Measure | Number (count, not rate per 100,000) |
| Accessed | 21 August 2026 |

**What one row means.** In July 2014, there were 202,927 unplanned presentations to NSW
public hospital emergency departments.

**What it counts.** Presentations, not patients — one person attending three times
counts three times. And presentations, not admissions: arriving at ED is what's
recorded, regardless of whether the patient was later admitted or sent home. Both
consume ED resource, which is why presentations are the right unit for staffing.

---

## Cleaning and validation

1. Dropped one trailing blank row (109 → 108)
2. `Number` arrived as text because of thousands separators — stripped and cast to int.
   Verified against the published chart: first value 202,927 ✓
3. `Period` cast to datetime, set as index
4. Dropped `Sex` — single value, no information
5. Set frequency explicitly to month-start (`MS`)

Step 5 is not cosmetic. Without an explicit frequency, statsmodels quietly declines to
model seasonality — no error, just wrong results.

**Checks passed:** 108 rows before and after setting frequency (no missing months), zero
nulls, 108 unique periods (no duplicates), index monotonic.

---

## The month-length problem

February has 28 days. January has 31. A raw monthly count therefore embeds a ~10%
calendar artefact with no clinical meaning.

Fix: divide each month's total by that month's day count (leap years handled
programmatically) to give **average daily presentations**. Row count stays at 108 —
"per day" here means the monthly average per day, not daily data.

**This changed the answer.** Lowest month per year, clean years only:

| Year | Lowest (raw count) | Lowest (per day) |
|---|---|---|
| 2015 | February | July |
| 2016 | June | June |
| 2017 | February | June |
| 2018 | February | June |
| 2019 | February | May |
| 2022 | February | February |

February was the raw low in 5 of 6 years and vanished almost entirely after adjustment.
It was never quiet — it was short.

Busiest months were unaffected, because December and August both have 31 days. One
adjustment removed a false pattern and confirmed a real one.

---

## Seasonal structure

STL decomposition on pre-COVID daily-average data (Jul 2014 – Feb 2020, period 12).

- **December: +320/day** — the clear annual peak
- **July: −280/day** — the trough
- **Peak-to-trough ~600/day**, about 7% of the level
- August is seasonally **neutral**

An earlier reading of the raw chart suggested two peaks (December and August). The
decomposition disproved it — August topped 2015 and 2017 because of what happened in
those particular years, not because of a repeating pattern.

**Validation:** seasonal components sum to −23.5 (≈0, as additive decomposition
requires), and trend + seasonal + residual reconstructs December 2019 exactly at 8,600.

### Seasonality is weakening

Peak-to-trough range, as a percentage of that year's trend level:

| Year | Amplitude |
|---|---|
| 2015 | 9.27% |
| 2016 | 9.51% |
| 2017 | 8.89% |
| 2018 | 7.48% |
| 2019 | 6.87% |

Monotonic decline after 2016 — a 27% fall in three years. In absolute terms the swing
fell from ~700/day to ~563/day while the level rose 19%.

**ED demand is growing, but the growth is spread across the year rather than
concentrated in peak months. The load profile is flattening.**

This is also why the model was specified as **additive**, not multiplicative:
multiplicative seasonality assumes the swing grows with the level. Here it does the
opposite. The specification was chosen from this evidence, not from test-set
performance.

---

## Modelling

**Model:** Holt-Winters exponential smoothing — additive trend, additive seasonality,
12 periods.

**Baselines:**
- *Seasonal naive* — this month equals the same month last year
- *12-month moving average* — a flat forecast at the mean of the last 12 training months

**Metrics:** MAE, MAPE, and MASE (test MAE ÷ the training set's seasonal-naive MAE).
MASE below 1 means the model beats a naive forecast; above 1 means it doesn't.

### A mistake worth documenting

The first evaluation used a single split and showed the flat moving-average baseline
beating Holt-Winters — MASE 0.36 against 0.45.

That baseline was leaking. It fed test actuals back into its rolling history, and a
scoping error assigned only the final iteration's value across all 12 forecast months —
which happened to be the average of the test period itself. It was reading the answer.

Rolling-origin backtesting caught it. The clue was that a flat line achieving MAE 98
against a series with ±320/day seasonal swing is not arithmetically possible.

**The lesson: a single split can produce a confident, wrong conclusion.**

### Backtest results

Three expanding-window folds, 12-month horizon each, training on 31 / 43 / 55 months.

| Fold (test year) | Holt-Winters | Moving Average | Seasonal Naive |
|---|---|---|---|
| 2017 | **0.64** | 1.15 | 1.10 |
| 2018 | **0.63** | 0.80 | 0.81 |
| 2019 | **0.45** | 1.64 | 1.64 |
| **Average MASE** | **0.57** | 1.20 | 1.18 |

Holt-Winters wins every fold. MAE ranged 121–180 presentations/day, roughly 1.5–2.4%.

Both baselines score above 1.0 on average — neither beats a naive forecast once
evaluated honestly.

---

## COVID counterfactual

Holt-Winters retrained on all 67 pre-COVID months, then forecast forward 41 months
(February 2020 – June 2023). The forecast is what should have happened. The gap against
actuals is the COVID effect.

**Cumulative shortfall: ~654,750 presentations** — about 525 fewer per day, every day,
for three and a half years.

| Period | Gap |
|---|---|
| 2020 (11 months) | −224,491 |
| 2021 | −149,438 |
| 2022 | −182,939 |
| 2023 (6 months) | −97,883 |

Worst month: **April 2020, −29.6%** — the first statewide lockdown.

**The gap is not closing.** 2023 is a half-year; annualised it comes to ~−196,000,
larger than 2022 and larger than 2021. The initial shock was three months long. What
followed was a permanent level shift.

### Is it real, or is it the model drifting?

A three-year-old trend extrapolated forward will overstate a gap if growth has actually
slowed. Testing this: fitting a linear trend to actuals over January 2022 – June 2023
gives **+20.0 presentations/day per month**, against a pre-COVID trend of ~23.

Growth resumed at broadly the pre-COVID rate — from a permanently lower base. The
counterfactual holds.

*(Caveat: that slope is fitted on 18 noisy points. "Broadly similar" is defensible;
"87% of the pre-COVID rate" would not be.)*

---

## Staffing and cost translation

Model retrained on all 108 months — this forecast is for planning, so it needs to see
the post-COVID level. Forecast horizon July 2023 – June 2024, with simulation-based 95%
prediction intervals.

**Assumptions** (all indicative, not a costing):

| | |
|---|---|
| Nurse throughput | 7 presentations per 8-hour shift |
| RN base rate | $47/hour |
| On-costs | 30% (superannuation, leave, penalty loadings) |
| Effective shift cost | $488.80 |

Throughput derivation: a 1:3 ratio with ~3-hour length of stay gives a theoretical 8 per
shift, discounted to 7 for handover, documentation and breaks.

**Results:**

| | |
|---|---|
| Annual ED nursing labour cost | **$221.1M** (95% CI $187.5M – $255.0M) |
| Nurse-shifts per day | 1,148 (July) to 1,304 (December) |
| Seasonal staffing swing | **+156 shifts/day**, December vs July |
| COVID shortfall, valued | $88.0M |

**On that last number:** this is *avoided nursing workload valued at labour cost*. It is
not a saving. Staff were rostered regardless. Reading it as $88M returned to the budget
would be wrong.

---

## Limitations

Stated plainly, because they matter more than the headline numbers.

1. **Reporting scope not verified.** Not every NSW facility has reported ED activity
   across the whole period, and the reporting set has changed. A facility joining the
   collection produces a step up that looks identical to real demand growth. This has
   not been checked and could affect the trend estimate.

2. **41-month forecast horizon.** Model accuracy was validated at 12 months
   (MASE 0.57). The counterfactual extends well past that. The further out it runs, the
   weaker it gets.

3. **Structural break.** Holt-Winters has no mechanism for a level shift — it adjusts
   gradually. The staffing forecast, trained on data spanning the break, may carry a
   small upward bias.

4. **Cost figures are nursing labour only.** Medical, diagnostic, and overhead costs are
   excluded. Total system cost is substantially higher.

5. **STL smoothing left at defaults.** Residuals showed a few large spikes — notably
   mid-2017, likely the record influenza season — which suggests the seasonal component
   may be absorbing year-specific noise.

6. **Presentations, not patients or admissions.** Forecasts inform ED staffing, not
   inpatient bed capacity.

---

## Repository

```
01_raw/          source data, untouched
02_notebook/     analysis
03_output/       exported tables and charts
README.md
```

**Stack:** Python, pandas, statsmodels, plotly.

Power BI was not used — the analysis environment is macOS-only, where Power BI Desktop
does not run. Visualisation is Plotly, exported to interactive HTML.

---

## What I'd do next

- **SARIMA as a second model class** — Holt-Winters alone is thin for a forecasting
  comparison
- **Local Health District breakdown** — 15 series instead of one, which would answer
  whether the COVID impact was even across districts
- **Verify the reporting-scope question** against NSW Health collection documentation
- **Tune the STL seasonal parameter** rather than accepting defaults
