# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=10` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 7310 | 337 / 454 | 24.2 / 24.7 | 1830 / 1978 / 1978 | 41.4 |
| UD-Q2_K_XL | 0.39 | 4664 | 438 / 473 | 46.1 / 50.9 | 3339 / 3652 / 3652 | 21.7 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.91x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

**2-bit không đáng dùng trên máy này.** `UD-Q2_K_XL` nhỏ hơn 22% (0.39 vs 0.50 GB) nhưng
decode **chậm hơn 1.91×** (21.7 vs 41.4 tok/s, TPOT P50 46.1 vs 24.2 ms) và TTFT P50 cũng
chậm hơn 30% (438 vs 337 ms). Thứ duy nhất nó thắng là thời gian load (4.7 vs 7.3 s).

Vì sao chậm hơn: bench này chạy với `ngl=99` trên **Intel UHD (iGPU) qua Vulkan**. iGPU dùng
chung DDR5 với CPU, và với model 0.5 GB thì mỗi token chỉ đọc ~0.5 GB weight — ~20 GB/s ở
41 tok/s, còn xa giới hạn băng thông RAM. Nghĩa là máy mình ở trường hợp **compute/overhead-bound**,
không phải bandwidth-bound: bớt byte không giúp gì, còn kernel dequant Q2_K/IQ (nhiều phép bit
hơn mỗi weight) thì tốn thêm ALU. `llama-bench` xác nhận điều này không chỉ đúng với iGPU: ở
`ngl=0` (CPU), Q2 cũng chỉ đạt 30.8 tok/s so với 59.4 của Q4 (xem `01-tuning-cpu-vs-igpu.md`, run A).

Chất lượng: mình hỏi cùng 3 câu cho cả hai bản (`temperature=0`, server `--compare` ở port 8090):

| Câu hỏi | Q4_K_M | UD-Q2_K_XL |
|:--|:--|:--|
| `17 * 23`, chỉ trả số | `521` (sai) | `361` (sai) |
| Giải thích decode bị chặn bởi memory bandwidth trong 2 câu | sai cơ chế nhưng đúng 2 đoạn | sai cơ chế, **lặp lại** chính ý đó lần 2, bị cắt giữa chừng |
| Một câu tiếng Việt về continuous batching | trả lời 2 câu, mạch lạc (dù sai nghĩa) | bỏ qua yêu cầu "một câu", mở đầu bằng lời chào, viết dài |

Cả hai đều yếu về kiến thức (0.8B là 0.8B), nhưng Q2 còn mất khả năng làm theo chỉ dẫn và bắt
đầu lặp. Kết luận: trên máy này 2-bit **vừa chậm hơn vừa kém hơn** — chỉ đáng cân nhắc khi bị giới
hạn RAM/disk (ví dụ máy 4 GB chạy Gemma 4 E2B), không phải để tăng tốc.
