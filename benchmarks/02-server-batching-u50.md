# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` � `--parallel 4` � 15 samples over
60s at 2.0s intervals � raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.67 of 4 slots (92%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a � not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1249 |

Highest sampled value was **3.67 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width was 3.67 of 4 slots (92%). `02-server-results.md` reports effective
concurrency at 50 users as 7.4 -- higher than the peak busy-slot gauge. The two don't
disagree so much as measure different things: effective concurrency (Little's Law,
RPS x latency) counts every in-flight request including ones still queued waiting for
a free slot, while `n_busy_slots_per_decode` only counts slots actually decoding right
now. I trust the slot gauge (3.67/4) for how full the server actually got, and the
Little's Law number (7.4) as evidence that demand outstripped those 4 slots -- the gap
between them (7.4 vs ~4) is exactly the queued/deferred requests
(`requests_deferred=46`) waiting their turn.
