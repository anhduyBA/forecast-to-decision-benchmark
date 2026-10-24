# Beyond Forecast Accuracy: a forecast-to-decision benchmark for retail replenishment

Code and result files for the paper *Beyond Forecast Accuracy: Comparing Probabilistic Demand Forecasts by the
Inventory They Require across Retail Settings and Periods* (manuscript prepared for ICIIT 2027).

The benchmark evaluates probabilistic demand forecasts by the inventory decisions they produce, and asks whether
inventory savings measured retrospectively survive a deployment-style (prospective) evaluation.

- **Methods**
  - seven probabilistic forecasting methods: EMP, ETS, TSB-NB, LGB-T, LGB-C, HGB-Q, LGB-Q;
  - quantile combinations: C2 (mean of LGB-Q and TSB-NB, chosen after the main analysis, i.e. post hoc) and
    E7 / E7-med (mean / median of all seven methods, specified before they were run), with leave-one-out ablations;
  - additional baselines: N-HiTS and zero-shot Chronos-2.
- **Policy:** one order-up-to policy with lost sales and optional liquidation (lead time 2 weeks, weekly review).
- **Data**
  - M5: brick-and-mortar, 30,490 series, three 26-week test windows;
  - VN1: multi-vendor e-commerce, 15,053 series, three 26-week test windows;
  - an independent footwear panel (Vietnamese retail chain, 909 eligible style–colour series, one 26-week window),
    used only to check the combinations.
- **Evaluation**
  - *retrospective:* inventory needed to reach the same realised fill rate, interpolated along each
    inventory–fill-rate curve, with bootstrap confidence intervals over series;
  - *prospective:* τ chosen on the previous window, or updated every week from the fill observed so far, reporting
    target fill, achieved fill and inventory together (Table 7 of the paper);
  - *resampling:* series, cluster (M5 items and stores, VN1 warehouses and clients, footwear styles) and
    week-block bootstraps.

Everything runs on a laptop CPU.

## Main findings

- On M5, LGB-Q needs 3–11% less inventory than the point-forecast, conformal, TSB-NB and exponential-smoothing
  alternatives in every window; this holds under prospective τ selection and cluster and block bootstraps, and
  TSB-NB stays behind LGB-Q on M5 in every stock-out (censoring) scenario.
- On VN1, the leading method changes between windows, and demand classes or clusters do not pick it in advance.
- The pre-specified median of the seven methods needs 7–8% less VN1 inventory than LGB-Q at matched realised fill,
  but the saving does not survive past-only τ calibration, warehouse-level resampling or the independent footwear
  panel, where LGB-Q again needs the least stock. Retrospective savings are therefore not taken as evidence of a
  better deployable method.

## Repository layout

```text
code/
├── f2d/                 # package: data loading, ADI–CV² classes, features, models, policy simulation, metrics
├── run_pipeline.py      # main benchmark: forecasts → order-up-to simulation → KPIs (one dataset, one window)
├── *.py                 # analysis scripts (one per result; see the table below)
├── requirements.txt
├── README.md            # script and module reference
└── outputs/             # result files reported in the paper (.md tables and .csv)
    ├── ensemble/        # seven-method combinations and ablation
    ├── censoring/       # suspected stock-outs and their effect on the rankings
    └── robustness/      # prospective τ, cluster/block bootstraps, past-only calibration, footwear validation
```

## Data (not included)

The datasets are **not redistributed**. Download them from the original sources and place them under `data/` at the repository root:

```text
data/
├── m5-forecasting-accuracy/      # Kaggle "M5 Forecasting – Accuracy" (accept the competition rules)
│   ├── calendar.csv
│   ├── sales_train_evaluation.csv
│   └── sell_prices.csv
├── Vn1 forcasting/               # "VN1 Forecasting – Accuracy Challenge", DataSource.ai (Vandeput, 2024)
│   ├── Phase 0 - Sales.csv, Phase 0 - Price.csv
│   ├── Phase 1 - Sales.csv, Phase 1 - Price.csv
│   └── Phase 2 - Sales.csv
└── vietnamsaleandinventory/      # Kaggle tienanh2003/sales-and-inventory-snapshot-data (footwear panel)
    └── InventoryAndSale_snapshot_data/
        ├── Sales_snapshot_data/
        ├── Inventory_snapshot_data/
        └── MasterData/
```

Expected sizes of the M5 files (bytes):

| File | Size |
|---|---|
| `calendar.csv` | 103,469 |
| `sales_train_evaluation.csv` | 121,736,518 |
| `sell_prices.csv` | 203,395,785 |

For the footwear panel, first run `python code/profile_datasets.py`, which builds `data/cache/vn_sales.parquet` from
the raw Excel files. The panel uses retail-channel sales aggregated to style–colour series for the whole chain;
returns are excluded and three partial weeks are dropped or merged (see `f2d/data.py:load_vnf`). Re-running the
forecasting pipeline on this file reproduces the cached footwear forecasts exactly.

Forecasts and intermediate panels are cached in `data/cache/`. Per-series files (forecasts, class labels, per-series inventory) are therefore not part of this repository.

## Environment

Python 3.12. Install the dependencies:

```bash
pip install -r code/requirements.txt
```

The N-HiTS baseline additionally needs `neuralforecast==3.3.0`, preferably in a separate virtual environment.

## Reproducing the results

All commands run from the repository root. The test window is selected with `--offset`: 0 = main, 26 = second, 52 = third.

| Paper result | Commands | Output in `code/outputs/` |
|---|---|---|
| Forecasts, simulation and KPIs (all sections) | `python code/run_pipeline.py --dataset M5 --grid tau --boot 200` and the same with `--dataset VN1`; repeat with `--offset 26` and `--offset 52` | `<DATASET>[_wN]/` |
| Footwear panel forecasts | `python code/profile_datasets.py`, then `python code/run_pipeline.py --dataset VNF --models empirical,tsb_nb,ets,lgb_tweedie,lgb_conformal,hgb_quantile,lgb_quantile` | `VNF/` (forecasts in `data/cache/`) |
| Accuracy and statistical tests (Table 2) | `python code/stat_tests.py --models empirical,tsb_nb,ets,lgb_tweedie,lgb_conformal,hgb_quantile,lgb_quantile --out-suffix m7 [--offset N]` and `python code/robust_accuracy.py --models ... --out-suffix m7` | `stat_tests*_m7.md`, `robust_accuracy_m7.md` |
| Inventory at equal fill rate (Table 3, Figure 1) | `python code/equal_fill_ci.py --models <same seven> --out-suffix m7 [--offset N]` and `python code/analyze_results.py [--tag wN] --exclude tsb --out-suffix m7` | `equal_fill_ci*_m7.md`, `comparison*_m7.md` |
| Demand-growth strata (Table 4) | `python code/demand_shift_analysis.py` | `demand_shift.md` |
| Segmentation and group-wise selection (Table 5) | `python code/segment_series.py` then `python code/segment_eval.py` | `segmentation.md`, `segment_eval.md` |
| Growth-prediction rule | `python code/growth_rule.py` | `growth_rule.md` |
| Quantile combination C2 and regret | `python code/combination.py`; learned vs equal weights: `python code/combination_weights.py` | `combination.md`, `combination_weights.md` |
| Seven-method combinations E7 / E7-med, leave-one-out ablation, all 127 subsets, prospective subset rule (Section 5.5) | `python code/ensemble_ablation.py` | `ensemble/ensemble_ablation.md` |
| Liquidation and break-even salvage ratio (Table 6) | `python code/liquidation_breakeven.py --dataset M5` (and `VN1`) | `<DATASET>/breakeven.csv` |
| Dataset comparability checks | `python code/sync_check.py`, `python code/dataset_shift.py`, `python code/matched_comparison.py`; price ablation: `run_pipeline.py --dataset VN1 --no-price`; shared features: `run_pipeline.py --dataset M5 --shared-features` | `sync_check.md`, `dataset_shift.md`, `matched_comparison.md`, `equal_fill_ci*_nop.md`, `equal_fill_ci*_sf.md` |
| Hyper-parameter sensitivity | `python code/hparam_sensitivity.py --datasets VN1,M5` | `hparam_sensitivity.md` |
| Lead-time sensitivity | `python code/run_pipeline.py --dataset <D> --grid full` (main window) or `python code/leadtime_forecasts.py` (other windows), then `python code/leadtime_equal_fill.py [--offsets 0,26,52]` | `leadtime_equal_fill*.md` |
| Cost view | `python code/cost_view.py` | `cost_view.md` |
| N-HiTS baseline | `<python-with-neuralforecast> code/run_neural.py --dataset <D> --offset <N> --h 3`, then `python code/neural_eval.py` | `neural_eval.md` |
| Suspected stock-outs (censored demand) | `python code/censoring_check.py`; effect on the rankings: `python code/censoring_rank_check.py` and `python code/censoring_rank_check_e7.py`; detector threshold × imputation grid: `python code/censoring_sensitivity.py` | `censoring/*.md` |
| τ chosen on the previous window (Section 5.8) | `python code/prospective_tau.py` | `robustness/prospective_tau.md` |
| Cluster and week-block bootstraps (Section 5.8) | `python code/bootstrap_robust.py` | `robustness/bootstrap_robust.md` |
| Past-only weekly τ calibration and footwear validation (Table 7) | `python code/rolling_calibration.py` (after `prospective_tau.py` and the footwear forecasts) | `robustness/rolling_calibration.md`, `robustness/vnf_validation.md` |

The run time depends on the dataset. On an Intel Core i5-12450H with 16 GB RAM, one horizon of LGB-Q takes about 2.5 minutes on VN1 and 21 minutes on M5. `rolling_calibration.py` writes one checkpoint per dataset and transition under `outputs/robustness/rolling_parts/`, so an interrupted run resumes where it stopped.

## Notes

- Random seed 2026 is used throughout; models are refitted at the start of each 13-week block of a test window.
- The paper's main intervals resample series; cluster and week-block bootstraps are reported separately. On VN1,
  warehouse- and client-level resampling widens the intervals considerably.
- C2 was specified after the main analysis, so its results are post hoc. E7 and E7-med, their decision criteria and
  the prospective checks were fixed before they were run; script headers refer to these design notes
  (`07_paper_draft/phase5_workflow.md`), which are kept in the authors' working repository.
- An inventory difference is read as a saving only when the achieved fill rates being compared differ by less than
  0.5 points.
- Liquidation economics use an assumed cost model (holding 10–40% per year, markups of 30–100% on unit cost), not
  observed profits.

## License

The code is released under the MIT License (see `LICENSE`). The datasets remain under their original terms.
