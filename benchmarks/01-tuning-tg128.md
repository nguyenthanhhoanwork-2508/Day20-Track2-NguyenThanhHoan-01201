# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` � host `Windows-AMD64` � llama.cpp `b10488`
CPU: **4 physical � 8 logical** cores � `ngl=99` � metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 33.4 | 90% |
| 2 | 35.7 | 96% |
| 4 | 31.5 | 85% |
| 8 | 37.0 | 100% |
| 16 | 28.0 | 76% |

**Best**: `-t 8` at 37.0 tok/s
**Slowest tested**: `-t 16` at 28.0 tok/s (1.32x spread)
**Against the physical-core default** (`-t 4`, 31.5 tok/s): 1.18x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

Peak is at -t 8, which is the logical core count (8), not the physical core count
(4) that the deck's default expectation assumes. This machine runs with GPU offload
via Vulkan (ngl=99), so most of the memory-bandwidth-bound decode work already runs
on the GPU. CPU threads are left handling lighter work (dispatch, sync with the GPU,
any non-offloaded layers), so the workload is less bandwidth-constrained on the CPU
side and hyperthreading still buys real parallelism instead of just contending for
the same memory channel. -t 16 is the slowest (28.0 tok/s, -24% vs peak) because it
oversubscribes past the 8 logical cores, forcing constant context-switching. This
contradicts the "peak at physical cores" shape from the deck, and the reason is the
GPU offload changing what the CPU threads are actually bottlenecked on.
