# Bonus (B5 / C8) - Semantic cache in front of the chat endpoint

Host `Windows-AMD64` · run offline (`--offline --sweep`) · 8-prompt stream · threshold 0.8

| Metric | Value |
|:--|--:|
| Cache hits | 3 of 8 |
| Hit rate | 38% |
| LLM calls saved | 3 |
| Latency on a hit | 0 ms (vs ~250 ms simulated miss) |

| Threshold | Hits |
|--:|--:|
| 0.70 | 3/8 |
| 0.80 | 3/8 |
| 0.85 | 3/8 |
| 0.90 | 3/8 |
| 0.95 | 3/8 |

## Finding

A semantic-cache hit skips **100% of compute** - no prefill, no decode - which is a
different saving from the KV/prefix cache underneath it. That layer only reuses a shared
prefix; this one returns the whole answer. On my numbers 3 of 8 prompts never reached the
model at all.

The threshold sweep is **flat, and that is an artifact, not a result**: offline mode uses
a bag-of-words stub embedder whose cosine similarity is essentially 1.0 or 0.0, so no
threshold between 0.70 and 0.95 can separate the cases. A real curve needs real
embeddings (`make serve-embed`, then drop `--offline`). I am reporting the stub run
honestly rather than presenting a flat line as evidence that threshold does not matter.

The risk the deck understates: too low a threshold returns a confidently wrong answer to
a paraphrase that was not equivalent, and an unsalted shared cache leaks prompts across
tenants through hit/miss timing (NDSS'25).
