# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` � llama.cpp `b10488` �
`--parallel 4` � `ctx=2048` � `threads=4` �
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 8 | 0.14 | 42000 | 59000 | 59000 | 4.7 | 0.0% |
| 50 | 11 | 0.21 | 35000 | 53000 | 53000 | 7.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.52x** (30% of linear) |
| P95 latency | **0.90x** |
| Effective concurrency at 50 users | 7.4 vs `--parallel 4` slots (occupancy/slot ratio 1.84) |

**Saturated.** Throughput delivered only 1.52x for 5x the offered load, and effective concurrency (7.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (0.90x vs 1.52x), so this server still has headroom at 50 users.

> **Small sample.** Only 8 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

The server saturates at or before 50 users. The number that convinced me is effective
concurrency at 50 users (7.4) exceeding the 4 available `--parallel` slots -- confirmed
by `requests_deferred=46` in the metrics run, which proves requests were genuinely
queuing rather than all being served immediately. Throughput only rose 1.52x for a 5x
increase in offered load, which is the classic signature of saturation. P95 did not
grow faster than throughput (0.90x), so there's still headroom before latency blows up.
To raise goodput@SLO I would increase `--parallel` first (more concurrent decode slots),
since the bottleneck is clearly slot availability (queueing), not per-token compute --
raising thread count or GPU offload wouldn't relieve a queue that's waiting for a free
slot, not for faster decode.
