# Bonus - GPU offload sweep

Host `Windows-AMD64` � backend(s) `vulkan` �
llama.cpp `b10488` � `threads=4` � metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 30.1 | 1.00x | 84% |
| 8 | 28.1 | 0.94x | 78% |
| 16 | 29.9 | 1.00x | 83% |
| 24 | 34.2 | 1.14x | 95% |
| 32 | 35.6 | 1.18x | 99% |
| 99 | 35.9 | 1.19x | 100% |

Best: `-ngl 99` at 35.9 tok/s
-- 1.19x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Full offload (`-ngl 99`) is best on this machine, 1.19x faster than CPU-only
(35.9 vs 30.1 tok/s). The curve climbs steadily from `-ngl 16` upward and never
turns over before `-ngl 99` -- it doesn't peak-and-drop at a partial value, it
just keeps improving until every layer is offloaded. That shape (monotonic
increase, no partial peak) is the signature of nothing running out: Qwen3.5
0.8B is tiny (~0.5 GB) next to the Iris Xe's 8083 MiB of shared VRAM, so the
whole model fits comfortably with room to spare -- there's no host<->device
bandwidth penalty from evicting/re-fetching weights, unlike what would happen
if the model were too big for VRAM. The one anomaly is `-ngl 8`, which dips
slightly below `-ngl 0` (28.1 vs 30.1) -- likely the overhead of splitting work
across host and device outweighs the benefit at that small an offload fraction,
before the GPU's throughput advantage takes over past `-ngl 16`.
