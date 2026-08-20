# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` � host `Windows-AMD64` � llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` � warm-up discarded
Completed requests: `Q4_K_M` 10/10 � `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 5296 | 961 / 1169 | 27.5 / 37.9 | 2683 / 3440 / 3440 | 36.3 |
| UD-Q2_K_XL | 0.39 | 12992 | 1242 / 1411 | 442.0 / 449.3 | 29091 / 29373 / 29373 | 2.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **15.78x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead � few cores, no GPU offload � the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

UD-Q2_K_XL chỉ nhỏ hơn Q4_K_M có 0.11 GB nhưng decode chậm hơn 15.78x (2.3 vs 36.3
tok/s) và load time gấp 2.45x. Trên máy này (Vulkan GPU offload, ngl=99), decode là
compute-bound trên GPU chứ không còn thuần memory-bandwidth-bound trên CPU, nên chi
phí dequantize thêm của định dạng 2-bit không được bù lại bằng lợi ích băng thông.
Kết luận: 2-bit không đáng dùng trên máy này -- Q4_K_M vừa nhanh hơn vừa chỉ nặng
hơn 0.11 GB, không đáng đánh đổi.
