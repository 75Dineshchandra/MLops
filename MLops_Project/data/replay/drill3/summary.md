# Replay summary: drill3

Created 2026-10-27 16:45:19. Status: finished. Model version(s) seen: 2, 3. Seed 42, 90.0 s per simulated week.

Flagged share in clean weeks: 20.1% (the cutoff is set so that about 20% of reference stays are flagged; the alert band is 12% to 28%).

| Week | Injected | Stays scored | Requests | Rejected (422) | Flagged | Mean score | Unseen values (API counter change) |
|---|---|---|---|---|---|---|---|
| 1 | none | 3,085 | 31 | 0 | 19.5% | 0.422 | 0 |
| 2 | none | 3,086 | 31 | 0 | 19.9% | 0.428 | 2 |
| 3 | none | 3,084 | 31 | 0 | 20.8% | 0.467 | 0 |
| 4 | none | 3,086 | 31 | 0 | 21.7% | 0.471 | 1 |
| 5 | none | 3,084 | 31 | 0 | 19.6% | 0.466 | 1 |
| 6 | none | 3,085 | 31 | 0 | 19.6% | 0.461 | 0 |
| 7 | none | 3,085 | 31 | 0 | 20.1% | 0.432 | 2 |
| 8 | none | 3,085 | 31 | 0 | 19.3% | 0.422 | 3 |

## Limits

- Everything above is label-free: it can show that inputs or scores moved, not that predictions got worse. Real performance needs the late labels in `labels.csv` (step 14).
- Weeks are simulated by shuffling whole patients; the data has no dates.
