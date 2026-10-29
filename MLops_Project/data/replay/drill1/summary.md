# Replay summary: drill1

Created 2026-10-27 15:46:23. Status: finished. Model version(s) seen: 2. Seed 42, 90.0 s per simulated week.

Flagged share in clean weeks: 20.2% (the cutoff is set so that about 20% of reference stays are flagged; the alert band is 12% to 28%).

| Week | Injected | Stays scored | Requests | Rejected (422) | Flagged | Mean score | Unseen values (API counter change) |
|---|---|---|---|---|---|---|---|
| 1 | none | 4,114 | 42 | 0 | 19.1% | 0.424 | 0 |
| 2 | none | 4,113 | 42 | 0 | 22.4% | 0.433 | 2 |
| 3 | none | 4,114 | 42 | 0 | 20.1% | 0.426 | 1 |
| 4 | none | 4,113 | 42 | 0 | 20.1% | 0.426 | 1 |
| 5 | none | 4,113 | 42 | 0 | 20.4% | 0.430 | 1 |
| 6 | none | 4,113 | 42 | 0 | 19.0% | 0.423 | 4 |

## Limits

- Everything above is label-free: it can show that inputs or scores moved, not that predictions got worse. Real performance needs the late labels in `labels.csv` (step 14).
- Weeks are simulated by shuffling whole patients; the data has no dates.
