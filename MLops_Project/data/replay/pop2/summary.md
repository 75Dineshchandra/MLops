# Replay summary: pop2

Created 2026-10-28 15:39:38. Status: finished. Model version(s) seen: 2. Seed 42, 120.0 s per simulated week.

Flagged share in clean weeks: - (the cutoff is set so that about 20% of reference stays are flagged; the alert band is 12% to 28%).

| Week | Injected | Stays scored | Requests | Rejected (422) | Flagged | Mean score | Unseen values (API counter change) |
|---|---|---|---|---|---|---|---|
| 11 | population_shift_young 0.80 | 981 | 10 | 0 | 20.5% | 0.404 | 1 |
| 12 | population_shift_young 0.80 | 906 | 10 | 0 | 16.3% | 0.389 | 0 |

## What to look at

Grafana (http://localhost:3000, dashboard Readmission API health), Prometheus alerts (http://localhost:9090/alerts) and the console's Monitoring tab.

- **population_shift_young:** Weeks get much younger and smaller; the age mix changes drastically. Scores may or may not move.

Record in your notes: which scenario, the first week it was visible, which alert fired and how long after the first affected week. A scenario nothing reacted to is a monitoring gap, which is a finding, not a failure of the replay.

## Limits

- Everything above is label-free: it can show that inputs or scores moved, not that predictions got worse. Real performance needs the late labels in `labels.csv` (step 14).
- Weeks are simulated by shuffling whole patients; the data has no dates.
