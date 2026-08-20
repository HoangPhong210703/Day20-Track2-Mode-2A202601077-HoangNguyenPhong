# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 16 | 0.31 | 28000 | 41000 | 41000 | 7.8 | 0.0% |
| 50 | 22 | 0.39 | 32000 | 56000 | 57000 | 12.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.24x** (25% of linear) |
| P95 latency | **1.37x** |
| Effective concurrency at 50 users | 12.0 vs `--parallel 4` slots (occupancy/slot ratio 3.01) |

**Saturated.** Throughput delivered only 1.24x for 5x the offered load, and effective concurrency (12.0) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.24x while P95 moved 1.37x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 16 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

Saturated at or below 10 users. The number that convinced me: `requests_deferred` sat at
45 while `requests_processing` stayed at 4 - 49 requests in the system against 50
offered, so nearly everything was waiting, not running. `n_busy_slots_per_decode` held
3.76/4 (94%), leaving no idle capacity. 5x offered load returned 1.24x throughput and
1.37x P95: that difference is queue time.

First knob would be `--ubatch-size`, not `--parallel`. Warm prefill runs at ~74 tok/s
against decode's ~78, yet prefill processes the whole prompt in parallel and should be
far faster per token - so that path is what to widen. More slots only split a budget
4-way concurrency already halved: 2295 tokens in 56.4s is 41 tok/s aggregate, against
~76 single-stream.
