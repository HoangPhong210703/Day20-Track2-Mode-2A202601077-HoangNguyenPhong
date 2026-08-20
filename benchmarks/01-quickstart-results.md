# 01 - Measure: latency baseline



Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`

Settings: `threads=8` `ngl=99` `ctx=2048`

`max_tokens=64` · warm-up discarded

Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10



| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |

|:--|--:|--:|--:|--:|--:|--:|

| Q4_K_M | 0.50 | 3551 | 2054 / 2163 | 12.8 / 13.5 | 2866 / 2942 / 2942 | 78.3 |

| UD-Q2_K_XL | 0.39 | 3543 | 2168 / 2274 | 14.4 / 15.5 | 3069 / 3250 / 3250 | 69.6 |



- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.

- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.

- `UD-Q2_K_XL` decodes **1.12x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.



## Your observation

`UD-Q2_K_XL` is not worth it here: 22% smaller, but 12.5% worse TPOT and 0.89x decode.

Decode is not bandwidth-bound. Flat load time 3551 vs 3543 ms.

Slower and no more accurate: dominated, so there is no tradeoff to weigh.
