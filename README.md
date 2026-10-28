# Sales Forecasting with Prophet and External Regressors

A notebook-based research project for evaluating weekly cigarette sales forecasts in Southeast Sulawesi, Indonesia, using data from 2022–2025. It compares Prophet models with different combinations of historical sales, Indonesian holidays, provincial minimum wages, and a national tobacco excise policy index.

The main question is whether additional regressors improve forecasting compared with simpler benchmarks. The workflow covers data validation, feature engineering, hyperparameter tuning, chronological evaluation, diagnostics, and exports for reporting and Power BI.

**This repository contains code only.** Source sales data, generated tables and charts, and saved notebook outputs are excluded. Running the notebook requires a compatible local dataset.

## What the project does

1. Loads weekly sales from an Excel workbook and validates the required years, weeks, and duplicate records.
2. Converts the data from wide to long format and creates dates using ISO weeks.
3. Engineers a one-week sales lag, weekly holiday indicators, a normalized provincial minimum wage regressor, and a cumulative national tobacco excise policy index.
4. Tunes six Prophet model families using 18 parameter combinations per family, including alternatives with and without yearly seasonality.
5. Selects a predictive model using validation WMAPE, preferring a simpler model when candidates fall within a 0.10 percentage-point tolerance.
6. Evaluates forecasts against held-out weeks and a naive previous-week-sales benchmark.
7. Runs regressor ablation, correlation, coefficient, residual, and prediction-interval diagnostics.
8. Exports forecast tables, metrics, plots, and a run manifest.

## Models and features

| Model | Regressors beyond Prophet's trend and seasonality | Role |
| --- | --- | --- |
| `prophet_only` | None | Pure Prophet benchmark |
| `lag_only` | Previous week's sales | Historical-sales benchmark |
| `baseline_only` | Holidays and sales lag | Research baseline |
| `ump_only` | Normalized provincial minimum wage, holidays, sales lag | Minimum-wage comparison |
| `cht_policy_only` | Tobacco excise policy index, holidays, sales lag | Policy-index comparison |
| `proposal_only` | Minimum wage, policy index, holidays, sales lag | Combined external-regressor model |
| `proposal_plus_intervention` | Combined regressors plus intervention indicator | Conditional analysis after a known event |
| `intervention_focused` | Holidays, sales lag, intervention indicator | Simpler conditional analysis |
| `naive_lag1` | Forecast equals previous week's sales | Reference benchmark; outside Prophet model selection |

UMP refers to the provincial minimum wage. CHT refers to tobacco excise policy. The CHT feature is a cumulative national policy index with a 2021 baseline of 100, rather than a product-specific tax rate or retail price.

## Evaluation design

The notebook uses chronological splits based on the source year and week:

| Split | Period | Observations in the common analysis sample |
| --- | --- | --- |
| Training | 2022–2024 | 153 |
| Validation | 2025, weeks 1–12 | 11 |
| Test | 2025, weeks 13–52 | 40 |

The expected input has 208 weekly observations: 52 weeks for each of four years. Week 1 of each source year is excluded from the common analysis sample because the notebook only accepts a lag within the same source year. This leaves 204 observations so all model families use the same sample.

Evaluation is **one-step-ahead walk-forward forecasting**. For each week, the model is fitted to the available history, predicts that week, and then receives its observed sales before forecasting the following week. The test period therefore measures sequential weekly forecasts rather than a single forecast for the entire remaining year.

Hyperparameters and the final predictive model are selected using validation data. The test set does not determine the selected model. The primary metric is WMAPE; MAE, RMSE, MAPE, and SMAPE are also reported. With the default configuration, the target uses a `log1p` transformation and predictions are converted back to the original sales units.

## Run locally

Use Python 3.12 as a starting point; the existing local run manifest records Python 3.12.4. The notebook contains compatibility guidance for Prophet/Stan failures.

1. Clone the repository and enter its folder:

   ```bash
   git clone https://github.com/AkhtarAriq/Sales-forecasting-whit-Prophet-and-External-Regressor.git
   cd Sales-forecasting-whit-Prophet-and-External-Regressor
   ```

2. Create a virtual environment:

   ```bash
   python -m venv .venv
   ```

   Activate it on Windows PowerShell:

   ```powershell
   .\.venv\Scripts\Activate.ps1
   ```

   Or on macOS/Linux:

   ```bash
   source .venv/bin/activate
   ```

3. Install the notebook dependencies:

   ```bash
   python -m pip install jupyterlab ipykernel prophet pandas numpy matplotlib holidays openpyxl
   ```

   Dependencies are not locked. The existing local manifest records Prophet 1.3.0, pandas 3.0.1, and NumPy 2.4.3; a fresh environment has not been verified as part of the repository documentation update.

4. Place your authorized input workbook, named `clear format.xlsx`, next to the notebook. Its `Sheet1` worksheet must contain these columns:

   | Column | Required contents |
   | --- | --- |
   | `week` | Integer week numbers 1–52 |
   | `2022`, `2023`, `2024`, `2025` | Numeric weekly sales for each year |

   Each year must contain all 52 weeks. The configured target unit is **thousand sticks**. If your workbook or study differs, review the configuration cell, including the filename, worksheet, years, feature mappings, and intervention dates.

5. Launch JupyterLab:

   ```bash
   python -m jupyterlab
   ```

   Open `Prophet_weekly_sultra_2022_2025_final_revisi.ipynb`, select the environment's Python kernel, and run **Restart Kernel → Run All**. The explanatory text inside the notebook is primarily in Indonesian.

The grid search repeatedly fits Prophet models, including additional ablation models when `RUN_EXTENDED_ABLATION = True`, so a full run can take time. The default `CLEAN_OUTPUT_DIR = True` deletes and recreates the existing `prophet_sultra_outputs` directory beside the input workbook at the start of a run. Copy any results you want to retain before rerunning.

## Generated outputs

Outputs are written to `prophet_sultra_outputs/` beside the input workbook. They are generated locally and are not included in this repository.

| Output | Purpose |
| --- | --- |
| `05_validation_model_comparison.csv` | Compare candidate models on validation data |
| `06_model_selection_summary.csv` | Record the selected predictive model and selection basis |
| `07_test_model_metrics.csv` | Compare held-out test performance |
| `10_selected_predictive_forecast.csv` | Inspect the selected model's weekly predictions and errors |
| `12_test_metrics_by_regime.csv` | Compare performance across normal and intervention periods |
| `13e_ablation_validation_comparison.csv` | Assess contributions from different regressor combinations |
| `13g_selected_model_residual_diagnostics.csv` | Summarize residual behavior and interval coverage |
| `14_powerbi_forecast.csv` | Import dates, actuals, predictions, intervals, and model labels into Power BI |
| `15_powerbi_model_metrics.csv` | Import model-level metrics into Power BI |
| `16_run_manifest.json` | Record configuration, versions, sample counts, and interpretation notes |
| `plot_01_*.png` through `plot_05_*.png` | Visualize splits, validation results, test forecasts, and residuals |

The notebook produces Power BI input files; this repository does not contain a Power BI report or a deployed application.

## Interpretation and limitations

- **Extra regressors are hypotheses to test.** Their inclusion does not establish improved accuracy. Use the validation comparison and ablation results to assess their contribution.
- **Intervention models are conditional/ex-post.** Their indicator uses an identified event period. They are excluded from predictive model selection and do not demonstrate that the model can anticipate an unexpected shock.
- **Annual regressors have limited variation.** Minimum wages and the policy index vary annually over only four source years and can be highly correlated. Their coefficients describe associations and do not establish causal effects.
- **MAPE can become very large when actual sales approach zero.** Interpret it alongside WMAPE and errors in the original units. WMAPE is an error metric, not a percentage accuracy score.
- **Future regressors must be available at forecast time.** The sales lag supports the implemented weekly workflow; longer-horizon forecasting would require a separate approach to unavailable future inputs.
- **Reproducibility requires the local data.** The public code alone cannot reproduce the original numerical results because the sales workbook and generated artifacts are excluded.

## Portfolio framing

Suggested summary:

> Developed a Python research workflow to evaluate weekly sales forecasting with Prophet and external regressors. Built data-quality checks, time-series features, hyperparameter tuning, chronological walk-forward evaluation, and regressor ablation analysis. Generated forecast visualizations, diagnostic tables, and Power BI-ready exports, while separating predictive evaluation from conditional analysis of a known intervention period.

Useful supporting artifacts include a methodology diagram, validation model comparison, actual-versus-predicted plot, residual analysis, and a concise explanation of what the external-regressor experiments showed. A Power BI dashboard can be a separate presentation layer built from the exports.

For a public portfolio, use synthetic data or outputs approved for disclosure. Charts and screenshots can reveal the excluded sales data; do not commit the original output folder just to illustrate the project. Keep validation and test results clearly labeled, and describe observed limitations alongside the findings.

## Data handling

`.gitignore` excludes Excel workbooks, CSV/TSV/Parquet datasets, data directories, generated outputs, notebook checkpoints, virtual environments, local credentials, and local notebook backups. The committed notebook has its saved outputs removed. Before committing a notebook after a local run, clear its outputs again and review the staged files.
