# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.98 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 15809 |

Highest sampled value was **3.98 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

**Peak batch width: 3.98 of 4 slots. Continuous batching is clearly happening, and the server is saturated at 50 users.**

- **What the samples show.** In all 14 samples taken during `load-50`, `requests_processing`
  was 4 (all slots busy) and
  `n_busy_slots_per_decode` stayed between 3.97 and 3.98. Decode steps were therefore
  almost always shared by 4 requests at once, not served one at a time.
- **Queueing.** `requests_deferred` sat at 44 to 46 for most of the run, then fell to 41 and
  31 in the last two samples as the load ended. 46 is the 50 virtual users minus the 4 in
  slots, so every user beyond the 4 slots was waiting. That wait is the queue time that
  shows up in the load-50 latency: Aggregated median about 30 s and P95 about 57 s
  (`locust-50` screenshot), against a median of about 13 s at 10 users.
- **Throughput barely moved.** Aggregated RPS was 0.64 at 10 users and 0.74 at 50 users, and
  completed requests went from 38 to 44. With 10 users already above the 4 slots, the server
  was already at capacity at 10 users, so adding 40 more only lengthened the queue.
- **Caveat on the gauge.** `n_busy_slots_per_decode` is a running average maintained by
  llama.cpp, not a per-interval value, and this server had already served the 10-user run
  and earlier trial runs before sampling started (the very first sample is already 3.97).
  So the 3.98 mostly reflects the whole server lifetime, and it cannot by itself prove that
  the 50-user phase packed more than the 10-user phase did. The stronger evidence for this
  run is `requests_processing = 4` together with `requests_deferred` around 46 in every
  sample, which are instantaneous. A cleaner test would restart the server before
  `load-50` so the average starts from zero.
- **Does it match effective concurrency in `02-server-results.md`?** I have not run
  `load-report` yet, so I have not compared the two. I will fill that in after reading it.
  If they disagree I would trust the instantaneous gauges (`requests_processing`) over the
  running average, for the reason above.
- **Sampling note.** The sampler took 14 samples in 60 s (about one every 4.3 s) instead of
  the nominal 2 s, probably because the server was busy answering `/metrics` under load.
