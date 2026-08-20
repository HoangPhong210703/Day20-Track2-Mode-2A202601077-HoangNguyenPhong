# Bonus - Batch-size sweep (chunked prefill)

Host `Windows-AMD64` · llama.cpp `b10488` ·
`threads=8` `ngl=99` · metric `pp512`

| -b (logical) | -ub (micro) | pp512 (tok/s) | vs best |
|:--|--:|--:|--:|
| 128 | 128 | 939.7 | 88% |
| 256 | 256 | 1048.7 | 99% |
| 512 | 256 | 1063.1 | 100% |
| 512 | 512 | 1047.8 | 99% |
| 1024 | 512 | 1061.4 | 100% |
| 2048 | 512 | 1048.8 | 99% |

Best: `-b 512 -ub 256` at 1063.1 tok/s
(1.13x the slowest point tested).

This sweep only measures the throughput half of the trade. The cost it hides is
TTFT for queued requests: a larger micro-batch holds the device longer per step,
so anything waiting behind it waits longer. To see both halves, re-run
`make load-50` with your best and worst settings via
`.venv/bin/python labs/02-serve/serve.py -- -b N -ub M` and compare P95.

## Your finding

I would run `-b 512 -ub 256`, but the honest reading is a **plateau, not a peak**:
everything from 256 upward sits within 1.5% of best, and only `-b 128 -ub 128` (88%)
is genuinely worse. So the decision is "do not go below 256", not "tune to 512/256".

The number that matters is elsewhere. llama-bench prefills at **1063 tok/s** here, while
my `llama-server` runs measured only ~74-87 tok/s of prefill. The hardware is over 12x
faster at prefill than my server configuration achieves, which is what makes
`--ubatch-size` worth attacking.

To be sure it does not hurt a contended P95 I would re-run `load-50` at `-ub 256` versus
`-ub 512` and compare P95 plus `n_busy_slots_per_decode`: a larger micro-batch holds the
device longer per decode step, so queued requests wait longer even as throughput rises.
