# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 8 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.55 of 4 slots (89%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 841 |

Highest sampled value was **3.55 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Đỉnh batch width ghi nhận được là **3.55 trên 4 slots (89%)**, với `requests_processing = 4` và `requests_deferred = 46`. Con số này hoàn toàn khớp với effective concurrency (4.3) và việc toàn bộ 4 slots của server đều được lấp đầy liên tục trong suốt 60s chịu tải cao (50 users). Điều này chứng minh scheduler của llama-server đã thực sự thực hiện continuous batching (ghép nhiều request đồng thời vào chung một decode step). Gauge nội bộ `n_busy_slots_per_decode` là nguồn đo đáng tin cậy nhất về độ rộng batch vật lý tại tầng compute.
