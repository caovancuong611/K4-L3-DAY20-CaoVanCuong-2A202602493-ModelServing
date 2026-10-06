# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 11466.9 | 11467.0 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.2 | 7360.0 | 7360.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.2 | 7175.0 | 7175.3 |

Mean per stage (ms): embed **0.0** · retrieve **0.2** ·
llm **8667.3** · total **8667.5**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it focuses on the specific metrics that directly impact system stability and reliability:

1.  **Accurate SLO Monitoring**: Goodput counts only requests per second that met the Target Time-to-Functional-Throughput (TTFT) and Target Time-to-Functional-Throughput-Overhead (TPOT) targets. This ensures that the syste

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory**.

While the context explicitly states that PagedAttention "stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory," it also notes that this approach removes the internal fragmentation caused by the **RadixAttention** mechanism (which uses a shared prefix to skip prefi

**When does splitting prefill and decode help?**

> Based on the provided context, splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bandwidth-bound**.

This is because the context states that "Disaggregated serving splits prefill and decode onto separate pools because prefill is compute-bound and decode is memory-bandwidth-bound." The solution involves using a radix-based approach where keys are cached by toke


## Which N16-N19 pieces are real

Server lúc chạy: `llama-server` CPU-only (`LAB_N_GPU_LAYERS=0 LAB_N_THREADS=8`), chạy ngay sau
`load-50` khi server đã rảnh.

| Day | Piece | Real hay stub? |
|:--|:--|:--|
| N16 Cloud/IaC | — | **stub**: không có hạ tầng cloud, mọi thứ chạy local trên laptop |
| N17 Data pipeline | — | **stub**: corpus là `TOY_DOCS` hard-code trong `pipeline.py` (STUB 1) |
| N18 Lakehouse | — | **stub**: không có lakehouse, doc nằm trong bộ nhớ |
| N19 Vector + features | `retrieve()` | **stub**: keyword overlap (STUB 2), không có embedding server → embed = 0.0 ms |
| N20 Serving | `llama-server` | **real** |

**Dominant stage: llm, 100% — đúng như kỳ vọng, nhưng con số 0% của embed/retrieve là do stub,
không phải vì retrieval miễn phí.** Keyword overlap trên 6 doc chỉ tốn 0.2 ms; một vector search
thật (embedding query + ANN) sẽ tốn vài chục ms — vẫn nhỏ so với 8.7 s của LLM.

Tách stage LLM theo server timings:

| Query | llm (ms) | prefill | decode | còn lại (ngoài server timings) |
|:--|--:|--:|--:|--:|
| 1 | 11467 | 151 tok / 1986 ms (17%) | 196 tok / 7291 ms (64%) | 2190 ms (19%) |
| 2 | 7360 | 114 tok / 1051 ms (14%) | 119 tok / 4105 ms (56%) | 2204 ms (30%) |
| 3 | 7175 | 113 tok / 1340 ms (19%) | 94 tok / 3599 ms (50%) | 2236 ms (31%) |

Decode là phần lớn nhất (50–64%), nhưng có một khoản **~2.2 s gần như hằng số** mỗi query không
nằm trong prefill/decode của server. Vì nó không đổi theo độ dài prompt hay câu trả lời, mình đoán
đó là chi phí cố định ngoài model (HTTP client, chat template, chờ slot được giải phóng sau
`load-50`) — mình chưa kiểm chứng nguyên nhân.

**Nếu phải giảm latency 2×, cắt độ dài câu trả lời là chưa đủ — phải xử lý cả hai khoản.**
Model trả lời dài, có tiêu đề và gạch đầu dòng (94–196 token) cho các câu chỉ cần 1–2 câu
(`max_tokens` trong `pipeline.py` là 200). Cap ~60 token + system prompt "trả lời trong 2 câu"
cắt decode còn ~2–2.5 s; cộng với bỏ được ~2.2 s overhead cố định (tìm nguyên nhân bằng cách đo
timestamp phía client so với `timings` của server) thì từ ~8.7 s xuống còn ~4 s, tức ~2×.
Prefill (1–2 s) là mục tiêu sau: context RAG lặp lại nên prompt caching cho prefix chung sẽ
giúp khi corpus lớn hơn.
