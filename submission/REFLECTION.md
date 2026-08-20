# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Thanh Hoàn - 01201
**Cohort:** A20-K3
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (build 10.0.26200)
- **CPU:** Intel Core i5-11300H @ 3.10GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** không đo trong `hardware.json`, CPU thế hệ 11 hỗ trợ AVX2
- **RAM:** 15.8 GB
- **Accelerator:** Vulkan (Intel Iris Xe Graphics, 8083 MiB, GPU offload ACTIVE)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare)

**Chạy ở đâu:** laptop của tôi (local, không dùng cloud fallback)

**Setup story** (≤ 80 chữ): Đổi sang Qwen3.5 0.8B vì file Gemma 4 tải trước đó không
đúng quant lab yêu cầu (repo `ggml-org` Q8_0 thay vì `unsloth` UD-Q4/UD-Q2). `lab.ps1`
cũng lỗi parse trên Windows PowerShell 5.1 do ký tự em-dash UTF-8 không BOM làm vỡ
string; sửa bằng cách ghi lại file với UTF-8 BOM.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 5296 | 961 / 1169 | 27.5 / 37.9 | 2683 / 3440 / 3440 | 36.3 |
| UD-Q2_K_XL | 0.39 | 12992 | 1242 / 1411 | 442.0 / 449.3 | 29091 / 29373 / 29373 | 2.3 |

**Quan sát** (≤ 60 chữ): UD-Q2_K_XL chỉ nhỏ hơn 0.11 GB nhưng decode chậm hơn **15.78x**
(2.3 vs 36.3 tok/s). Trên máy này GPU offload (Vulkan) làm decode compute-bound thay vì
memory-bandwidth-bound, nên dequant overhead của quant 2-bit ăn hết lợi ích băng thông —
2-bit **không đáng dùng** ở đây dù nhẹ hơn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.14 | 42000 | 59000 | 59000 | 4.7 | 0.0% |
| 50 | 0.21 | 35000 | 53000 | 53000 | 7.4 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.52×
- **P95 tăng:** 0.90× (giảm nhẹ, không tăng)
- **Effective concurrency ở 50 users:** 7.4 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.67 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hoà ở hoặc trước mốc 50 users — bằng chứng
là throughput chỉ tăng 1.52× cho 5× tải, và effective concurrency (7.4) vượt cả 4 slot
decode. `requests_deferred=46` xác nhận có hàng chờ thật. P95 không tăng nhanh hơn RPS
nên vẫn còn headroom; muốn nâng goodput@SLO tôi sẽ tăng `--parallel` trước (nhiều slot
hơn) vì bottleneck là số slot đồng thời, không phải compute per-token.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | — | không dùng trong lab này |
| N17 Data pipeline | — | không dùng trong lab này |
| N18 Lakehouse | — | không dùng trong lab này |
| N19 Vector + features | retrieval | **stub** (keyword overlap, không phải embedding thật) |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 6076.6 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): LLM chiếm gần như toàn bộ latency, đúng như kỳ vọng vì
retrieval ở đây là stub (keyword overlap, gần như tức thời) chứ không phải embedding
vector thật. Nếu phải giảm latency 2×, tôi sẽ tấn công stage llm — cụ thể là decode,
bằng cách dùng quant nhanh hơn hoặc giảm `max_tokens`, vì embed/retrieve gần như không
đóng góp gì vào tổng.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** tăng `-t` (số thread CPU) từ 16 xuống best value là 8

```
before:  -t 16 → 28.0 tok/s (worst tested)
after:   -t 8  → 37.0 tok/s (best tested)
speedup: 1.32×
```

Full sweep: `-t 1` = 33.4, `-t 2` = 35.7, `-t 4` = 31.5, `-t 8` = 37.0 (best), `-t 16` = 28.0.

**Tại sao nó work** (1–2 đoạn):

Đỉnh nằm ở `-t 8` — bằng số **logical core** (8), không phải physical core (4) như deck
thường kỳ vọng. Lý do: máy này chạy với GPU offload qua Vulkan (`ngl=99`), nên phần lớn
việc decode (matrix-vector multiply, vốn memory-bandwidth-bound) đã được đẩy sang GPU.
Threads CPU trong trường hợp này chỉ còn lo phần nhẹ hơn (dispatch, phần layer chưa
offload, I/O đồng bộ với GPU), nên bài toán bớt bị giới hạn bởi băng thông bộ nhớ CPU và
hyperthreading vẫn còn chỗ sinh lợi — 8 thread logic tận dụng được các core ảo mà không
tranh chấp cache dữ liệu nặng như khi CPU phải tự làm toàn bộ decode.

`-t 16` chậm nhất (28.0 tok/s, giảm 24% so với đỉnh) vì oversubscribe: số thread vượt cả
số logical core thật (8), buộc OS phải context-switch liên tục giữa các thread tranh
cùng core vật lý, sinh overhead lớn hơn lợi ích song song hoá. `-t 4` (bằng physical core)
lại thấp hơn cả `-t 1` và `-t 2` trong lần đo này (31.5 tok/s) — kết quả hơi nhiễu do máy
chạy nền có tiến trình khác (bench trước đó, terminal) chiếm core, nhưng xu hướng chung
vẫn rõ: đỉnh ở 8, sụp ở 16.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
