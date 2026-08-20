# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Hữu Nhật Minh
**Cohort:** 2A202601551 (AICB-P2T2)
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (AMD64)
- **CPU:** AMD Ryzen 5 3450U with Radeon Vega Mobile Gfx
- **Cores:** 4 physical · 8 logical
- **CPU extensions:** AVX2
- **RAM:** 5.9 GB
- **Accelerator:** Vulkan (AMD Radeon Graphics)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): Máy có 5.9 GB RAM (< 8 GB) nên script tự động chọn model Qwen3.5 0.8B (~0.9 GB). Sử dụng script PowerShell `lab.ps1` và bật `PYTHONUTF8=1` để tương thích Windows console. Tải runtime Vulkan prebuilt b10488 và tải 2 bản quant hoàn toàn tự động.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 12407 | 5136 / 5542 | 51.9 / 55.8 | 8272 / 9054 / 9054 | 19.3 |
| UD-Q2_K_XL | 0.39 | 11872 | 4963 / 5252 | 47.8 / 51.7 | 7994 / 8511 / 8511 | 20.9 |

**Quan sát** (≤ 60 chữ): Bản 2-bit nhẹ hơn 22% (0.39 GB vs 0.50 GB), decode nhanh hơn 1.08x (20.9 vs 19.3 tok/s), TTFT giảm ~173 ms do giảm tải bus RAM. Với model 0.8B, bản Q4 cho chất lượng và độ mạch lạc cú pháp tốt hơn đáng kể.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.14 | 44000 | 51000 | 51000 | 5.1 | 0.0% |
| 50 | 0.13 | 39000 | 45000 | 45000 | 4.3 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.97×
- **P95 tăng:** 0.88×
- **Effective concurrency ở 50 users:** 4.3 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.55 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa hoàn toàn ở 50 users khi RPS đi ngang (0.13), cả 4 slot đều bận (`processing=4`) và 46 request bị hoãn (`deferred=46`). Độ trễ tăng thêm là queue time. Muốn tăng goodput@SLO, tôi sẽ đổi knob `LAB_N_THREADS=2` hoặc hạ quant sang 2-bit để giải phóng slot nhanh hơn.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Localhost (Windows) | stub |
| N17 Data pipeline | In-memory dictionary | stub |
| N18 Lakehouse | `TOY_DOCS` | stub |
| N19 Vector + features | Keyword overlap fallback | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.3 ms
- llm: 14639.0 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection** (≤ 60 chữ): LLM chiếm 100% thời gian do retrieval trên tập nhỏ diễn ra tức thì (~0.3ms). Để giảm độ trễ pipeline 2x, cần áp dụng Prompt Caching / Prefix Caching cho context RAG và tối ưu tốc độ decode của LLM.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Điều chỉnh số thread `-t` từ 1 lên 2 threads trên CPU Ryzen 5 3450U

```
before:  15.8 tok/s (-t 1)
after:   18.5 tok/s (-t 2)
speedup: 1.17×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Quá trình autoregressive decode (`tg128`) là bài toán bị giới hạn bởi băng thông bộ nhớ (memory-bandwidth-bound) thay vì số phép tính FLOPs thuần túy. Trên kiến trúc APU AMD Ryzen laptop dùng chung băng thông RAM DDR4 với GPU tích hợp, việc tăng từ 1 lên 2 thread giúp tận dụng tối đa kênh bộ nhớ (memory bus) và cache L3 để fetch trọng số mô hình liên tục.

Tuy nhiên, khi tăng tiếp lên 4 physical cores (17.3 tok/s) hay 8/16 logical cores (18.2 - 18.4 tok/s), băng thông RAM đã bị bão hòa hoàn toàn. Các thread vượt mức 2 bắt đầu tranh chấp cùng một bus dữ liệu và gây ra overhead chuyển ngữ cảnh (context switching / synchronization contention) khiến tốc độ đi ngang hoặc thậm chí sụt giảm so với `-t 2`. Vì vậy, điểm uốn (knee) tối ưu nhất của hệ thống đạt được chính xác tại `-t 2`.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** 

**Numbers:**

```
before:  
after:   
speedup: 
```

**Điều này nói lên gì mà deck chưa nói:**



---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điểm thú vị nhất là cơ chế Continuous Batching thực sự hoạt động rõ nét trên máy cá nhân: thông số `n_busy_slots_per_decode` đạt tới 3.55/4 slot khi có tải lớn, giúp hệ thống phục vụ được nhiều user cùng lúc mà compute latency của từng token không bị nhân tuyến tính theo số người dùng.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

