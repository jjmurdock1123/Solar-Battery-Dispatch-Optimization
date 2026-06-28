# Data files

Processed/intermediate inputs used by the notebooks (see top-level README for the full
pipeline). Each file below is produced by the notebook noted in parentheses.

| File | Produced by | Used by |
|---|---|---|
| `sdge_2024_lmp.csv` | `02_caiso_processing.ipynb` | `06_battery_optimization.ipynb` |
| `weather_hourly_avg.csv` | `03_weather_processing.ipynb` | `04_irradiance_model_comparison.ipynb` |
| `X_irr.csv`, `y_irr.csv` | `04_irradiance_model_comparison.ipynb` | `05_irradiance_optimal_tree.ipynb` |
| `ghi_preds.csv`, `datetime_predictions_df_final.csv`, `ghi_predictions_new_time.csv` | `05_irradiance_optimal_tree.ipynb` (see note) | `06_battery_optimization.ipynb` |

## GHI prediction file lineage

`ghi_preds.csv`, `datetime_predictions_df_final.csv`, and `ghi_predictions_new_time.csv`
are three successive snapshots of the same Optimal Regression Tree forecast, not three
different models. **`ghi_predictions_new_time.csv` is the canonical one** — it's the
file `06_battery_optimization.ipynb` actually reads.

When `05_irradiance_optimal_tree.ipynb` is rerun, it writes its raw output to
`ghi_oct_predictions.csv` / `ghi_oct_actuals.csv` (not currently present here, since
that requires a licensed IAI install). Turning that raw output into
`ghi_predictions_new_time.csv` — renaming it and realigning timestamps to the 7-day
dispatch horizon used in notebook 06 — was done by hand between runs and isn't captured
in any notebook cell. If you rerun the ORT model, you'll need to redo that alignment
step yourself before notebook 06 will pick up the new forecast.
