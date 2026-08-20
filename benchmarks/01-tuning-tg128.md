# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 15.8 | 86% |
| 2 | 18.5 | 100% |
| 4 | 17.3 | 94% |
| 8 | 18.2 | 98% |
| 16 | 18.4 | 99% |

**Best**: `-t 2` at 18.5 tok/s
**Slowest tested**: `-t 1` at 15.8 tok/s (1.17x spread)
**Against the physical-core default** (`-t 4`, 17.3 tok/s): 1.07x

Use this in your run:

```bash
LAB_N_THREADS=2 make bench
```

## Your explanation

Điểm uốn (knee) đạt được tại `-t 2` với 18.5 tok/s (nhanh hơn 1.07x so với mặc định 4 physical cores `-t 4` ở 17.3 tok/s, và 1.17x so với `-t 1`). Quá trình autoregressive decode (`tg128`) bị giới hạn nghiêm ngặt bởi băng thông bộ nhớ (memory bandwidth bound). Trên kiến trúc APU AMD Ryzen Mobile chia sẻ băng thông RAM DDR4 với GPU, chỉ cần 2 thread là đã bão hòa gần như toàn bộ băng thông bus RAM và cache L3 khả dụng. Việc nâng lên 4 core vật lý hay 8/16 core logic không đem lại thêm thông lượng tính toán mà làm tăng overhead đồng bộ hóa (synchronization/lock contention) và tranh chấp kênh bộ nhớ giữa các lõi CPU.
