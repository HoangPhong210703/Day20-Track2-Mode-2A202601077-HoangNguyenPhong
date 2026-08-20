# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.

**Họ Tên:** Hoàng Nguyễn Phong
**Cohort:** A20-K3
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

- **OS:** Windows 11 Home Single Language (10.0.26200)
- **CPU:** AMD Ryzen 7 7735HS with Radeon Graphics
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2 (Zen 3+)
- **RAM:** 27.3 GB
- **Accelerator:** AMD Radeon 680M (iGPU), backend Vulkan, chạy với `ngl=99`
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare), theo `models/active.json`

**Chạy ở đâu:** laptop của tôi (local, không dùng Colab/Kaggle).

**Setup story:** Tôi chủ động chọn Qwen3.5 0.8B dù `hardware.json` khuyến nghị Gemma 4
E2B — máy đủ 27.3 GB RAM, tôi chọn model nhỏ để vòng lặp đo nhanh hơn. Bốn thứ phải sửa:
(1) `fetch-runtime.py` không đối chiếu `Content-Length`, nhận file zip đứt 28.2/34.8 MB
rồi crash `BadZipFile` — tải lại bằng `curl`; (2) `lab.ps1` là UTF-8 không BOM, PowerShell
5.1 đọc theo ANSI nên em-dash thành dấu nháy và file không parse — thêm BOM; (3) báo cáo
`.md` bị ghi bằng cp1252 (`labkit.py:513` thiếu `encoding="utf-8"`); (4) cổng 8080 đã bị
`AgentService` chiếm — chuyển sang `LAB_SERVER_PORT=8090`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3551 | 2054 / 2163 | 12.8 / 13.5 | 2866 / 2942 / 2942 | 78.3 |
| UD-Q2_K_XL | 0.39 | 3543 | 2168 / 2274 | 14.4 / 15.5 | 3069 / 3250 / 3250 | 69.6 |

**Quan sát:** 2-bit **chậm hơn** 4-bit 1.12×, dù nhỏ hơn 22%. Băng thông hiệu dụng cho
thấy lý do: Q4 đạt ~39 GB/s, Q2 chỉ ~27 GB/s — nếu nghẽn ở memory thì cả hai đã chạm
cùng trần (~77 GB/s, DDR5-4800). Q2 bỏ phí băng thông vì kẹt ở compute (dequant Q2_K
tốn nhiều thao tác hơn). Tôi chưa chạy A/B chất lượng: 2-bit đã thua cả tốc độ lẫn độ
chính xác nên không cần đo thêm để loại.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.31 | 28000 | 41000 | 41000 | 7.8 | 0.0% |
| 50 | 0.39 | 32000 | 56000 | 57000 | 12.0 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.24×
- **P95 tăng:** 1.37×
- **Effective concurrency ở 50 users:** 12.0 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`:** 3.76 / 4 slots (94%)

**Saturation reading:** Server bão hoà ở **dưới 10 users**. Bằng chứng thuyết phục nhất
là `requests_deferred = 45` trong khi `requests_processing = 4`: 49 request nằm trong hệ
thống so với 50 offered — gần như tất cả đang **đợi**, không phải đang chạy. Slot đã bận
94% nên không còn compute rảnh; phần latency tăng thêm là queue time, không phải compute.
Để nâng goodput@SLO tôi đổi `--ubatch-size` trước, không phải `--parallel`: prefill warm
chỉ ~74 tok/s so với decode ~78 tok/s, trong khi prefill xử lý cả prompt song song và lẽ
ra phải nhanh hơn nhiều mỗi token. Thêm slot chỉ chia nhỏ một ngân sách vốn đã bị
concurrency làm giảm một nửa (41 tok/s tổng khi chạy 4 luồng, so với ~76 tok/s một luồng).

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | không có hạ tầng cloud trong pipeline | stub |
| N17 Data pipeline | corpus nhúng sẵn trong `pipeline.py` | stub |
| N18 Lakehouse | không có bảng/lakehouse nào được đọc | stub |
| N19 Vector + features | keyword overlap, không nạp embedding model | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 6388.6 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection:** Tỉ lệ 100% cho llm là **hệ quả của stub**, không phải phát hiện thật:
embed bằng 0 vì không có model embedding, retrieve 0.1 ms vì corpus chỉ là vài dict trong
bộ nhớ. Đúng như kỳ vọng. Nếu phải giảm 2× tôi tấn công phần "chết" bên trong stage llm
trước: client đo 5.7–7.5 s nhưng server chỉ báo 2.3–3.1 s prefill+decode — hơn một nửa
stage đó không phải compute.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

**Change:** giữ `Q4_K_M` thay vì hạ xuống `UD-Q2_K_XL` (quantization, đo bằng `make bench`)

```
before:  69.6 tok/s  (UD-Q2_K_XL)
after:   78.3 tok/s  (Q4_K_M)
speedup: 1.12x
```

**Tại sao nó work:**

Kết quả này **ngược** với kỳ vọng từ deck — ít bit hơn lẽ ra phải nhanh hơn. Ít bit chỉ
mua được tốc độ khi decode bị giới hạn bởi **memory bandwidth**. Nhân số byte trọng số
với tốc độ decode ra băng thông hiệu dụng: Q4 đạt 0.50 GB × 78.3 = ~39 GB/s, Q2 chỉ
0.39 GB × 69.6 = ~27 GB/s. Nếu memory là trần thì cả hai đã hội tụ về cùng một mức
(~77 GB/s lý thuyết của DDR5-4800 dual-channel). Q2 chuyển **ít dữ liệu hơn 30% mỗi
giây** — nó đang bỏ phí băng thông vì kẹt ở compute: superblock Q2_K mang nhiều metadata
scale/min hơn và cần nhiều thao tác giải nén trên mỗi trọng số hơn Q4_K. Ở quy mô 0.8B,
số byte tiết kiệm được quá nhỏ để bù cho phần ALU đó. Thời gian load gần như bằng nhau
(3551 vs 3543 ms) dù ít hơn 22% byte, càng xác nhận I/O không phải nút thắt.

Ba phép đo độc lập khác cùng chỉ về một kết luận — máy này **compute-bound, không
bandwidth-bound**: (1) sweep thread phẳng từ 1 đến 32 luồng (82.8 → 85.3 tok/s, 1.03×),
`-t 1` đã đạt 97% — với `ngl=99` thì decode không nằm trên CPU thread; (2) prefill warm
~74 tok/s xấp xỉ decode ~78 tok/s, trong khi prefill lẽ ra phải nhanh hơn nhiều mỗi
token; (3) batching 4 luồng chỉ cho 24 tok/s tổng so với 76.8 tok/s một luồng — batching
không amortize được gì. Trên phần cứng này, chọn đúng quantization đáng giá hơn mọi knob
về song song.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

**Đã làm:** B2 `sweep-batch` (chunked prefill) · B3 số liệu bên dưới · B5/C8 semantic
cache (`semantic-cache --offline --sweep`) — xem `benchmarks/bonus-batch-size-sweep.md`
và `benchmarks/bonus-semantic-cache.md`.

**Numbers (B2, metric `pp512`):**

```
before:  939.7 tok/s   (-b 128  -ub 128)
after:  1063.1 tok/s   (-b 512  -ub 256)
speedup: 1.13x
```

**Điều này nói lên gì mà deck chưa nói:**

Deck trình bày chunked prefill như một đánh đổi cần tune. Trên máy này nó không phải
đường cong mà là **ngưỡng**: mọi cấu hình từ 256 trở lên nằm trong 1.5% của nhau, chỉ
`-b 128 -ub 128` (88%) là thực sự tệ. Kết luận đúng là "đừng xuống dưới 256", không phải
"chỉnh tới 512/256".

Nhưng con số đáng nói nhất là chỗ khác: `llama-bench` prefill **1063 tok/s**, trong khi
`llama-server` của tôi chỉ đạt ~74–87 tok/s prefill. Phần cứng nhanh hơn **12×** so với
những gì server đang khai thác — đúng chỗ mà §3 đề xuất tấn công `--ubatch-size`, và giờ
có bằng chứng độc lập.

Với B5/C8: một hit của semantic cache bỏ qua **100% compute** (không prefill, không
decode), khác hẳn KV/prefix cache bên dưới vốn chỉ tái dùng phần prefix chung — 3/8 prompt
không hề chạm tới model. Sweep threshold **phẳng**, và đó là **artifact của stub embedder
bag-of-words** (cosine chỉ ra 1.0 hoặc 0.0), không phải phát hiện; muốn có đường cong thật
phải chạy `make serve-embed` rồi bỏ `--offline`.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Continuous batching làm **giảm** tổng throughput: 4 slot bận 94% chỉ sinh 24.1 tok/s,
trong khi một request đơn lẻ đạt 76.8 tok/s.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** đã được thay
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/`
