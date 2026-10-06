# 01 - Tune: thread-count sweep (GPU offload run, ngl=99)

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 27.1 | 97% |
| 2 | 27.8 | 99% |
| 4 | 27.9 | 100% |
| 8 | 27.8 | 99% |
| 16 | 28.0 | 100% |

**Best**: `-t 16` at 28.0 tok/s
**Slowest tested**: `-t 1` at 27.1 tok/s (1.03x spread)
**Against the physical-core default** (`-t 4`, 27.9 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=16 make bench
```

## Note

This is the GPU-offload run (`ngl=99`, default) of `make tune`, kept as the "before" for
REFLECTION section 5. The script writes the same filename for every run, so the later
CPU-only run (`LAB_N_GPU_LAYERS=0`) overwrote it; this file is the copy I saved just before
that. The numbers above are unedited script output. The explanation is in
`01-tuning-tg128.md`.
