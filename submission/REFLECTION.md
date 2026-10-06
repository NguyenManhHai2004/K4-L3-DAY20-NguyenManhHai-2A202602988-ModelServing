# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyen Manh Hai
**MSSV:** 2A202602988
**Cohort:** A20-K1
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (AMD64)
- **CPU:** 11th Gen Intel Core i5-1135G7 @ 2.40GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2 / AVX-512 (theo thông số Intel của i5-1135G7; script probe không in mục này)
- **RAM:** 15.7 GB
- **Accelerator:** Vulkan — Intel Iris Xe Graphics tích hợp (probe báo `GPU offload: ACTIVE -- Vulkan0`)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (không dùng Colab/Kaggle).

**Setup story** (≤ 80 chữ): `lab.ps1` báo lỗi cú pháp trên Windows PowerShell 5.1 vì file có ký tự không phải ASCII mà thiếu BOM; tôi thêm BOM UTF-8. Lần `setup` đầu bị ngắt để lại `.venv` dở dang, tôi xóa và chạy lại bằng `bootstrap.ps1`. Máy đủ RAM cho Gemma nhưng tôi chọn Qwen3.5 0.8B cho nhẹ. Tải từ GitHub rất chậm (khoảng 13 KB/s) nên bước runtime mất vài phút.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 6860 | 599 / 638 | 37.4 / 39.0 | 2943 / 3074 / 3074 | 26.8 |
| UD-Q2_K_XL | 0.39 | 6187 | 704 / 744 | 62.3 / 63.4 | 4630 / 4720 / 4720 | 16.1 |

**Quan sát** (≤ 60 chữ): Bản 2-bit **chậm hơn 1.66×** (TPOT 62.3 so với 37.4 ms) mà chỉ nhỏ hơn 0.11 GB, nên không đáng. Chất lượng cũng kém: cùng prompt, `temperature=0`, Q4 trả lời đúng 17×23=391, Q2 trả lời sai (381) và bỏ qua giới hạn 4 câu, lặp câu. Chỉ chạy một lần mỗi câu nên mang tính minh họa.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.64 | 13000 | 19000 | 19000 | 8.1 | 0.0% |
| 50 | 0.74 | 30000 | 57000 | 58000 | 22.6 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.16×
- **P95 tăng:** 3.00×
- **Effective concurrency ở 50 users:** 22.6 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.98 / 4 slots

**Saturation reading** (≤ 80 chữ): Server đã bão hòa ngay ở 10 users: effective concurrency 8.1, gấp đôi 4 slot. Từ 10 lên 50 users RPS chỉ tăng 1.16× nhưng P95 tăng 3×, nên phần tăng thêm là queue time (`requests_processing`=4, `requests_deferred`≈46). Bốn slot chỉ cho ~39.9 tok/s tổng so với ~27 tok/s một luồng, nên giới hạn là tính toán, không phải số slot. Tôi sẽ đổi đường tính toán trước (`ngl=0`), không thêm slot.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | một `llama-server` cục bộ, không có cluster | stub |
| N17 Data pipeline | không dùng; tài liệu là danh sách cố định | stub |
| N18 Lakehouse | không dùng; corpus là `TOY_DOCS` (8 đoạn ngắn) | stub |
| N19 Vector + features | keyword overlap, không có embedding server hay vector index | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 4982.1 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Khớp kỳ vọng: embed và retrieve là stub nên gần như 0, llm chiếm hết, và trong llm thì decode ~90% (~38 ms/token). Muốn giảm 2× tôi sẽ rút ngắn đầu ra (câu 3 chạm trần 200 token) và thử chạy CPU (1.31×). Lưu ý: gọi qua `localhost` tốn thêm ~2.3 s mỗi lần trên máy này; dùng `127.0.0.1`.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** chuyển decode từ GPU tích hợp (`ngl=99`, Iris Xe qua Vulkan, mặc định) sang CPU (`ngl=0`) với `-t 4` (bằng số core vật lý)

```
before:  28.0 tok/s   (ngl=99, điểm tốt nhất của đường GPU) — benchmarks/01-tuning-tg128-ngl99.md
after:   36.8 tok/s   (ngl=0,  -t 4)                       — benchmarks/01-tuning-tg128.md
speedup: 1.31×
```

(So cùng `-t 4`: 27.9 → 36.8 tok/s, 1.32×. Trong đường CPU, đổi `-t 1` → `-t 4` là 17.2 → 36.8 tok/s, tức 2.14×.)

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Kết quả trái với kỳ vọng ban đầu của tôi: tôi tưởng GPU offload sẽ nhanh hơn. Đường GPU gần như phẳng theo số thread (27.1–28.0 tok/s với 1/2/4/8/16 thread, chênh 1.03×), vì decode chạy trên Iris Xe, CPU chỉ phát lệnh. Ở đường CPU thì có knee rõ rệt đúng tại 4 thread (số core vật lý): 17.2, 27.6, 36.8, 36.9, 26.6 tok/s cho 1/2/4/8/16 thread. Từ 4 lên 8 không tăng vì 4 thread thêm là hyperthread dùng chung đơn vị thực thi và cache; decode đọc lại toàn bộ trọng số mỗi token nên bị giới hạn bởi bộ nhớ, đã bão hòa ở 4 core. Ở 16 thread tốc độ giảm còn 72% vì vượt số core logic: thread bị lập lịch xen kẽ và phải chờ nhau ở các điểm đồng bộ giữa các layer (đây là suy đoán, tôi chưa profile).

Vì sao CPU thắng GPU ở đây: Iris Xe là GPU tích hợp, dùng chung RAM hệ thống với CPU nên không có lợi thế băng thông, và model 0.5 GB nhỏ nên chi phí phát nhiều kernel Vulkan nhỏ cho mỗi layer chiếm tỉ trọng lớn so với phần tính toán. Đây là giả thuyết, tôi chưa chụp profile Vulkan để xác nhận. Con số GPU khớp với `make bench` qua server (26.8 tok/s). Một lưu ý trung thực: file `01-tuning-tg128.md` bị ghi đè khi tôi chạy lần hai với `ngl=0`, nên số GPU nằm ở bản sao `01-tuning-tg128-ngl99.md` (nguyên văn output của lần chạy đầu); mỗi điểm chỉ đo 2 lần trên laptop nên có thể lệch vài phần trăm. Tôi cũng chưa đo xem lợi thế của CPU có giữ được khi chạy 4 slot song song.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

Không làm bonus.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

GPU tích hợp lại chậm hơn CPU cho model nhỏ (28 so với 37 tok/s), và gọi `localhost` trên Windows tốn thêm ~2.3 s mỗi request so với `127.0.0.1`.

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
- [x] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Tôi dùng **Claude Code (Anthropic)** trong suốt bài lab này, vào các việc sau:
- Chạy các lệnh của lab (`probe`, `setup`, `bench`, `tune`, `serve`, `smoke`, `load-10`, `load-50`, `metrics`, `load-report`, `pipeline`) và gỡ lỗi (lỗi BOM của `lab.ps1`, `.venv` dở dang).
- Chụp các screenshot `03`–`07`: Claude Code mở các cửa sổ terminal thật chạy các lệnh trên máy tôi rồi chụp lại bằng script. Ảnh là chụp thật, không dựng giả. Hai ảnh locust được chạy bằng đúng lệnh locust của `make load-10` / `load-50` có thêm cờ `--only-summary` để bảng percentile hiện đủ.
- Soạn bản nháp các phần nhận xét ("required") trong `benchmarks/*.md` và các phần lập luận trong REFLECTION này, dựa trên số liệu thật sinh ra từ máy tôi. Số liệu không bị sửa tay; số GPU của §5 lấy từ bản sao nguyên văn `01-tuning-tg128-ngl99.md`.

Tôi chịu trách nhiệm về nội dung và sẽ giải thích lại được các lập luận trên nếu được hỏi.
