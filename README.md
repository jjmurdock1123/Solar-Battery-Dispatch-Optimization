# Predicting Solar Output to Prescribe Battery Dispatch Decisions

**Robust Optimization for the San Diego Electricity Market**

Jacob Lebovitz, Justine Murdock /n

15.095: Machine Learning Under a Modern Optimization Lens, MIT

Full writeup: [`report/Machine_Learning_Final_Report.pdf`](report/Machine_Learning_Final_Report.pdf)

## Overview

San Diego's solar-heavy grid produces sharp midday price drops and evening demand
peaks in the CAISO day-ahead market. This project builds a predict-then-optimize
pipeline for a solar-paired battery in the SDG&E zone:

1. **Forecast** next-day solar Global Horizontal Irradiance (GHI) from NSRDB weather
   data using several ML models, including an Optimal Regression Tree (ORT).
2. **Dispatch** the battery over a rolling 7-day / hourly horizon to maximize revenue
   from CAISO day-ahead prices, subject to battery physics (capacity, charge/discharge
   limits, round-trip efficiency, degradation cost) and a "no grid charging" constraint
   (the battery can only charge from on-site solar).
3. **Robustify** the dispatch against day-ahead price forecast error using ellipsoidal,
   budgeted (Bertsimas–Sim), and box uncertainty sets, and compare against a rule-based
   heuristic baseline.

### Headline result (7-day weekly profit)

| Scenario                       | Profit ($) |
|---------------------------------|-----------:|
| No Battery (sell all solar)     |  2,595.45 |
| Rule-Based Dispatch              |  3,207.59 |
| **Nominal Optimization**         | **3,849.04** |
| Ellipsoidal Robust Optimization  |  3,465.84 |
| Budgeted Robust Optimization     |  3,285.86 |

Robustness trades some expected profit for protection against price forecast error,
but both robust formulations still beat the heuristic and no-storage baselines. See
the report for the full methodology, the December 24 case study, and discussion.

## Repository structure

```
.
├── report/
│   └── Machine_Learning_Final_Report.pdf   final writeup
├── notebooks/                              pipeline, run in this order
│   ├── 01_caiso_download.ipynb             Python — pull CAISO day-ahead LMPs from the OASIS API
│   ├── 02_caiso_processing.ipynb           Python — merge/clean LMPs, filter to the SDG&E node
│   ├── 03_weather_processing.ipynb         Python — average NSRDB grid cells into one hourly series
│   ├── 04_irradiance_model_comparison.ipynb Python — Linear/kNN/Tree/RF/GBM next-day GHI models
│   ├── 05_irradiance_optimal_tree.ipynb    Julia  — Optimal Regression Tree (final GHI model)
│   └── 06_battery_optimization.ipynb       Julia  — deterministic + robust battery dispatch, baseline
├── data/                                   processed inputs/intermediates (small, tracked)
│   ├── sdge_2024_lmp.csv                   cleaned 2024 SDG&E day-ahead LMPs (output of 02)
│   ├── weather_hourly_avg.csv              region-averaged hourly NSRDB weather (output of 03)
│   ├── X_irr.csv / y_irr.csv               feature/target matrices for GHI forecasting (output of 04)
│   └── ghi_preds.csv, datetime_predictions_df_final.csv, ghi_predictions_new_time.csv
│       curated GHI forecast snapshots; ghi_predictions_new_time.csv is the one consumed by notebook 06
│       (notebook 05 writes its raw output to ghi_oct_predictions.csv / ghi_oct_actuals.csv when rerun,
│        not currently present since that requires a licensed IAI install — see caveat below)
└── results/                                 dispatch outputs and figures (output of 06)
    ├── battery_dispatch_7days_results*.csv          hourly charge/discharge/export, per strategy
    ├── battery_dispatch_with_timestamps*.csv         same, with UTC + Pacific timestamps
    ├── battery_dispatch_solar_*uncertainty*.csv      L0/L1 robust-solar variants
    ├── *_operator_prescription.csv                   human-readable "what to do, when" summaries
    ├── Baseline_Plot.png                             rule-based dispatch behavior (Fig. 1 in report)
    └── Nominal_Optimal_Plot.png                       nominal optimization dispatch behavior (Fig. 2)
```

## Data not included in this repo

Three raw/intermediate data sources are excluded via `.gitignore` because they are far
too large for git (GitHub hard-blocks files over 100 MB, and these total ~24 GB). They
are left in place on disk but are not tracked:

- `weather_data/` — raw per-grid-cell hourly NSRDB downloads for the SDG&E region (459 MB, 372 files).
  Source: [NSRDB](https://nsrdb.nrel.gov).
- `Missing_CAISO/` — daily CAISO DAM LMP backfill files (1.6 GB), each over the 100 MB GitHub limit.
- `caiso_dam_lmp_parallel_mit.csv` — the full-year merged raw CAISO download (22 GB).
  Source: [CAISO OASIS](https://www.caiso.com).

To regenerate them: download NSRDB hourly data for the SDG&E grid cells into
`weather_data/`, then run `notebooks/01_caiso_download.ipynb` to pull CAISO day-ahead
LMPs into `caiso_dam_lmp_parallel_mit.csv` / `Missing_CAISO/`. Everything downstream
(`02`–`06`) only needs the small processed files already committed under `data/`.

## Reproducing the pipeline

**Python** (notebooks 01–04): pandas, numpy, requests, tqdm, scikit-learn, seaborn, matplotlib.

**Julia** (notebooks 05–06): CSV, DataFrames, JuMP, LinearAlgebra, MathOptInterface,
Plots, TimeZones, Dates, Statistics, plus two commercially licensed packages —
free academic licenses are available for both:

- **Gurobi** — the LP/SOCP solver used for every optimization model in notebook 06.
- **Interpretable AI (IAI)** — provides `IAI.OptimalTreeRegressor`, the Optimal
  Regression Tree used in notebook 05. Without an IAI license, notebook 05 cannot run,
  but the sklearn model comparison in notebook 04 and the battery optimization in
  notebook 06 are independent of it (06 just needs a GHI forecast CSV in `data/`).

Notebooks assume Jupyter's default working directory (the notebook's own folder), so
run them from inside `notebooks/`; all paths are relative to that location.

### Known caveats

- Notebook 05's `data/ghi_oct_predictions.csv`/`ghi_oct_actuals.csv` output is the raw
  ORT prediction; the curated `data/ghi_predictions_new_time.csv` actually fed into
  notebook 06 reflects manual reformatting (timestamp alignment to the dispatch horizon)
  done between runs and isn't fully scripted.
- Notebook 05's later cells (the `IAI.GridSearch` / time-based split block) depend on
  `X_mat`, which is commented out earlier in the same notebook — uncomment that line
  first if you want to run that alternate split.
