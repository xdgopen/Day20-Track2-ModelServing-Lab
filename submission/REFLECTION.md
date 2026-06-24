# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Nguyễn Danh Thành
**MSSV:** 2A202600581
**Ngày submit:** 2026-06-24

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** macOS trên kiến trúc arm64
- **CPU:** Apple M4 Pro
- **Cores:** 12 physical / 12 logical
- **CPU extensions:** NEON (Apple Silicon)
- **RAM:** 48 GB
- **Accelerator:** Apple Metal
- **llama.cpp backend đã chọn:** Metal
- **Recommended model tier:** Qwen2.5-7B-Instruct (Q4_K_M); bài lab chủ động dùng TinyLlama-1.1B làm model mặc định để thời gian setup và benchmark ngắn hơn.

**Setup story** (≤ 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn (vd: dùng WSL2, install CUDA Toolkit, fall back sang Vulkan vì ROCm phiên bản kén, tắt antivirus để pip install nhanh hơn, v.v.):

Môi trường Python đã có llama-cpp-python build với Metal. Để phù hợp thời gian thực hành, tôi đổi downloader mặc định sang TinyLlama-1.1B và tải cả Q4_K_M lẫn Q2_K. Khi khởi chạy API server, bổ sung các dependency server của llama-cpp-python vào requirements.txt.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `make bench`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| TinyLlama-1.1B Q4_K_M | 6515 | 18 / 18 | 4.8 / 8.4 | 318 / 325 / 325 | 206.3 |
| TinyLlama-1.1B Q2_K | 54 | 19 / 61 | 7.0 / 7.8 | 461 / 520 / 522 | 142.7 |

**Một quan sát** (≤ 50 chữ): Q4_K_M vs Q2_K trên máy bạn — số liệu nói gì? Quality đáng đánh đổi không?

Q4_K_M vừa cho TPOT P50 thấp hơn (4.8 ms so với 7.0 ms), vừa đạt 206.3 tok/s so với 142.7 tok/s của Q2_K. Trên M4 Pro, Q4_K_M là lựa chọn hợp lý hơn: chất lượng quantization tốt hơn nhưng không đánh đổi tốc độ trong lần đo này.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | E2E P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 2.90 | 2300 | 3700 | 3900 | 0 / 171 (0.00%) |
| 50 | 3.03 | 7200 | 13000 | 15000 | 0 / 180 (0.00%) |

Locust dùng request không streaming, vì vậy số ở bảng là E2E latency (không phải TTFB tách riêng). Khi tăng từ 10 lên 50 users, RPS gần như giữ nguyên nhưng E2E P95 tăng từ 3.7 s lên 13.0 s; đây là dấu hiệu hàng đợi/saturation, không phải năng lực throughput tăng.

**KV-cache observation:** `record-metrics.py` đã ghi 30 samples trong `benchmarks/02-server-metrics.csv`. Sampler báo peak `llamacpp:kv_cache_usage_ratio = 0.00`, trong khi `requests_processing` đạt 4 và `requests_deferred` đạt 32 ở đầu phiên. Giá trị 0.00 cho thấy metric KV ratio không được native server/backend này export theo tên mà script đang parse; không nên suy ra KV cache không được sử dụng. Quan sát đáng tin cậy từ trace là 4 slot đang bận và request bị xếp hàng, phù hợp với P95 tăng mạnh ở 50 users.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: llama-server localhost trên cổng 8080
- **N17 (Data pipeline):** stub: dữ liệu in-memory trong `TOY_DOCS`
- **N18 (Lakehouse):** stub: chưa kết nối lakehouse; dữ liệu demo lưu trong mã nguồn
- **N19 (Vector + Feature Store):** stub: `TOY_DOCS` với keyword-overlap retrieval, trả provenance ID cho từng context

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: Không áp dụng (stub keyword retrieval)
- retrieve: 0.0 ms (cả 3 query, toy keyword retrieval)
- llama-server: 528.7 ms trung bình (266.9–782.7 ms)

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

Pipeline chạy end-to-end cho 3 query và in provenance context: `n20-paged`, `n20-radix`, `n20-disagg`. Bottleneck là llama-server (266.9–782.7 ms), còn retrieval in-memory đo được 0.0 ms. Kết quả khớp kỳ vọng: trong demo này retrieval chỉ là keyword overlap trên vài document, còn phần sinh token mới là chi phí chính.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** Chọn TinyLlama Q4_K_M thay cho Q2_K sau khi đo benchmark trên Apple M4 Pro.

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: Q2_K decode rate = 142.7 tok/s; TPOT P50 = 7.0 ms
after:  Q4_K_M decode rate = 206.3 tok/s; TPOT P50 = 4.8 ms
speedup: ~1.45× theo decode rate
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Kết quả này ngược với trực giác đơn giản rằng quantization thấp hơn luôn nhanh hơn. Với model nhỏ trên M4 Pro, overhead điều phối kernel, memory layout và đường đọc/dequantize có thể chi phối nhiều hơn số bit mỗi weight. Q4_K_M có layout/kernel phù hợp hơn cho lần build Metal này nên giảm TPOT từ 7.0 ms xuống 4.8 ms.

Vì vậy, chỉ chọn Q2_K khi bị giới hạn RAM là chưa đủ. Cần benchmark trực tiếp quantization trên đúng backend và đúng model đang triển khai; Q4_K_M đồng thời cho chất lượng văn bản tốt hơn và đạt tốc độ cao hơn trong số liệu của tôi.

---

## 6. (Optional) Điều ngạc nhiên nhất

_(1–2 câu — không bắt buộc, nhưng người grader đọc tất cả)_

Quantization nhỏ hơn không bảo đảm tốc độ decode cao hơn. Kết quả thực nghiệm trên M4 Pro khiến việc đo TPOT quan trọng hơn nhiều so với suy luận dựa riêng vào dung lượng GGUF.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã tạo
- [x] `models/active.json` đã tạo và path model resolve
- [x] `benchmarks/01-quickstart-results.md` đã tạo
- [x] `benchmarks/02-server-metrics.csv` đã tạo (30 samples)
- [x] `benchmarks/bonus-thread-sweep.md` đã tạo nhưng toàn bộ giá trị là `0.0 tok/s`; cần chạy lại để được coi là bonus evidence hợp lệ
- [x] Đã có 7 screenshots, gồm evidence Track 01–03 và bonus sweep
- [x] `make verify` cần chạy lại trước khi push
- [x] Repo GitHub public: cần xác nhận trước khi nộp
- [x] URL LMS: cần dán sau khi push

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
