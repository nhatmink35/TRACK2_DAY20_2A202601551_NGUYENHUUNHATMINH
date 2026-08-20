# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 7 | 0.14 | 44000 | 51000 | 51000 | 5.1 | 0.0% |
| 50 | 6 | 0.13 | 39000 | 45000 | 45000 | 4.3 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.97x** (19% of linear) |
| P95 latency | **0.88x** |
| Effective concurrency at 50 users | 4.3 vs `--parallel 4` slots (occupancy/slot ratio 1.07) |

**Saturated.** Throughput delivered only 0.97x for 5x the offered load, and effective concurrency (4.3) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (0.88x vs 0.97x), so this server still has headroom at 50 users.

> **Small sample.** Only 6 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

Server bão hòa hoàn toàn ở mức 10-50 users. Bằng chứng rõ ràng nhất là: khi offered load tăng gấp 5x (10 lên 50 users), RPS thực tế hoàn toàn đi ngang ở mức ~0.13 - 0.14 req/s (chỉ đạt 0.97x), trong khi `requests_processing = 4` chạm trần toàn bộ 4 slots của `--parallel 4` và `requests_deferred = 46` request phải xếp hàng chờ. Toàn bộ độ trễ tăng thêm cho người dùng đến từ **queue time (thời gian chờ slot)** chứ không phải compute time.

Nếu cần nâng **goodput@SLO**, knob tôi sẽ đổi **đầu tiên** là **giảm quantization sang 2-bit (`UD-Q2_K_XL`)** hoặc đặt **`LAB_N_THREADS=2`** để tăng tốc độ decode mỗi token (từ 15.8 lên 18.5+ tok/s). Lý do: trên máy bị nghẽn băng thông phần cứng, việc rút ngắn thời gian chiếm giữ slot của mỗi request là cách trực tiếp nhất giúp giải phóng slot nhanh hơn, giảm triệt để thời gian xếp hàng trong queue cho các request tiếp theo mà không làm tràn RAM/VRAM.
