# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.8 | 17945.2 | 17946.2 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 12432.6 | 12432.7 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 13539.1 | 13539.3 |

Mean per stage (ms): embed **0.0** · retrieve **0.3** ·
llm **14639.0** · total **14639.4**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput is more useful than raw throughput because it filters out requests that fail to meet their specific Service Level Objectives (SLOs), such as those exceeding the Time-to-First-Response (TTFT) or Time-to-Poll (TPOT) targets. In contrast, raw throughput ignores SLOs entirely, as noted in the context. By focusing only on requests that satisfy these targets, Goodput provides a more accurate and

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** by storing the KV cache in non-contiguous pages.

This allows the engine to utilize more of the available GPU memory than a contiguous array would, as it avoids the wasted space required to align the data.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound** and **decode is memory-bandwidth-bound**.

This is because the context explicitly states that prefill is compute-bound and decode is memory-bandwidth-bound. By splitting them, the system allows the engine to skip prefill entirely for shared prefixes, thereby optimizing the memory bandwidth used during the decode step.


## Which N16-N19 pieces are real

- **N16 Cloud/IaC:** Stub (chạy Localhost Windows)
- **N17 Data pipeline:** Stub (In-memory dictionary)
- **N18 Lakehouse:** Stub (In-memory `TOY_DOCS`)
- **N19 Vector + features:** Stub (Keyword overlap matching fallback)
- **N20 Serving:** Real (`llama-server` OpenAI-compatible API trên `:8080`)

Stage chiếm ưu thế tuyệt đối là **LLM generation (100% thời gian, ~14.6s)**, hoàn toàn khớp với kỳ vọng vì retrieval trên tập document nhỏ diễn ra gần như tức thì (~0.3 ms). Nếu muốn giảm độ trễ pipeline 2×, ta phải tập trung tối ưu **LLM stage**: bật **Prompt Caching / Prefix Caching** cho system prompt và tài liệu RAG cố định (giúp triệt tiêu prefill time ~4-5s), đồng thời sử dụng quantization 2-bit hoặc tăng tốc độ decode để giảm thời gian sinh token.
