# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 17.2 | 47% |
| 2 | 27.6 | 75% |
| 4 | 36.8 | 100% |
| 8 | 36.9 | 100% |
| 16 | 26.6 | 72% |

**Best**: `-t 8` at 36.9 tok/s
**Slowest tested**: `-t 1` at 17.2 tok/s (2.15x spread)
**Against the physical-core default** (`-t 4`, 36.8 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

**Knee at 4 threads = my 4 physical cores. Beyond it, extra threads buy nothing and then hurt.**

CPU-only sweep above (`ngl=0`, `tg128`, Q4_K_M, 2 reps per point):
1 -> 17.2, 2 -> 27.6, 4 -> 36.8, 8 -> 36.9, 16 -> 26.6 tok/s.

- **Rising part (1 -> 4 threads): 17.2 -> 36.8 tok/s, 2.14x.** Each physical core adds
  its own execution units and L1/L2, so decode scales until the cores together saturate
  what the memory system can feed them.
- **Flat part (4 -> 8): 36.8 vs 36.9, identical within noise.** The 4 extra threads are
  hyperthreads. They share the physical core's execution units and cache with the first
  four, so they add no extra weight reads per second. Decode is a stream of reading
  weights, which is memory-bound, and the memory channel is already saturated at 4.
- **Falling part (16 threads, 2x logical): 26.6 tok/s, 72% of best.** This oversubscribes
  8 logical cores with 16 threads. Threads get descheduled and the per-token barriers
  that llama.cpp uses between layers wait on the slowest (preempted) thread. This is
  scheduling and synchronisation overhead; I did not profile it, so that is the likely
  mechanism, not a measured one.
- `-t 8` shows the "best" label (36.9 vs 36.8) but that 0.3% gap is noise, so I treat the
  physical-core count (4) as the real optimum: it is the same speed with half the threads.

**The bigger finding: on this machine the GPU path is slower than the CPU path.**
The first sweep used the default `ngl=99` (all layers offloaded to the Intel Iris Xe via
Vulkan) and was flat: 27.1, 27.8, 27.9, 27.8, 28.0 tok/s for 1/2/4/8/16 threads (1.03x
spread). Threads barely matter there because decode runs on the GPU, so the CPU only
launches work. That GPU result (about 28 tok/s) is also what `make bench` measured for
Q4_K_M through the server (26.8 tok/s). CPU-only at 4 threads is 36.8 tok/s, about
**1.31x faster** than the offloaded default (36.8 / 28.0).

Likely reason (hypothesis, not measured): the Iris Xe is an integrated GPU with no VRAM
of its own, so it reads the same system RAM the CPU does and gets no bandwidth advantage,
and for a 0.5 GB model the per-token cost of dispatching many small Vulkan kernels per
layer is large relative to the work. I did not capture a Vulkan profile to confirm this.

**Before / after for REFLECTION section 5:** default `ngl=99` ~28.0 tok/s -> `ngl=0 -t 4`
36.8 tok/s (1.31x). Within the CPU path, `-t 1` -> `-t 4` is 2.14x.

**Caveat.** Single run on a laptop; one model/quant (Q4_K_M); `llama-bench` reports `tg128`
only, so this is decode speed, not TTFT/prefill. Laptop thermal throttling or background
load could move individual points by a few percent. The GPU sweep file was overwritten
by the CPU one; its numbers above come from the run I made just before.
