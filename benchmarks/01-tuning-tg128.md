# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 82.8 | 97% |
| 4 | 85.3 | 100% |
| 8 | 84.0 | 98% |
| 16 | 83.7 | 98% |
| 32 | 83.6 | 98% |

**Best**: `-t 4` at 85.3 tok/s
**Slowest tested**: `-t 1` at 82.8 tok/s (1.03x spread)
**Against the physical-core default** (`-t 8`, 84.0 tok/s): 1.02x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

There is no knee. The curve is flat from 1 to 32 threads - 82.8 to 85.3 tok/s, a 1.03x
spread - and `-t 1` already reaches 97% of best. Ordering inside that 3% band is noise,
so "best = 4" is not a real peak.

Reason: with `ngl=99` the layers are on the iGPU, and `-t` only sizes host-side work.
The arithmetic confirms it. At `-t 1`, 82.8 tok/s x 0.50 GB = ~41 GB/s of weight
traffic, which a single Zen3+ core cannot sustain from DRAM. That thread is not doing
the decode. Thread count is simply not the knob here.
