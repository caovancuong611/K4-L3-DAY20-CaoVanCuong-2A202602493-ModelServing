# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **10 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 40.9 | 92% |
| 5 | 42.9 | 97% |
| 10 | 43.7 | 99% |
| 16 | 44.2 | 100% |
| 32 | 42.9 | 97% |

**Best**: `-t 16` at 44.2 tok/s
**Slowest tested**: `-t 1` at 40.9 tok/s (1.08x spread)
**Against the physical-core default** (`-t 10`, 43.7 tok/s): 1.01x

Use this in your run:

```bash
LAB_N_THREADS=16 make bench
```

## Your explanation

**Không có knee — curve phẳng (spread chỉ 1.08×, trong mức nhiễu).** Lý do: sweep này chạy
với `ngl=99`, tức toàn bộ layer nằm trên **iGPU Intel UHD (Vulkan)**. CPU thread chỉ lo việc
điều phối (submit command buffer, sampling) nên `-t 1` và `-t 16` gần như như nhau. "Best = 16"
ở đây là nhiễu, không phải tín hiệu — mình **không** dùng `LAB_N_THREADS=16`.

Curve phẳng gợi ý thread count không phải knob đúng khi offload, nên mình chạy thêm
`llama-bench` với `-ngl 0` (chi tiết: [`01-tuning-cpu-vs-igpu.md`](01-tuning-cpu-vs-igpu.md)).
Ở chế độ CPU-only mới thấy hình dạng kỳ vọng:

| `-ngl 0 -t` | 1 | 4 | 6 | 8 | 10 | 12 | 16 |
|:--|--:|--:|--:|--:|--:|--:|--:|
| tg128 (tok/s) | 23.3 | 26.7 | 28.4 | **29.4** | 27.5 | 16.7 | 12.4 |

Knee ở **8 threads**, và vượt 10 (số physical core) thì sụp hẳn (−43% ở 12, −58% ở 16).
i7-13620H có 6 P-core (có HT) + 4 E-core. Mỗi bước decode là một chuỗi matmul nhỏ chia đều
cho các thread rồi **đợi ở barrier**, nên bước đó chậm bằng thread chậm nhất. Từ 12 thread trở
lên, các thread bắt đầu dùng chung một P-core qua hyperthreading (chia đôi execution unit và
L1/L2) hoặc nằm trên E-core chậm hơn → một vài thread chậm kéo cả bước. Thêm nữa, tăng từ 1→8
thread chỉ được 1.26×: model 0.8B quá nhỏ, phần việc mỗi token ít nên chi phí đồng bộ chiếm
tỉ lệ lớn.

Thay đổi thật sự quan trọng là **`ngl=99` → `ngl=0`**: CPU-only decode nhanh hơn iGPU
1.18–1.48× (xem REFLECTION §5). Phần serving (`02-*`) vì vậy chạy với `LAB_N_GPU_LAYERS=0
LAB_N_THREADS=8`.
