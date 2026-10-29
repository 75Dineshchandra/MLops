# Replay summary: demo2

Created 2026-10-29 09:38:29. Status: finished. Model version(s) seen: 2. Seed 42, 120.0 s per simulated week.

Flagged share in clean weeks: 20.7% (the cutoff is set so that about 20% of reference stays are flagged; the alert band is 12% to 28%).

| Week | Injected | Stays scored | Requests | Rejected (422) | Flagged | Mean score | Unseen values (API counter change) |
|---|---|---|---|---|---|---|---|
| 1 | none | 2,057 | 21 | 0 | 18.7% | 0.419 | 0 |
| 2 | none | 2,057 | 21 | 0 | 19.6% | 0.428 | 0 |
| 3 | none | 2,057 | 21 | 0 | 20.8% | 0.427 | 2 |
| 4 | none | 2,056 | 21 | 0 | 24.0% | 0.439 | 0 |
| 5 | covariate_shift 0.50 | 2,057 | 21 | 0 | 48.5% (outside band) | 0.524 | 0 |
| 6 | covariate_shift 0.50 | 2,057 | 21 | 0 | 49.7% (outside band) | 0.536 | 1 |
| 7 | none | 2,057 | 21 | 0 | 20.2% | 0.429 | 1 |
| 8 | new_categories 0.05 | 2,056 | 21 | 0 | 23.2% | 0.434 | 204 |
| 9 | new_categories 0.05 | 2,056 | 21 | 0 | 23.0% | 0.436 | 198 |
| 10 | none | 2,057 | 21 | 0 | 21.0% | 0.436 | 1 |
| 11 | bad_data 0.25 | 2,063 | 28 | 7 | 19.7% | 0.426 | 1 |
| 12 | population_shift 0.60 | 1,676 | 17 | 0 | 19.1% | 0.430 | 3 |

## What to look at

Grafana (http://localhost:3000, dashboard Readmission API health), Prometheus alerts (http://localhost:9090/alerts) and the console's Monitoring tab.

- **bad_data:** Rejected-requests panel (nulls, not_numeric, missing_columns); error share; ReplayErrorShare alert.
- **covariate_shift:** Flagged share rises above its clean-week level; the score heatmap shifts up.
- **new_categories:** Unseen-category panel rises; ReplayUnseenCategories alert.
- **population_shift:** Weeks get smaller and older; flagged share and the score distribution move.

Record in your notes: which scenario, the first week it was visible, which alert fired and how long after the first affected week. A scenario nothing reacted to is a monitoring gap, which is a finding, not a failure of the replay.

## Limits

- Everything above is label-free: it can show that inputs or scores moved, not that predictions got worse. Real performance needs the late labels in `labels.csv` (step 14).
- Weeks are simulated by shuffling whole patients; the data has no dates.
