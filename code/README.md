# Code

## Data (not in git)

Put the datasets under `data/` at the repository root (the folder is git-ignored):

```text
data/
├── m5-forecasting-accuracy/            # Kaggle: M5 Forecasting - Accuracy
│   ├── calendar.csv
│   ├── sales_train_evaluation.csv
│   └── sell_prices.csv
├── Vn1 forcasting/                     # VN1 Forecasting – Accuracy Challenge (Vandeput, 2024)
│   ├── Phase 0 - Sales.csv, Phase 0 - Price.csv
│   ├── Phase 1 - Sales.csv, Phase 1 - Price.csv
│   └── Phase 2 - Sales.csv
└── vietnamsaleandinventory/            # Kaggle: tienanh2003/sales-and-inventory-snapshot-data  (independent footwear panel)
    └── InventoryAndSale_snapshot_data/
        ├── Sales_snapshot_data/
        ├── Inventory_snapshot_data/
        └── MasterData/
```

## Scripts

| Script | Output | Purpose |
|---|---|---|
| `profile_datasets.py` | `outputs/data_profile.md`, `data/cache/*.parquet` | Size, sparsity and ADI–CV² classes of M5, VN1 and the Vietnam footwear dataset (builds `vn_sales.parquet` for the footwear panel); inventory vs sales check |
| `liquidation_breakeven.py` | `outputs/<DATASET>/{breakeven,series_inventory}.csv` | RQ3: break-even salvage ratio of liquidation (upper/lower bound) over a holding-rate × markup grid; series-level weeks of supply |
| `stat_tests.py` | `outputs/stat_tests.md`, `outputs/<DATASET>/stat_tests.csv` | Per-series Friedman–Nemenyi and Wilcoxon–Holm tests (SQL, fill rate, inventory) |
| `per_quantile_loss.py` | `outputs/per_quantile_loss.csv` | Scaled pinball loss per quantile and demand class (lgb_quantile vs tsb_nb), both test windows |
| `analyze_results.py` | `outputs/comparison.md`, `outputs/fig_tradeoff_<D>.png` | Cross-dataset tables, inventory–fill-rate trade-off, ranking consistency |
| `run_pipeline.py` | `outputs/<DATASET>/{classes,forecast_metrics,kpi}.csv`, `summary.md` | Full benchmark: panel → ADI–CV² → forecasts → Decision Engine → lost-sales simulation → KPIs |
| `ensemble_ablation.py` | `outputs/ensemble/` | Seven-method quantile combinations (mean E7, median E7-med), leave-one-out ablation, all 127 mean subsets, subset chosen on one window and applied to the next |
| `censoring_check.py` | `outputs/censoring/censoring_check.md` | Suspected stock-out weeks from sales only (zero runs with (1 − p)^k < α) |
| `censoring_rank_check.py`, `censoring_rank_check_e7.py` | `outputs/censoring/censoring_rank_check*.md` | Rankings on series without flagged weeks and with imputed demand; C2 and the seven-method combinations |
| `censoring_sensitivity.py` | `outputs/censoring/censoring_sensitivity.md` | Detector threshold (α 0.001–0.1) × imputation rule (half mean, mean, mean positive sales) |
| `robust_common.py` | — | Shared loading and simulation helpers for the three scripts below |
| `prospective_tau.py` | `outputs/robustness/prospective_tau.md` | τ chosen on the previous window to hit a target fill, applied unchanged to the next |
| `bootstrap_robust.py` | `outputs/robustness/bootstrap_robust.md` | Series, cluster (items, stores, warehouses, clients) and week-block bootstraps of the paper's interval claims |
| `rolling_calibration.py` | `outputs/robustness/rolling_calibration.md`, `vnf_validation.md` | Weekly past-only τ calibration (target vs achieved fill, inventory, cluster intervals; paper Table 7) and the independent footwear validation |
| `full_policy_bootstrap.py` | `outputs/robustness/full_policy_bootstrap.md` | Table 7 intervals: clusters resampled, weekly τ path and inventory recomputed in each of 2,000 paired replicates |
| `service_tradeoff.py` | `outputs/robustness/service_tradeoff.md` | Target attainment, fill comparability at 0.25 / 0.5 / 1.0 points and inventory verdicts |
| `ranking_reversal.py` | `outputs/robustness/ranking_reversal.md` | TSB-NB vs LGB-Q by VN1 warehouse: pre-test predictors, out-of-window prediction, decision value |

## Package `f2d/`

| Module | Pipeline step |
|---|---|
| `config.py` | Paths, quantiles, ADI–CV² cut-offs, split sizes, default scenario |
| `data.py` | 1–3. Weekly panel for M5 (277 full Walmart weeks), VN1 (196 weeks, Phase 0+1+2) and the footwear panel (82 weeks, retail channel, style–colour × chain) |
| `classify.py` | 4. ADI–CV² classes on the training period |
| `features.py` | 5. Lags, rolling stats, calendar/events, price, static attributes, scale |
| `models.py` | 6. Quantiles of D_{L+R} and D_H. Statistical: `empirical`, `ets` (ETS(A,N,N), normal), `tsb` (Poisson), `tsb_nb` (negative binomial). ML: `lgb_tweedie` (+ normal safety stock), `lgb_conformal` (+ split conformal by demand class), `hgb_quantile` (scikit-learn), `lgb_quantile` (main). The N-HiTS baseline is in `code/run_neural.py` |
| `policy.py` | 7–8. Order-up-to + liquidation rules (quantile, fixed weeks of supply, dead stock), multi-period lost-sales simulator, stock-out risk |
| `evaluate.py` | 9. SQL, RMSSE, coverage; fill rate, CSL, inventory and excess in weeks of demand, liquidation share, bootstrap CIs |

Design: last 26 weeks are the test period (2 blocks of 13 weeks); every model is refitted at each block cutoff
using only targets observed before it and forecasts every week from the newest data; the first 4 simulated
weeks are warm-up. Forecasts are cached in `data/cache/forecasts/<DATASET>/` — delete a file to recompute it.

```bash
pip install -r code/requirements.txt
```

```bash
python code/profile_datasets.py
```

```bash
python -u code/run_pipeline.py --dataset VN1 --boot 200
```

```bash
python -u code/run_pipeline.py --dataset M5 --boot 200
```

Quick check on one M5 store (a few minutes): `--stores CA_1 --tag smoke`. Scenario grid for RQ4: `--grid full`; τ grid only: `--grid tau`.
Earlier test window (robustness): `--offset 26` (outputs and forecasts go to `<DATASET>_w26`), then `stat_tests.py --offset 26` and `analyze_results.py --tag w26`.
Full reproduction steps for every reported result: see the top-level `README.md`.
