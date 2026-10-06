# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Cao Văn Cường
**MSSV:** 2A202602493
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime _(rubric 1, 2 — 10 điểm)_

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 Home Single Language (10.0.26200), Python 3.14.7
- **CPU:** Intel Core i7-13620H (Raptor Lake-H: 6 P-core + 4 E-core)
- **Cores:** 10 physical / 16 logical
- **CPU extensions:** AVX2 + AVX-VNNI, không có AVX-512 (llama.cpp chọn backend `ggml-cpu-alderlake`)
- **RAM:** 15.7 GB
- **Accelerator:** Intel UHD Graphics (iGPU, dùng chung RAM) qua Vulkan — không có GPU rời
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** `Q4_K_M` (primary, 0.50 GB) + `UD-Q2_K_XL` (compare, 0.39 GB) (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (local, không dùng Colab/Kaggle).

**Setup story** (≤ 80 chữ):

Setup chạy được nguyên bản trên Windows bằng `.\lab.ps1`. Hai chỗ phải sửa: report
`benchmarks/*.md` bị ghi bằng cp1252 (mặc định Windows) và `verify.py` đọc file cũng bằng
cp1252 nên crash với tiếng Việt — sửa thành UTF-8 trong `lib/labkit.py` và `scripts/verify.py`.
Probe chọn `ngl=99` vì thấy Vulkan, nhưng thiết bị đó là iGPU Intel UHD; đo xong mình chuyển
phần serving sang CPU-only (§5).

---

## 2. Đo lường _(rubric 3, 4, 5 — 20 điểm)_

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).
> Chạy với `threads=10 ngl=99 ctx=2048 max_tokens=64`.

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
| ------------ | --------: | --------: | ----------------: | ----------------: | -------------------: | -------------: |
| Q4_K_M       |      0.50 |      7310 |         337 / 454 |       24.2 / 24.7 |   1830 / 1978 / 1978 |           41.4 |
| UD-Q2_K_XL   |      0.39 |      4664 |         438 / 473 |       46.1 / 50.9 |   3339 / 3652 / 3652 |           21.7 |

**Quan sát** (≤ 60 chữ):

2-bit nhỏ hơn 22% nhưng decode **chậm hơn 1.91×** (21.7 vs 41.4 tok/s), TTFT cũng chậm hơn
30% — **không đáng dùng**. Máy mình compute-bound chứ không bandwidth-bound, nên bớt byte không
giúp mà dequant 2-bit tốn thêm ALU. Hỏi cùng 3 câu (`temperature=0`): cả hai đều sai phép tính
17×23, nhưng Q2 còn lặp lại câu và bỏ qua yêu cầu "một câu".

---

## 3. Serving under load _(rubric 8, 9, 10 — 20 điểm)_

> Từ `benchmarks/02-server-results.md` (`make load-report`).
> Server: `LAB_N_GPU_LAYERS=0 LAB_N_THREADS=8`, `--parallel 4`, `ctx=2048`.

| Users |  RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
| ----: | ---: | -------: | -------: | -------: | ---------------: | -------: |
|    10 | 0.64 |    12000 |    20000 |    23000 |              8.3 |     0.0% |
|    50 | 0.64 |    25000 |    58000 |    58000 |             18.3 |     0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.00×
- **P95 tăng:** 2.90×
- **Effective concurrency ở 50 users:** 18.3 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 4.00 / 4 slots (kèm `requests_deferred` tới 37)

**Saturation reading** (≤ 80 chữ):

Bão hoà ngay từ **10 users**: effective concurrency 8.3 đã gấp đôi 4 slot, và 5× tải chỉ cho
1.00× RPS. Phần P95 tăng thêm là **queue time**: 4 slot luôn đầy nên thời gian trong slot
≈ 4 / 0.64 ≈ 6.2 s ở cả hai mức tải; ~52 s còn lại của P95 = 58 s là chờ (37 request deferred).
Knob đổi trước: **giới hạn hàng đợi (~16–20 request) chứ không tăng `--parallel`** —
batched-bench cho thấy thêm batch không tăng tổng tok/s trên CPU này.

---

## 4. Integration _(rubric 12, 13 — 15 điểm)_

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day                   | Piece                                         | Real hay stub? |
| --------------------- | --------------------------------------------- | -------------- |
| N16 Cloud/IaC         | không có, chạy local                          | stub           |
| N17 Data pipeline     | `TOY_DOCS` hard-code (6 doc)                  | stub           |
| N18 Lakehouse         | không có, doc nằm trong bộ nhớ                | stub           |
| N19 Vector + features | `retrieve()` keyword overlap, không embedding | stub           |
| N20 Serving           | `llama-server`                                | real           |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms (không có embedding server)
- retrieve: 0.2 ms
- llm: 8667.3 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ):

Bottleneck là LLM, đúng kỳ vọng — nhưng 0% của retrieve là do stub. Trong LLM: decode 50–64%,
prefill 14–19%, và ~2.2 s cố định nằm ngoài server timings. Để giảm 2×: cap câu trả lời
(~60 token thay vì 94–196) và tìm/bỏ khoản overhead 2.2 s → ~8.7 s xuống ~4 s.

---

## 5. The single change that mattered most _(rubric 11 — 10 điểm)_

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** tắt offload lên iGPU — `ngl=99` (Intel UHD qua Vulkan) → `ngl=0` (CPU-only),
cùng Q4_K_M. Phát hiện ra vì `make tune` cho curve thread phẳng (1.08× spread).

```
decode, llama-bench tg128, -t 10 (run A, 01-tuning-cpu-vs-igpu.md):
before:  40.17 tok/s   (ngl=99, iGPU)
after:   59.38 tok/s   (ngl=0, CPU)
speedup: 1.48×

kiểm chứng: cặp đo xen kẽ -t 8 -r 5, lặp 2 lần (run C):  1.18× và 1.32×

end-to-end, make load-10 (10 users, 60 s):
before:  0.18 RPS, 9 request xong, P50 31 s   (ngl=99, threads=10)
after:   0.64 RPS, 36 request xong, P50 12 s  (ngl=0, threads=8)
speedup: 3.6× throughput  (một lần chạy CPU khác: 0.44 RPS → 2.4×)
```

(Số "before" của load-10 lấy từ output console; file `locust-10_stats.csv` của lần đó đã bị
các lần chạy CPU ghi đè. File hiện tại trong repo là lần chạy "after" 0.64 RPS.)

**Tại sao nó work:**

Deck nói "đưa lên GPU thì nhanh hơn", nhưng điều đó đúng khi GPU có **bộ nhớ riêng nhanh hơn**.
Intel UHD là iGPU: nó đọc weight từ **chính DDR5 mà CPU dùng**, nên với decode — mỗi token phải
đọc toàn bộ ~0.5 GB weight — iGPU không có lợi thế băng thông nào. Ở 40–60 tok/s mình chỉ dùng
~20–30 GB/s, chưa chạm trần RAM, nên decode ở đây không bị chặn bởi bandwidth mà bởi
**chi phí cố định mỗi bước**. Một token của model 0.8B là vài trăm phép mat-vec rất nhỏ; trên
iGPU mỗi phép là một lần dispatch Vulkan có độ trễ riêng và đồng bộ với CPU, còn trên CPU nó là
vòng lặp SIMD (AVX2/VNNI) chạy ngay trong cache. Việc càng nhỏ, overhead dispatch càng chiếm phần
lớn → CPU thắng. Cùng logic giải thích vì sao Q2 chậm hơn Q4 (§2): khi không bandwidth-bound,
bớt byte vô ích còn dequant thêm thì tốn.

Kết quả ngược lại ở **prefill** khẳng định cơ chế này: pp64 trên iGPU đạt 470 tok/s so với
127 tok/s trên CPU (3.7×), vì prefill là mat-mat lớn — đủ việc để che overhead dispatch và tận
dụng nhiều EU song song. Vậy lựa chọn đúng phụ thuộc workload: chat ngắn, decode nhiều → CPU;
RAG prompt dài, prefill nhiều → iGPU có thể thắng. Dưới tải, CPU càng thắng rõ: với 4 sequence song song, iGPU chỉ
decode ~23 tok/s tổng, CPU được 32–61 tok/s (run D). Kèm theo đó, trên CPU thread count
mới bắt đầu có ý nghĩa: knee ở 8 thread, còn 12–16 thread sụp còn 12–17 tok/s vì mỗi bước decode
phải đợi thread chậm nhất ở barrier, và từ 12 thread trở đi các thread chia nhau P-core qua
hyperthreading hoặc nằm trên E-core — nên mình serve với `-t 8`.

---

## 6. Bonus _(optional — tối đa 10 điểm)_

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _(chưa làm)_

**Numbers:**

```
before:  —
after:   —
speedup: —
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất _(optional)_

"GPU offload" làm decode chậm hơn, và continuous batching chạy đủ 4/4 slot mà gần như không tăng
tổng tok/s: trên laptop không có GPU rời, slot thêm chỉ chia nhau cùng một lượng compute.

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
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI _(xem `docs/RULES.md` §3)_

Claude Code (Claude Opus 5.5): chạy các lệnh lab trên máy tôi, chạy thêm `llama-bench` /
`llama-batched-bench` để so CPU vs iGPU, sửa lỗi encoding UTF-8 trong `lib/labkit.py` và
`scripts/verify.py`, và viết nháp phần nhận xét trong `benchmarks/*.md` và REFLECTION. Tôi đã
đọc lại, đối chiếu số liệu với `benchmarks/*.md` và chịu trách nhiệm về phần lập luận.
