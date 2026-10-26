# Champion model

Created 2026-10-21 12:35. Generated automatically by `register_champion.py` and replaced on every run.

## Current champion

- **Model:** LightGBM (registry `readmission`, version 2, alias `champion`).
- **Load it with:** `mlflow.sklearn.load_model("models:/readmission@champion")`
- **Training run:** `4270f0d3da7546dcaaca227cec44d00f`; saved file `models/02_lightgbm_prauc0.224.joblib`.
- **Registered:** 2026-10-21 12:35. Registry copy checked against the saved file: predictions match.

## Why it is the champion

Passed gate; tied with XGBoost on PR-AUC; chosen for more credible SHAP feature reasoning and stability.

## Evidence (test split, never used for training)

| Measure | Value |
|---|---|
| Test PR-AUC | 0.222 (95% range 0.200 to 0.247) |
| CV PR-AUC at training | 0.224 |
| Test ROC-AUC | 0.670 |
| Lift over chance | 1.93x |

Gate checks:

- Pass: lift >= 1.5
- Pass: CV-to-test drop <= 0.02
- Pass: ROC-AUC gap by group <= 0.05
- Pass: recall gap by group <= 0.1
- Pass: beats logistic baseline

## Other registered versions

| Version | Model | Alias |
|---|---|---|
| 1 | XGBoost | `challenger` |
| 2 | LightGBM | `champion` |
| 3 | Logistic regression | `baseline` |

## Settings it was trained with

`{"class_weight": null, "colsample_bytree": 0.8, "learning_rate": 0.05, "max_depth": -1, "min_child_samples": 40, "min_child_weight": 0.001, "n_estimators": 300, "num_leaves": 31, "random_state": 42, "subsample": 0.8, "subsample_freq": 1, "scale_pos_weight": 7.704}`

## Replacing the champion

A challenger replaces this model only if it passes the same evaluation gate and beats it on the test split by a margin that is clearly more than noise (the 95% range of the paired PR-AUC difference must be entirely above zero). A tie is not a reason to switch.

## Limits

- Scores rank patients by risk. They are not calibrated probabilities (class weights inflate them).
- Performance is weakest for the oldest patients; see the fairness section of `models/test_report.md`.
- Not validated for clinical use.
