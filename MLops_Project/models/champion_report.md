# Champion model

Updated 2026-10-27 16:41 by `promote.py`. Replaced on every promotion or rollback.

- **Model:** LightGBM (registry `readmission`, version 2, alias `champion`).
- **Load it with:** `mlflow.sklearn.load_model("models:/readmission@champion")`
- **Training run:** `4270f0d3da7546dcaaca227cec44d00f`
- **Scores:** CV PR-AUC 0.224, test PR-AUC 0.222, test ROC-AUC 0.67, lift 1.93x.
- **Why it is champion:** Rolled back from version 3.

Full evidence for the latest decision is in `models/promotion_report.md`; every decision is in `models/promotion_log.jsonl`. Scores rank patients by risk and are not calibrated probabilities.
