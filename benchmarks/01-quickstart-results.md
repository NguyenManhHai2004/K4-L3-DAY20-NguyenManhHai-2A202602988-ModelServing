# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 6860 | 599 / 638 | 37.4 / 39.0 | 2943 / 3074 / 3074 | 26.8 |
| UD-Q2_K_XL | 0.39 | 6187 | 704 / 744 | 62.3 / 63.4 | 4630 / 4720 / 4720 | 16.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.66x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

**Not worth it on this machine: the 2-bit quant is both slower and worse.**

- **Speed.** `UD-Q2_K_XL` is *slower*, not faster: TPOT P50 62.3 ms vs 37.4 ms (1.66x worse),
  decode 16.1 vs 26.8 tok/s, E2E P50 4630 vs 2943 ms. TTFT is also worse (704 vs 599 ms).
  It saves only 0.11 GB (0.39 vs 0.50 GB), and load time is about the same (6.2 s vs 6.9 s).
- **Which case is mine.** This run used `ngl=99` with the Vulkan build, and the probe
  confirms the runtime enumerates a real device (`Vulkan0: Intel(R) Iris(R) Xe Graphics`),
  so all layers were offloaded to the integrated GPU, not run on the CPU. That makes the
  "no GPU offload" case in the note above not mine. Fewer bytes did not translate into
  faster decode here, so decode is not limited by weight bandwidth for a 0.5 GB model:
  the Q2 format's heavier dequantization work costs more than the 0.11 GB it saves. Why
  it is that much slower on an Iris Xe (for example less optimised Vulkan kernels for
  this quant type, or the iGPU sharing system RAM) is my hypothesis, not something I
  measured; I did not run a CPU-only comparison (`LAB_N_GPU_LAYERS=0`).
- **Quality (same prompts, `temperature=0`, one run each, so anecdotal).**
  - "What is 17 * 23?": Q4 answered `391` (correct); Q2 answered `381` (wrong).
  - Vietnamese explanation of continuous batching, max 4 sentences: Q4 gave one short,
    fluent answer (43 tokens) that is only partly right (it describes merging requests
    into a bigger batch, missing the per-iteration scheduling that makes batching
    "continuous"). Q2 ignored the 4-sentence limit, ran to the 200-token cap, repeated the
    same sentence several times and described the wrong concept (data grouping and caching).
- **Verdict.** On a model this small, 2-bit costs visible accuracy and instruction-following
  for no speed gain, so I would serve `Q4_K_M`. The 2-bit file would only be worth it if
  disk or RAM were the binding constraint, which is not the case with 15.7 GB RAM.
- **Caveat.** Only 10 prompts per quant, `max_tokens=64`, and the first run after setup can
  be slower because the weights are not yet in the OS page cache.
