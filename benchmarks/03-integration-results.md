# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 4661.0 | 4661.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 2233.4 | 2233.6 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 8052.0 | 8052.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **4982.1** · total **4982.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **goodput** is more useful than raw throughput because it specifically counts only the requests per second that met the Target Time-to-Fault (TTFT) and Target Time-to-Poll (TPOT) targets.

Raw throughput ignores SLOs (Service Level Objects), meaning it does not account for how many requests actually met the performance requirements. Goodput addresses this by being SL

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

By storing the KV cache in non-contiguous pages, it removes the wasted space that would otherwise exist if the cache were stored contiguously.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when the **prefill operation is compute-bound** (requires significant CPU/GPU processing) and the **decode operation is memory-bandwidth-bound** (requires significant memory throughput), but the total memory bandwidth required for the entire pipeline is still manageable.

In this context, the context explicitly states that splitting helps because:
1.  **Prefill i


## Which N16-N19 pieces are real

**All four are stubbed in this run. Nothing from N16-N19 is wired into this pipeline.**

| Piece | Status in this run | What stands in for it |
|:--|:--|:--|
| N16 (cluster) | **stub** | one local `llama-server` process on this laptop, no cluster |
| N17 (data pipeline) | **stub** | none: documents are a hard-coded list |
| N18 (lakehouse) | **stub** | none: the corpus is `TOY_DOCS` (8 short snippets, STUB 1 in `pipeline.py`) |
| N19 (vector index) | **stub** | keyword-overlap scoring in `retrieve()` (STUB 2); no embedding server, so the `embed` stage is 0.0 ms and no vector index exists |

The only real component is the LLM stage: a real `llama-server` serving Qwen3.5 0.8B Q4_K_M.
(If I later wire in real code for any of N16-N19, this table must be updated; I am not
claiming any of it here.) Retrieval quality is also weak because it is keyword overlap: for
the first query only one document scored above 0 and the other two "contexts" were
zero-score padding.

## Latency per stage

Final run (`--base-url http://127.0.0.1:8080`, the table at the top): mean over 3 queries,
embed **0.0 ms**, retrieve **0.1 ms**, llm **4982 ms**, total **4982 ms**. Dominant stage:
llm (100%).

Inside the llm stage (from the server's own timings, summed over the 3 queries):
prefill 479 ms (3%) for 12 prompt tokens that were not cached, decode 13429 ms (90%) for
354 tokens (about 38 ms per token, the same as the `make bench` TPOT of 37.4 ms), and about
1 s (7%) of other overhead (HTTP, prompt templating, reading the response).

**Is the dominant stage what I expected? Yes.** With a 0.8B model on a laptop, embed and
retrieve are free (they are stubs), so the LLM took ~100%, and inside it, decode dominates:
latency is roughly output tokens x 38 ms. The third query hit the 200-token cap
(7577 ms), the first used 108 tokens (4120 ms), the second 46 (1732 ms).

## Two things I found while running it

1. **`localhost` costs 2.3 s per call on this Windows machine.** My first run used the
   default `http://localhost:8080` and measured mean llm = **7793 ms**, but the server's
   timings only accounted for about 5.4 s of it: a constant gap of about 2.35 s in every one
   of the 3 queries. A direct test confirmed it: the first request to `localhost` took
   2587 ms wall-clock against 227 ms to `127.0.0.1` for the same call. My explanation (not
   fully proven) is that `localhost` is tried over IPv6 (`::1`) first and the server
   only listens on `127.0.0.1`, so each new connection pays a fallback delay;
   `pipeline.py` opens a new connection per call, so every query paid it. Using `127.0.0.1`
   removed it (the gap fell to about 0.4 s per query). I have not checked how much this
   inflated the earlier load-test numbers: locust keeps connections alive, so I expect little
   effect, but I did not verify.
2. **Prompt caching showed up on the repeat run.** In the first run the prefill was 151, 114
   and 113 tokens; in the second run, with the same prompts, it was 4 tokens each, because
   the server reused its cached prefix. So the drop from 7793 ms to 4982 ms mixes two effects:
   about 2.3 s from the `localhost` delay and about 0.5 s from the cache. I did not separate
   them cleanly, and the second run is not a cold run.

## If I had to halve this pipeline's latency, I would attack the decode part of the llm stage

Decode is 90% of the llm stage, which is 100% of the total, and it scales with the number of
output tokens, so that is where the leverage is:

- **Cap and shorten the output.** Halving average output tokens roughly halves latency,
  because decode is about 38 ms per token. The third query ran to the 200-token cap, so a
  `max_tokens` limit and an instruction like "answer in 2 sentences" would help directly.
- **Faster decode per token.** My tune results show `ngl=0` with 4 CPU threads reaches about
  36.8 tok/s vs about 28 tok/s for the default GPU path (1.31x), which would cut decode time by
  about a quarter on its own. That is not enough to halve the latency alone, and I have not
  tested it inside this pipeline.
- Not worth attacking: retrieval and embedding (0.1 ms), and prefill (3%) unless the context
  grows. If the stubs are replaced by a real vector index, an embed server (`--embed-url`)
  could become a larger stage; I did not measure that option.

**Answer quality.** The 0.8B model's answers contain errors (for example it expanded TTFT as
"Throughput to Throughput" in the first run), so the numbers above measure serving, not answer
quality.
