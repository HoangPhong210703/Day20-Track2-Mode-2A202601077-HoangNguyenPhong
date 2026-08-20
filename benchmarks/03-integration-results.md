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


## Which N16-N19 pieces are real

N16 Cloud/IaC - **stub**. N17 Data pipeline - **stub**. N18 Lakehouse - **stub**.
N19 Vector + features - **stub**: no embedding model is loaded, retrieval is keyword
overlap over an in-memory corpus in `pipeline.py`. N20 serving is **real**
(`llama-server`).

So the 100% llm split is an artefact, not a finding: embed is 0.0 ms because nothing
embeds, retrieve 0.1 ms because the corpus is a handful of dicts.

To halve latency I would attack dead time inside the llm stage first. Client-side llm
was 5.7-7.5 s, but the server reported only 2.3-3.1 s of prefill+decode. Over half that
stage is not compute.
