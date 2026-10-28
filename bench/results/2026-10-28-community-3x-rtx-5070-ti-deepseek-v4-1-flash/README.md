# Community benchmark on NVIDIA GeForce RTX 5070 Ti + NVIDIA GeForce RTX 5070 Ti + NVIDIA GeForce RTX 5070 Ti

Measured on 2026-10-28 with `tools/strata_bench.py` (suite 1, 3 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: NVIDIA GeForce RTX 5070 Ti + NVIDIA GeForce RTX 5070 Ti + NVIDIA GeForce RTX 5070 Ti, 48 GiB VRAM
- CPU: AMD Ryzen 7 9700X 8-Core Processor (8 cores, 16 threads); installed RAM: 93.1 GiB; WD_BLACK SN7100 1 TB NVMe (PCIe 4.0) for the model files (the experts that fit neither VRAM nor RAM and the Engram table are read from it)
- OS: Windows 11 (10.0.26200); GPU driver: 616.92; CUDA (driver supports up to): 13.4; engine built for CUDA 13.0
- Strata: desktop app 0.2.7 (Windows installer); engine strata-dsv41 0.1.52 (cuda13)
- PCIe link: two cards PCIe Gen5 x8 (x16 slot split x8/x8), the third PCIe Gen4 x4 through an OCuLink adapter in the CPU-attached M.2 slot (Gen4 redriver, link fixed to Gen4); all GPUs on CPU lanes, none behind the chipset
- Memory and CPU: RAM 96 GB as 4 DIMMs (2x 32 GB + 2x 16 GB DDR5) at JEDEC DDR5-4800, no EXPO; CPU Curve Optimizer -20
- GPU detail (after the run, near idle): power_limit_w=300.00, power_draw_w=55.42, pcie=gen5 x8, pstate=P1, temp_c=44
- GPU detail (after the run, near idle): power_limit_w=300.00, power_draw_w=67.03, pcie=gen5 x8, pstate=P1, temp_c=38
- GPU detail (after the run, near idle): power_limit_w=300.00, power_draw_w=47.17, pcie=gen4 x4, pstate=P1, temp_c=38
- Background workloads and power limits: Windows power plan "Balanced"; all three GPUs undervolted with an MSI Afterburner V/F curve applied at Windows start, 2797 MHz at 885 mV, power limit 300 W (default), memory +1000 MHz effective (15001 MHz instead of 14001), as in the earlier 5070 Ti reports; no other load

## Model and configuration

- Model and quantization: DeepSeek-V4.1-Flash, IST-DAS 2.00 bpw (ISTA-DASLab GSQ-RCO IQ2_XXS, 138 GB) with the Engram table IQ4_NL (111 GB); DSpark drafter present but left unloaded by `--dspark auto`
- Placement (engine log): 3,622 of 15,360 routed experts in VRAM (31.5 GB over the three cards, layer split 14/13/13 layers; `--tp 2` falls back to one card per layer), a RAM store of 60 GB (5,840 slots, 45.4 GB of the profile's hottest experts read in at start), the rest read in place from the NVMe
- Context: 65536 tokens; parallel slots: 1
- Settings: the install's own configuration (desktop app setup), with one change: `--ssd-cache 60` (the RAM store; the automatic size on this machine was 51.1 GB); thinking off for the suite; temperature 0

```text
strata-dsv41.exe --pack deepseek-v4.1-flash-ist-das-2.00bpw --max-context 65536 --eos 1 --vision off --dspark auto --dspark-conf 0.65 --tp 2 --ssd-cache 60
```

## Method

Run by `python tools/strata_bench.py` against the HTTP API of a Strata server (started by the benchmark from the install configuration); model loading is not timed. One warm-up chat and a 1K prompt are discarded first. Every prompt starts with a unique tag, so the conversation cache cannot help the fresh-prompt tests.

- short chat: the three questions of `tools/ab_engine.py`, up to 100 new tokens, 3 runs
- 4K / 16K prompts: generated word lists sized by a calibration request, 24 new tokens, 3 fresh prompts each
- agent session: system prompt and four tools, then 5 turns that each add about 1500 words of a project file and continue from the model's own earlier answers (the way agent clients resend a conversation), 64 new tokens per turn, 3 sessions; the cache numbers come from the answer's usage (`cached_tokens`) and timings (`prompt_n`)
- concurrency (when run): N requests sent at the same moment, total tokens over the wall time of the burst, latency over all requests

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| short chat | 32 (29-38) | 0 | 100 (100-100) | 9 | not measured | 16.9 (13.9-23.1) | 1.00 (0.61-1.21) |
| 4K prompt, fresh | 4125 (4125-4125) | 0 (0-0) | 24 cap | 3 | 543.6 (467.9-578.5) | not measured | 7.61 (7.14-8.83) |
| 16K prompt, fresh | 16562 (16562-16562) | 0 (0-0) | 24 cap | 3 | 1073.3 (975.2-1115.3) | not measured | 15.45 (14.88-17.00) |
| agent session, follow-up turns | 8879 (8853-8883) at the last turn | see below | up to 64 | 3 | see below | 20.7 (17.0-22.9) | 6.10 (5.10-7.16) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 71.0 % (71.0-71.1); earlier-turn tokens read again per session: 5 (5-5) (earlier turns that the cache did not hold; the 8055 (8055-8055) new tokens of the files are read once); first turn (nothing cached) first token: 8.97 s (8.93-11.17).

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 602 | 0 | 602 | 54.0 | 11.172 | 14.9 |
| 1 | 2246 | 634 | 1612 | 293.9 | 5.497 | 18.9 |
| 2 | 3903 | 2291 | 1612 | 225.9 | 7.155 | 17.0 |
| 3 | 5555 | 3943 | 1612 | 228.7 | 7.059 | 18.6 |
| 4 | 7204 | 5592 | 1612 | 233.0 | 6.938 | 19.1 |
| 5 | 8853 | 7241 | 1612 | 238.5 | 6.769 | 21.3 |

Per-request data: `results.json`. Engine log lines from the start: `engine-log-excerpt.txt` (when this run started the server).

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load is not recorded. Total time of the suite: 331 s.
