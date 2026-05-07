# 02 — Server Results

| Concurrency | Requests | Failures | Avg (ms) | Median (ms) | P95 (ms) | P99 (ms) | RPS |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 10 | 1 | 0 | 42105 | 42105 | 42000 | 42000 | 0.02 |
| 50 | 2 | 0 | 36167 | 17000 | 55000 | 55000 | 0.03 |

Notes:
- CPU-only serving on this machine led to very low goodput under 60-second test windows.
- Metrics scrape during load showed deferred queue growth and low kv_cache usage ratio in this short run.
