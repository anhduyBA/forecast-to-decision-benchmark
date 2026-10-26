# Results manifest — from each number in the paper to the code and output behind it

Paper version: rev22 (ICIIT 2027 manuscript, 10 pages). All commands run from the repository root with Python 3.12 and
`code/requirements.txt`; random seed 2026 throughout (`code/f2d/config.py`). Data are not redistributed (see
`README.md`). Forecasts are cached in `data/cache/forecasts/<DATASET>[_wN]/`; the analysis scripts below read that cache,
so the forecasting runs come first.

## 0. Forecasting runs (needed by everything else)

| Run | Command | Cache / output |
|---|---|---|
| M5, VN1, three windows | `python code/run_pipeline.py --dataset M5 --grid tau --boot 200` and `--dataset VN1`; repeat with `--offset 26` and `--offset 52` | `data/cache/forecasts/<D>[_w26,_w52]/`, `code/outputs/<D>[_wN]/` |
| Footwear panel | `python code/profile_datasets.py`, then `python code/run_pipeline.py --dataset VNF --models empirical,tsb_nb,ets,lgb_tweedie,lgb_conformal,hgb_quantile,lgb_quantile` | `data/cache/forecasts/VNF/` (rerun reproduces the cached forecasts exactly) |

## 1. Tables and figure

| Paper element | Content | Script | Output in `code/outputs/` |
|---|---|---|---|
| Table 1 | dataset characteristics before the main window | `sync_check.py`, `profile_datasets.py` | `sync_check.md`, `data_profile.md` |
| Table 2 + note | wQL, median scaled loss, mean ranks; Nemenyi CD; h = 13 ranks | `stat_tests.py --models <seven> --out-suffix m7 [--offset N]`, `robust_accuracy.py --models <seven> --out-suffix m7` | `stat_tests*_m7.md`, `<D>[_wN]_m7/stat_tests.csv`, `robust_accuracy_m7.md` |
| Table 3, Figure 1 | inventory at fill 0.94 with series-bootstrap CIs; inventory–fill curves | `equal_fill_ci.py --models <seven> --out-suffix m7 [--offset N]`; Figure 1 is drawn from `<D>/kpi.csv` (default policy, all series) by the authors' paper-build script, not included here | `equal_fill_ci*_m7.md`, `<D>/kpi.csv` |
| Table 4 | stable vs growing demand groups (retrospective) | `demand_shift_analysis.py` | `demand_shift.md` |
| Table 5 | selection rules trained on one window, tested on the next | `segment_series.py`, then `segment_eval.py` | `segmentation.md`, `segment_eval.md` |
| Table 6 | liquidation outcomes, break-even salvage ratio s* | `liquidation_breakeven.py --dataset M5` (and `VN1`) | `<D>[_wN]/breakeven.csv`, `<D>[_wN]/kpi.csv` |
| Table 7 | past-only τ calibration: target vs achieved fill, inventory; full-policy cluster-bootstrap intervals; † flags | `prospective_tau.py` → `rolling_calibration.py` (point values) → `full_policy_bootstrap.py` (intervals) | `robustness/rolling_calibration.md`, `robustness/full_policy_bootstrap.md` |

`<seven>` = `empirical,tsb_nb,ets,lgb_tweedie,lgb_conformal,hgb_quantile,lgb_quantile`.

## 2. Numbers in the text

| Section | Numbers | Script | Output in `code/outputs/` |
|---|---|---|---|
| 3.1 | classifier AUC 0.91 between M5 and VN1 | `dataset_shift.py` | `dataset_shift.md` |
| 5.5 | C2; E7 / E7-med; leave-one-out ablation; 127 subsets; prospective subset rule | `combination.py`, `ensemble_ablation.py` | `combination.md`, `ensemble/ensemble_ablation.md` |
| 5.7 | reweighting VN1 to M5 (SMD 0.32–0.39 → 0.02–0.03); price and shared-feature ablations | `matched_comparison.py`; `run_pipeline.py --dataset VN1 --no-price`; `run_pipeline.py --dataset M5 --shared-features` | `matched_comparison.md`, `equal_fill_ci*_nop.md`, `equal_fill_ci*_sf.md` |
| 5.8 Hyperparameters | four LightGBM settings | `hparam_sensitivity.py --datasets VN1,M5` | `hparam_sensitivity.md` |
| 5.8 Lead time | lead times 1 and 4 weeks, 18 runs | `leadtime_forecasts.py`, `leadtime_equal_fill.py --offsets 0,26,52` | `leadtime_equal_fill*.md` |
| 5.8 Neural baseline | N-HiTS | `run_neural.py`, `neural_eval.py` | `neural_eval.md` |
| 5.8 Computational cost | run times | logs of the forecasting runs | `logs/` of the authors' runs (not all public) |
| 5.8 Prospective τ | τ chosen on the previous window | `prospective_tau.py` | `robustness/prospective_tau.md` |
| 5.8 Resampling | item, store, warehouse, client and week-block bootstraps | `bootstrap_robust.py` | `robustness/bootstrap_robust.md` |
| 5.8 Table 7 reading | 13 of 20 comparable; 1 saving, 5 more stock, 7 inconclusive; thresholds 0.25 / 1.0 | `service_tradeoff.py` (after `full_policy_bootstrap.py`) | `robustness/service_tradeoff.md` |
| 5.8 Footwear panel | E7-med +2.4% at fill 0.92 (style-level interval) | `rolling_calibration.py` | `robustness/vnf_validation.md` |
| 6 Censored demand | detector shares; clean-subset and imputed rankings; E7-med; 12 scenarios | `censoring_check.py`, `censoring_rank_check.py`, `censoring_rank_check_e7.py`, `censoring_sensitivity.py` | `censoring/*.md` |
| 6 Ranking reversal | warehouse-level ρ 0.21–0.27; decision value −9.4 / −0.2 / +2.9% | `ranking_reversal.py` | `robustness/ranking_reversal.md` |
| 6 Regret | maximum regret over six runs | `combination.py`, `ensemble_ablation.py` | `combination.md`, `ensemble/ensemble_ablation.md` |

## 3. Order of the robustness scripts

```bash
python code/prospective_tau.py          # τ* per method and transition (needed by the next two)
python code/rolling_calibration.py      # Table 7 point values and footwear validation (checkpoints in robustness/rolling_parts/)
python code/full_policy_bootstrap.py    # Table 7 intervals, 2,000 paired replicates (checkpoints in robustness/fpb_parts/)
python code/service_tradeoff.py         # comparability counts and thresholds
python code/bootstrap_robust.py
python code/ranking_reversal.py
```

`full_policy_bootstrap.py` checks that its original-sample values equal those of `rolling_calibration.py` (largest
difference reported at the end of `full_policy_bootstrap.md`; 0 for the published run).

## 4. Design notes

The designs and decision criteria of the combination analysis, the prospective checks and the final robustness work
were written down before the runs, in the authors' working repository (`07_paper_draft/phase5_workflow.md`,
sections 7–10). The closest-study comparison is in `docs/contribution_matrix.md`.
