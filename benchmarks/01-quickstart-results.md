# 01 - Measure: latency baseline

Model Qwen3.5 0.8B - host Windows-AMD64 - llama.cpp 10488
Settings: 	hreads=4 
gl=99 ctx=2048
max_tokens=64 - warm-up discarded
Completed requests: Q4_K_M 10/10 - UD-Q2_K_XL 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 12407 | 5136 / 5542 | 51.9 / 55.8 | 8272 / 9054 / 9054 | 19.3 |
| UD-Q2_K_XL | 0.39 | 11872 | 4963 / 5252 | 47.8 / 51.7 | 7994 / 8511 / 8511 | 20.9 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. decode tok/s = 1000 / TPOT_p50.
- UD-Q2_K_XL decodes **1.08x faster** than Q4_K_M here, for 0.11 GB less on disk.

## Your observation

Bản UD-Q2_K_XL (2-bit) nhỏ hơn 22% (0.39 GB so với 0.50 GB của Q4_K_M), tốc độ decode nhanh hơn ~1.08x (20.9 tok/s so với 19.3 tok/s), đồng thời TTFT P50 giảm từ 5136 ms xuống 4963 ms do lượng dữ liệu trọng số cần truyền tải qua bus RAM ít hơn. Đối với model 0.8B, sự chênh lệch chất lượng giữa Q4 và UD-Q2 là có nhưng chấp nhận được với các prompt ngắn; tuy nhiên với model siêu nhỏ như 0.8B, Q4_K_M vẫn là lựa chọn cân bằng hơn cho độ chính xác cú pháp trừ khi RAM bị giới hạn cực độ.
