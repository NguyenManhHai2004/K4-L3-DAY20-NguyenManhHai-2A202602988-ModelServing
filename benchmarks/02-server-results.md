# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 38 | 0.64 | 13000 | 19000 | 19000 | 8.1 | 0.0% |
| 50 | 44 | 0.74 | 30000 | 57000 | 58000 | 22.6 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.16x** (23% of linear) |
| P95 latency | **3.00x** |
| Effective concurrency at 50 users | 22.6 vs `--parallel 4` slots (occupancy/slot ratio 5.65) |

**Saturated.** Throughput delivered only 1.16x for 5x the offered load, and effective concurrency (22.6) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.16x while P95 moved 3.00x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**The server is already saturated at 10 users, so the saturation point is at or below 10 users, not somewhere near 50.**

- **The number that convinced me: effective concurrency 8.1 at 10 users.** The server has
  `--parallel 4` slots, and at only 10 users Little's Law already puts about 8 requests in
  the system (RPS 0.64 x average latency 12.6 s). That is twice the slot count, so at 10
  users about half the requests were already waiting for a slot. The 50-user run just made
  the queue longer: 22.6 in flight against 4 slots (occupancy ratio 5.65).
- **Throughput did not scale.** 5x the users delivered 1.16x the RPS (0.64 to 0.74, 23% of
  linear), while P95 rose 3.00x (19 s to 57 s) and the median 2.3x (13 s to 30 s). Latency grew
  much faster than throughput, so the extra latency is queue time, not compute time. The
  same short prompt averaged 11.7 s at 10 users and 29.1 s at 50 users (locust screenshots), and the
  token output per request did not change, so what grew is the wait for a slot.
- **The server's own gauges agree.** During `load-50`, `requests_processing` was 4 and
  `requests_deferred` was 44 to 46 in nearly every sample (50 users minus 4 slots), with
  `n_busy_slots_per_decode` about 3.98 of 4 (a running average, see the caveat in
  `02-server-batching-u50.md`). All four slots were full the whole time.
- **Is the limit the slot count?** Probably not. The 4 slots together produced about
  39.9 tok/s (2232 tokens in 55.9 s, from `02-server-metrics-u50.csv`), versus about 27 tok/s
  for one stream in `make bench`. So batching four requests gave only about 1.5x the
  single-stream rate, which means the hardware is compute-bound, not short of slots. That is
  the second row of the guide's table (throughput flat, slots full): adding slots would
  mostly add more waiting requests that share the same compute.

**Goodput at an SLO.** I choose the SLO "P95 <= 20 s". At 10 users P95 was 19 s, so that run
meets it and delivers 0.64 RPS of goodput. At 50 users P95 was 57 s, nearly 3x over the SLO,
so the 50-user run does not meet it at all: the extra 0.10 RPS of throughput was bought by
spending latency, and none of it is goodput. I cannot give an exact share of individual
requests under 20 s at 50 users, because locust only gave me aggregate percentiles, not
per-request latencies. So the most I can say for the SLO is: goodput is about 0.64 RPS at
10 users and 0 at the level of the P95 at 50 users.

**What I would change first: the compute path, not the slot count.** My tune results show
that on this machine the default `ngl=99` (Iris Xe through Vulkan) decodes at about 28 tok/s
single-stream, while `ngl=0` with 4 CPU threads gives about 36.8 tok/s (1.31x, see
`01-tuning-tg128.md`). If compute is the bottleneck, a faster compute path raises every
slot's speed and shrinks queue time directly. I would try `LAB_N_GPU_LAYERS=0` first. I have
not measured whether the CPU path also keeps that advantage with 4 concurrent slots (CPU
decode is memory-bound and batching may help it less), so this is a hypothesis to test, not a
result. Second choice: a smaller `--parallel`/lower offered load or admission control so the
queue does not grow without bound, since beyond 4 in flight a user only waits longer.

**Experiment I did not run.** The guide suggests `LAB_PARALLEL=1` vs 4 with `load-50`. I did
not run it because it would overwrite the CSVs this reading is based on. It would tell
whether slots or compute dominate directly.

**Caveats.** Small samples (38 and 44 completed requests, one run each, no repetition).
Effective concurrency uses the average latency of completed requests only, so requests
still in flight when the 60 s ended are not counted. The 10-user run is already past
saturation, so this lab never measured the below-saturation regime; to find the real knee
I would test 2 and 4 users.
