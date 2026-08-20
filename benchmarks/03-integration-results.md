# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` � llama.cpp `b10488` �
retrieval backend: **keyword overlap** � 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 6257.4 | 6257.5 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 4256.1 | 4256.2 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 7716.4 | 7716.5 |

Mean per stage (ms): embed **0.0** � retrieve **0.1** �
llm **6076.6** � total **6076.7**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **goodput is more useful than raw throughput** because:

1.  **It respects SLOs:** Goodput counts only requests per second that met the Target Time-to-Fill (TTFT) and Target Time-to-Poll (TPOT) targets.
2.  **It avoids ignoring SLOs:** The text explicitly states that "Throughput at saturation ignores SLOs." Since throughput measures total requests per second without 

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

By storing the KV cache in non-contiguous pages, it removes the wasted space that would otherwise be consumed by the internal fragmentation of contiguous memory blocks (like a stack or a fixed-size buffer).

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when the prefill operation is **compute-bound** (requiring significant CPU/GPU cycles) and the decode operation is **memory-bandwidth-bound** (requiring significant memory throughput).

By splitting them into separate pools, the system can:
1.  **Maximize compute efficiency**: The compute-intensive prefill phase can be handled by dedicated compute resources (like


## Which N16-N19 pieces are real

N16 (Cloud/IaC), N17 (Data pipeline), N18 (Lakehouse): not used by this pipeline.
N19 (Vector + features / retrieval): **stubbed** -- keyword overlap matching, not real
embeddings or a vector index. N20 (Serving, `llama-server`): **real** -- actual model
inference over HTTP.

The dominant stage (llm, 100% of total) is exactly what I expected, since the stub
retrieval takes ~0.1ms and doesn't compete with a multi-second LLM call at all. If I
had to halve this pipeline's latency, I'd attack the llm stage directly -- e.g. switch
to a faster quant, cap `max_tokens` lower, or increase GPU offload -- since embed and
retrieve together are already under 0.2ms and have essentially nothing left to cut.
