# Reflection — Lab 20 (Personal Report)

**Họ Tên:** _Minh Khang Le_
**Cohort:** _A20-K2_
**Ngày submit:** _2026-05-06_

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

- **OS:** _Linux 6.6.87.2-microsoft-standard-WSL2_
- **CPU:** _Intel(R) Core(TM) Ultra 7 155H_
- **Cores:** _22 physical / 22 logical_
- **CPU extensions:** _AVX2_
- **RAM:** _15.4 GB_
- **Accelerator:** _CPU only_
- **llama.cpp backend đã chọn:** _CPU (AVX/NEON tuning)_
- **Recommended model tier:** _Qwen2.5-1.5B-Instruct (Q4_K_M)_

**Setup story:** Máy thiếu `venv/pip` mặc định nên tôi bootstrap `.venv` bằng `uv`, sau đó cài dependencies và build `llama-cpp-python`. Để có `/metrics`, tôi build native `llama.cpp` và chạy `llama-server --metrics` thay vì Python server mặc định.

## 2. Track 01 — Quickstart numbers

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 5295 | 2552 / 5124 | 839.4 / 1566.7 | 54226 / 103293 / 109682 | 1.2 |
| qwen2.5-1.5b-instruct-q2_k.gguf   | 759 | 1361 / 2523 | 429.5 / 618.8 | 28092 / 40455 / 43114 | 2.3 |

**Một quan sát:** Q2_K nhanh hơn đáng kể trên CPU-only (decode ~2x), nhưng quality giảm rõ nên Q4_K_M vẫn là điểm cân bằng tốt hơn cho use case chat thông thường.

## 3. Track 02 — llama-server load test

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.02 | 42000 | 42000 | 42000 | 0 |
| 50 | 0.03 | 17000 | 55000 | 55000 | 0 |

**KV-cache observation:** peak `llamacpp:kv_cache_usage_ratio` ở concurrency 50 = _0.00_ trong run 60s này. Điều này cho thấy bottleneck chính là decode throughput CPU nên queue deferred tăng trước khi KV cache bị stress.

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** _stub: localhost-only serving (no cluster)_
- **N17 (Data pipeline):** _stub: in-memory retrieval pipeline_
- **N18 (Lakehouse):** _stub: no external lakehouse, local toy corpus_
- **N19 (Vector + Feature Store):** _stub: TOY_DOCS retrieval in pipeline code_

**Nơi tốn nhiều ms nhất** trong pipeline:

- embed: _0 ms (not implemented in toy pipeline)_
- retrieve: _0.0–0.1 ms_
- llama-server: _~5938 ms to ~33593 ms (dominant)_

**Reflection:** Bottleneck nằm ở LLM decode trên CPU-only runtime. Kết quả này khớp kỳ vọng: retrieval rất nhẹ, phần generation chiếm gần như toàn bộ end-to-end latency.

## 5. Bonus — The single change that mattered most

**Change:** _Build native `llama.cpp` + chạy `llama-server --metrics` để có observability và tuning surface rõ hơn (thay cho Python wrapper server)._

**Before vs after**

```text
before: Python server variant trả /metrics = 404 (không đo được KV/decode counters)
after:  Native llama-server trả đầy đủ Prometheus metrics (tokens_predicted_total, n_decode_total, kv_cache_usage_ratio)
speedup: không phải throughput speedup; đây là observability speedup (0 -> full visibility)
```

**Tại sao nó work:** Python wrapper thuận tiện để bắt đầu nhưng build variant có thể thiếu các feature flag runtime. Native binary từ source cho control trực tiếp vào serving flags (`--metrics`, `--parallel`, `--cont-batching`) và xuất đúng counters cần cho production tuning.

Về mặt kỹ thuật, khi không có metrics thì không thể phân biệt queueing do compute, do memory, hay do KV saturation. Sau khi có metrics, tôi quan sát được deferred requests tăng trong khi KV usage gần như không tăng, suy ra bottleneck nằm ở decode throughput CPU chứ không phải cache pressure.

## 6. (Optional) Điều ngạc nhiên nhất

Điều bất ngờ nhất là trong cửa sổ 60 giây, số request hoàn thành rất thấp ở cả 10 và 50 users trên CPU-only, nhưng metrics vẫn rất hữu ích để chỉ ra đúng loại bottleneck.

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [x] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**
- [x] Đã paste public repo URL vào VinUni LMS
