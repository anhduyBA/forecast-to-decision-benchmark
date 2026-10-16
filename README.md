# Beyond Forecast Accuracy: a forecast-to-decision benchmark for retail replenishment

Code and result files for the paper *Beyond Forecast Accuracy: Comparing Probabilistic Demand Forecasts by the
Inventory They Require across Retail Settings and Periods* (ICIIT 2027, under review).

The benchmark evaluates probabilistic demand forecasts by the inventory decisions they produce:

- **Methods**
  - seven probabilistic forecasting methods: EMP, ETS, TSB-NB, LGB-T, LGB-C, HGB-Q, LGB-Q;
  - an equal-weight quantile combination (C2);
  - an additional N-HiTS baseline.
- **Policy:** one order-up-to policy with lost sales and optional liquidation.
- **Data:** all eligible series of two public retail datasets:
  - M5: brick-and-mortar, 30,490 series;
  - VN1: multi-vendor e-commerce, 15,053 series.
- **Test windows:** three 26-week windows.
- **Comparison:** methods are compared by the inventory they need to reach the same realised fill rate, with bootstrap confidence intervals over series.

Everything runs on a laptop CPU.

## Repository layout

```text
code/
├── f2d/                 # package: data loading, ADI–CV² classes, features, models, policy simulation, metrics
├── run_pipeline.py      # main benchmark: forecasts → order-up-to simulation → KPIs (one dataset, one window)
├── *.py                 # analysis scripts (one per result; see the table below)
├── requirements.txt
├── README.md            # script and module reference
└── outputs/             # result files reported in the paper (.md tables and .csv)
```

## Data (not included)

The datasets are **not redistributed**. Download them from the original sources and place them under `data/` at the repository root:

```text
data/
├── m5-forecasting-accuracy/      # Kaggle "M5 Forecasting – Accuracy" (accept the competition rules)
│   ├── calendar.csv
│   ├── sales_train_evaluation.csv
│   └── sell_prices.csv
└── Vn1 forcasting/               # "VN1 Forecasting – Accuracy Challenge", DataSource.ai (Vandeput, 2024)
    ├── Phase 0 - Sales.csv, Phase 0 - Price.csv
    ├── Phase 1 - Sales.csv, Phase 1 - Price.csv
    └── Phase 2 - Sales.csv
```

Expected sizes of the M5 files (bytes):

| File | Size |
|---|---|
| `calendar.csv` | 103,469 |
| `sales_train_evaluation.csv` | 121,736,518 |
| `sell_prices.csv` | 203,395,785 |

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
| Accuracy and statistical tests (Table 2) | `python code/stat_tests.py --models empirical,tsb_nb,ets,lgb_tweedie,lgb_conformal,hgb_quantile,lgb_quantile --out-suffix m7 [--offset N]` and `python code/robust_accuracy.py --models ... --out-suffix m7` | `stat_tests*_m7.md`, `robust_accuracy_m7.md` |
| Inventory at equal fill rate (Table 3, Figure 1) | `python code/equal_fill_ci.py --models <same seven> --out-suffix m7 [--offset N]` and `python code/analyze_results.py [--tag wN] --exclude tsb --out-suffix m7` | `equal_fill_ci*_m7.md`, `comparison*_m7.md` |
| Demand-growth strata (Table 4) | `python code/demand_shift_analysis.py` | `demand_shift.md` |
| Segmentation and group-wise selection (Table 5) | `python code/segment_series.py` then `python code/segment_eval.py` | `segmentation.md`, `segment_eval.md` |
| Growth-prediction rule | `python code/growth_rule.py` | `growth_rule.md` |
| Quantile combination C2 and regret | `python code/combination.py`; learned vs equal weights: `python code/combination_weights.py` | `combination.md`, `combination_weights.md` |
| Liquidation and break-even salvage ratio (Table 6) | `python code/liquidation_breakeven.py --dataset M5` (and `VN1`) | `<DATASET>/breakeven.csv` |
| Dataset comparability checks | `python code/sync_check.py`, `python code/dataset_shift.py`, `python code/matched_comparison.py`; price ablation: `run_pipeline.py --dataset VN1 --no-price`; shared features: `run_pipeline.py --dataset M5 --shared-features` | `sync_check.md`, `dataset_shift.md`, `matched_comparison.md`, `equal_fill_ci*_nop.md`, `equal_fill_ci*_sf.md` |
| Hyper-parameter sensitivity | `python code/hparam_sensitivity.py --datasets VN1,M5` | `hparam_sensitivity.md` |
| Lead-time sensitivity | `python code/run_pipeline.py --dataset <D> --grid full` (main window) or `python code/leadtime_forecasts.py` (other windows), then `python code/leadtime_equal_fill.py [--offsets 0,26,52]` | `leadtime_equal_fill*.md` |
| Cost view | `python code/cost_view.py` | `cost_view.md` |
| N-HiTS baseline | `<python-with-neuralforecast> code/run_neural.py --dataset <D> --offset <N> --h 3`, then `python code/neural_eval.py` | `neural_eval.md` |

The run time depends on the dataset. On an Intel Core i5-12450H with 16 GB RAM, one horizon of LGB-Q takes about 2.5 minutes on VN1 and 21 minutes on M5.

## Notes

- Random seed 2026 is used throughout; models are refitted at the start of each 13-week block of a test window.
- Bootstrap intervals resample series, not periods.
- The combination C2 was specified after the main analysis, so its results are post hoc.
- Liquidation economics use an assumed cost model (holding 10–40% per year, gross margins 30–100%), not observed profits.

## License

The code is released under the MIT License (see `LICENSE`). The datasets remain under their original terms.
