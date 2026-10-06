# 01 - Tune (supplementary): CPU-only vs Intel iGPU offload

Raw `llama-bench` output, run by hand after `make tune` showed a flat thread curve.
Model `Qwen3.5-0.8B` · host `Windows-AMD64` (i7-13620H, 6P+4E = 10 cores / 16 threads,
Intel UHD Graphics via Vulkan) · llama.cpp `b10488` · laptop on AC, Balanced power plan.

Command shape: `runtime/b10488/llama-bench.exe -m <model> -ngl <0|99> -t <n> -p 64 -n 128 -r <reps>`

## Run A — offload vs CPU, both quantizations (`-t 10`, `-r 2`)

| model | ngl | test | t/s |
|:--|--:|:--|--:|
| Q4_K_M | 0 | pp64 | 127.36 ± 28.39 |
| Q4_K_M | 0 | tg128 | 59.38 ± 1.05 |
| Q4_K_M | 99 | pp64 | 469.76 ± 0.49 |
| Q4_K_M | 99 | tg128 | 40.17 ± 4.33 |
| UD-Q2_K_XL | 0 | pp64 | 90.35 ± 3.76 |
| UD-Q2_K_XL | 0 | tg128 | 30.76 ± 0.19 |
| UD-Q2_K_XL | 99 | pp64 | 222.30 ± 10.03 |
| UD-Q2_K_XL | 99 | tg128 | 15.98 ± 1.75 |

## Run B — CPU-only thread sweep (`-ngl 0`, Q4_K_M, `-r 3`)

| threads | tg128 (t/s) |
|--:|--:|
| 1 | 23.25 ± 3.33 |
| 4 | 26.66 ± 5.16 |
| 6 | 28.44 ± 3.44 |
| 8 | 29.35 ± 0.45 |
| 10 | 27.54 ± 0.66 |
| 12 | 16.71 ± 4.50 |
| 16 | 12.37 ± 4.10 |

Same session, `-ngl 99 -t 10`: 26.04 ± 4.39 t/s. Absolute numbers in run B are lower than
run A across the board (including the iGPU row), so the machine was in a slower
thermal/power state; only compare rows within one run.

## Run C — interleaved pair, back to back (`-t 8`, Q4_K_M, `-r 5`, repeated twice)

| pass | ngl=99 (iGPU) tg128 | ngl=0 (CPU) tg128 | CPU / iGPU |
|--:|--:|--:|--:|
| 1 | 28.65 ± 9.42 | 33.90 ± 5.60 | 1.18x |
| 2 | 27.38 ± 3.74 | 36.21 ± 3.88 | 1.32x |

## Run D — does batching raise total decode throughput? (`llama-batched-bench`)

`llama-batched-bench.exe -m Qwen3.5-0.8B-Q4_K_M.gguf -ngl 0 -t 8 -c 8192 -npp 32 -ntg 48 -npl 1,1,2,4,8`
(B = number of parallel sequences; S_TG = total decode tokens/s across all of them)

| B | T_PP (s) | S_PP (t/s) | T_TG (s) | S_TG (t/s) | total S (t/s) |
|--:|--:|--:|--:|--:|--:|
| 1 | 0.601 | 53.24 | 0.839 | 57.24 | 55.57 |
| 1 | 0.483 | 66.20 | 0.873 | 54.97 | 58.97 |
| 2 | 0.548 | 116.74 | 1.979 | 48.50 | 63.31 |
| 4 | 1.065 | 120.16 | 3.171 | 60.55 | 75.54 |
| 8 | 2.300 | 111.32 | 8.918 | 43.06 | 57.05 |

Earlier, same tool, `-npl 1,4`, comparing offload (`-c 4096`, `-t 8`):

| ngl | B | S_TG (t/s) | total S (t/s) |
|--:|--:|--:|--:|
| 99 | 1 | 11.33 | 16.80 |
| 99 | 4 | 22.61 | 19.96 |
| 0 | 1 | 55.81 | 27.11 |
| 0 | 4 | 32.16 | 43.74 |
