# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 4.00 of 4 slots (100%) |
| `requests_processing` | 4 |
| `requests_deferred` | 37 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 5015 |

Highest sampled value was **4.00 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Server lúc đo: `make serve` với `LAB_N_GPU_LAYERS=0 LAB_N_THREADS=8` (CPU-only, lý do ở
`01-tuning-tg128.md`).

Lưu ý về thời điểm: lần này `make metrics` bắt đầu khi `load-50` đã chạy được ~40 s, nên chỉ
**5 mẫu đầu (~20 s) trùng với tải**; từ mẫu thứ 6 locust đã dừng (`requests_processing = 0`).
Gauge `n_busy_slots_per_decode` là trung bình cộng dồn nên vẫn đọc 4.0 sau khi tải dừng — chỉ
5 mẫu đầu là bằng chứng. Một lần chạy trước đó với cùng cấu hình, trùng trọn 60 s, cho kết quả
giống: 4.00/4 slot ở mọi mẫu và `requests_deferred` 43–46.

**Batch width đạt trần: 4.00/4 slot trong cả 5 mẫu có tải**, với `requests_processing = 4` và
`requests_deferred` giảm dần 37 → 29 → 24 → 19 → 13 (locust đang tắt dần nên hàng đợi được
xả). Ở mẫu đầu có 4 + 37 = **41 request trong hệ thống** trên 50 user. Continuous batching hoạt
động: scheduler luôn nhét đủ 4 sequence vào mỗi bước decode.

**Không khớp với effective concurrency 18.3 trong `02-server-results.md`, và mình tin gauge hơn.**
Little's Law (`L = λ·W`) chỉ đúng ở steady state, và locust chỉ tính request **đã hoàn thành**.
Ở đây P95 latency (58 s) gần bằng độ dài run (60 s), nên nhiều request phát ra trong lúc chạy
chưa kịp xong và không được đếm; λ và W chỉ đo trên 38 request đã xong → L bị đánh giá thấp.
Gauge của server đọc trực tiếp số đang xử lý + đang chờ nên không có bias này.

Điều đáng chú ý: batch full không có nghĩa throughput cao. `llama-batched-bench` trên CPU
cho tổng decode ~56 tok/s ở B=1, ~61 tok/s ở B=4 và 43 tok/s ở B=8
(`01-tuning-cpu-vs-igpu.md`, run D). Với model 0.8B trên CPU, gộp 4 sequence gần như không tăng
tổng token/s — 4 slot chỉ chia nhau cùng một lượng compute. Vì vậy các request deferred là
thuần **queue time**.
