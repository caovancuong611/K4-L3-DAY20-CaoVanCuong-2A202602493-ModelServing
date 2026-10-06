# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=0` (server env: `LAB_N_GPU_LAYERS=0 LAB_N_THREADS=8`)

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 36 | 0.64 | 12000 | 20000 | 23000 | 8.3 | 0.0% |
| 50 | 38 | 0.64 | 25000 | 58000 | 58000 | 18.3 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.00x** (20% of linear) |
| P95 latency | **2.90x** |
| Effective concurrency at 50 users | 18.3 vs `--parallel 4` slots (occupancy/slot ratio 4.58) |

**Saturated.** Throughput delivered only 1.00x for 5x the offered load, and effective concurrency (18.3) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.00x while P95 moved 2.90x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Server lúc đo: `make serve` với `LAB_N_GPU_LAYERS=0 LAB_N_THREADS=8` (CPU-only, 8 thread).
Header phía trên mình sửa tay: `load-report.py` ghi `threads=10 ngl=99` vì nó đọc env của chính nó, không phải
của process server.

**Server đã bão hoà ngay từ 10 users.** Con số thuyết phục mình: throughput đứng yên hoàn toàn
(0.64 → 0.64 RPS, **1.00×** cho 5× tải), và ngay ở 10 users effective concurrency đã là 8.3 —
gấp đôi 4 slot. `make metrics` trong lúc 50 users cho thấy 4/4 slot bận và **37 request
deferred**. Capacity của máy này là ~0.64 RPS: `tokens_predicted_total` tăng 4540 → 4884 trong
12 s ≈ **28 tok/s tổng**, cùng cỡ với 0.64 RPS × ~55 token/request ≈ 35 tok/s.

**Phần latency tăng thêm là queue time, không phải compute.** Áp Little's Law riêng cho 4 slot
(luôn đầy theo gauge): thời gian một request nằm trong slot ≈ 4 / 0.64 ≈ **6.2 s**. Ở 10 users
P50 = 12 s ≈ 6 s compute + 6 s chờ; ở 50 users P95 = 58 s, tức ~52 s là chờ slot. Compute mỗi
request không đổi giữa hai mức tải — chỉ hàng đợi dài ra. (Effective concurrency 18.3 ở 50 users
vẫn thấp hơn ~41 request thật sự trong hệ thống theo gauge, vì run 60 s ngắn hơn thời gian sống
của request; xem `02-server-batching-u50.md`.)

**Goodput@SLO.** Chọn SLO = P95 end-to-end ≤ 30 s (rộng, vì đây là laptop CPU-only). Ở 10 users
P95 = 20 s → đạt, goodput ≈ 0.64 RPS. Ở 50 users chỉ khoảng 55–60% request xong dưới 30 s
(P50 25 s, P66 39 s) → goodput ≈ 0.37 RPS: **thêm tải làm goodput giảm** dù throughput giữ nguyên.

**Knob mình đổi trước: giới hạn hàng đợi (admission control), không phải `--parallel`.**
`llama-batched-bench` trên CPU cho tổng decode ~56 tok/s ở B=1, ~61 ở B=4 và 43 ở B=8
(`01-tuning-cpu-vs-igpu.md`, run D) — thêm slot không tăng capacity, chỉ chia nhỏ cùng
lượng compute và làm mỗi request chậm hơn. Vì capacity cố định ~0.64 RPS, cách duy nhất giữ
request được nhận trong SLO là chặn bớt: với ~6 s/request và SLO 30 s, mỗi slot chỉ chịu được
~4 request xếp hàng → giới hạn ~16–20 request trong hệ thống, quá thì trả 429 ngay. Knob thứ hai
(tăng capacity thật): giảm số token mỗi request (cap `max_tokens`) hoặc tách load generator
(locust 50 users) ra máy khác — hiện nó tranh CPU với chính server.

Ghi chú về độ lặp lại: một lần chạy trước đó với cùng cấu hình cho 0.44 / 0.45 RPS và P95
29 s / 56 s — mức capacity tuyệt đối dao động giữa các lần chạy (nhiệt/nguồn của laptop), nhưng
hình dạng giống hệt: throughput ~1.0× trong khi P95 tăng 2–3×.
