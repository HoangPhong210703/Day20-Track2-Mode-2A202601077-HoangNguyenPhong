# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 7527.1 | 7527.2 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 5713.7 | 5713.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 5924.9 | 5925.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **6388.6** · total **6388.7**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **goodput** is more useful than raw throughput because it specifically accounts for **SLOs** (Service Level Objectives).

While raw throughput measures total requests per second, goodput filters for only those requests that met the TTFT (Total Throughput to Failure) and TPOT (Total Throughput to Performance) targets. This ensures that the system only counts requests 

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

By storing the KV cache in non-contiguous pages, it avoids the wasted space that would occur if all data were packed tightly into a single contiguous block of memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when the prefill operation is **compute-bound** and the decode operation is **memory-bandwidth-bound**, as stated in the context. This allows the system to split the workload across separate pools, enabling the engine to skip prefill entirely when a shared prefix is used in RadixAttention keys.


## Which N16-N19 pieces are real (required -- replace this line)

_List each of N16, N17, N18, N19 as real or stubbed. Stubbing costs no points;
misrepresenting it does. Then answer: is the dominant stage above what you expected?
If you had to halve this pipeline's latency, which stage would you attack and why?_
