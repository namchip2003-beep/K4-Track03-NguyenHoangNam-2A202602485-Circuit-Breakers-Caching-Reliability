# Lab 25B — Report

**Họ tên / nhóm:** Nguyễn Hoàng Nam - 2A202602485  **Commit:** `HEAD`  **Điểm tự động (`make grade`):** 84.0 / 88 (+ 4đ SLO hidden)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS |

---

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| `attempt_timeout_s` | 8.0 s | Tránh treo quá lâu khi provider bị sập/hang (ở G6, nếu đặt 30 s thì 122 requests bị timeout và mất tính sẵn sàng). 8.0 s đủ lớn hơn median bình thường (1.2–1.8 s) và p99 (~5.8 s), nhưng chừa đủ 22 s trong deadline 30 s để kịp failover. |
| `max_attempts_per_provider` | 2 | Cho phép thử lại 1 lần trên cùng provider khi gặp transient error. Nếu lần 2 vẫn lỗi, lập tức failover thay vì tiếp tục dồn ép provider đang nghẽn (ở G2 naive thử 36 lần gây bão 34.475 attempts). |
| `backoff_base_s` / `cap_s` | 0.25 s / 4.0 s | Full jitter phân tán ngẫu nhiên các attempt thử lại, tránh cộng hưởng thundering herd. Cap 4.0 s đảm bảo retry không bị trễ quá lâu trước hạn mức deadline. |
| `retry_budget_ratio` | 0.1 | Hạn chế lượng retry không vượt quá 10% tổng số request gửi mới. Ở G3, kiểm soát `retry_ratio = 0.034` (dưới ngưỡng tối đa 0.10), dập tắt hoàn toàn retry storm. |
| `breaker.window_size` / `min_calls` | 20 / 10 | Cửa sổ trượt 20 kết quả và yêu cầu tối thiểu 10 cuộc gọi để có ý nghĩa thống kê; ngăn breaker mở nhầm do vài lỗi đơn lẻ lúc traffic thấp (G5 tránh mở breaker nhầm khi gặp bad deploy 400). |
| `breaker.failure_rate_threshold` | 0.5 | Khi tỷ lệ lỗi đạt 50% trong cửa sổ 20 call, mạch ngắt ngay lập tức chuyển sang OPEN để bảo vệ provider và chuyển hướng tải sang backup. |
| `breaker.slow_call_s` | 5.0 s | Điểm mấu chốt để vượt qua G4: alpha chậm thêm 5 s (median ~6.2 s) nhưng không trả về lỗi. Nhận diện phản hồi >= 5.0 s là cuộc gọi chậm để ngắt mạch, giảm p95 từ 13.764 s (naive) xuống 5.562 s. |
| `breaker.open_duration_s` | 10.0 s | Thời gian cách ly 10 s vừa đủ để hàng đợi của provider (max queue 10 s) xả hết các request tồn đọng trước khi chuyển sang HALF-OPEN thăm dò. |
| `breaker.half_open_max_probes` | 2 | Chỉ cho phép tối đa 2 request đồng thời thăm dò trong trạng thái HALF-OPEN, ngăn chặn thundering herd làm sập lại provider đang trong giai đoạn hồi phục. |

### Sơ đồ quyết định của `on_error`:

```
                    ┌───────────────────────────┐
                    │       Provider Error      │
                    └─────────────┬─────────────┘
                                  │
                               classify()
                                  │
      ┌───────────────────────────┼───────────────────────────┬───────────────────────────┐
      ▼                           ▼                           ▼                           ▼
[CLIENT_ERROR]           [QUOTA_EXHAUSTED]              [RATE_LIMITED]          [OVERLOAD / SERVER / TIMEOUT]
(400, 401, 422)             (429 quota)                (429 limit_exceeded)        (500, 502, 503, 529, 0)
      │                           │                           │                           │
Release permit              Release permit              Release permit              Record on_failure()
(Không tính breaker)       (Không tính breaker)        (Không tính breaker)        (Tính vào Breaker)
      │                           │                           │                           │
GiveUp("client_error")      Cooldown provider           Cooldown provider                         │
Không retry                 300s (quota_cooldown_s)     now + retry_after                         │
Không failover                    │                           │                                   │
                                  ▼                           ▼                                   ▼
                            Failover ngay               Failover ngay               Tries < max_attempts (2)
                            (route sang provider khác)  (route sang provider khác)  AND fits deadline
                                                                                    AND budget.try_acquire()?
                                                                                     ┌────┴────┐
                                                                                    YES        NO
                                                                                     │          │
                                                                                 Retry same  Failover
                                                                                 (Backoff)   (route next)
```

---

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối:**
   Khi cố tình vô hiệu hóa Circuit Breaker (`breaker.min_calls: 1.0e9`) và nâng `max_attempts_per_provider: 5`:
   - *(a) Khi GIỮ Retry Budget:* Ở G2, amplification giữ ở mức **1.380**, retry ratio là **0.058**, recovery **0.0 s**, availability đạt **99.87%**. Ở G3, retry ratio là **0.068** (đạt yêu cầu <= 0.10).
   - *(b) Khi TẮT Retry Budget:* Ở G2, amplification vọt lên **1.673**, retry ratio tăng gần 10 lần lên **0.494**, recovery mất **8.0 s**, availability giảm còn **97.04%**. Ở G3, retry ratio tăng lên **0.159** (vi phạm tiêu chí <= 0.10).
   - *Kết luận:* Kể cả khi breaker hoạt động bình thường, vẫn cần cả hai: Breaker ngắt mạch dựa trên sức khỏe của *từng provider cụ thể*, còn Retry Budget là chốt chặn tài nguyên toàn cục (global rate limiter) ngăn client khuếch đại lưu lượng lên hệ thống khi các điều kiện ngắt mạch chưa kịp thỏa mãn.

2. **`attempt_timeout_s: 30`:**
   Khi nâng attempt timeout từ 8 s lên 30 s:
   - Ở G6 (outage có `alpha-eu` bị treo/hang), availability tụt từ **1.0000** xuống **0.9488** (FAIL tiêu chí >= 0.95), và xuất hiện **122 lượt give up do deadline**. Tuy nhiên, p95 lại giảm từ **5.961 s** xuống **5.318 s**! Đây là minh chứng rõ nét cho hiện tượng *survivorship bias* (độ trễ p95 chỉ tính trên các lượt thành công, còn các lượt bị treo 30 s đã chết chìm và bị loại khỏi mẫu đo).
   - Ở G4 (slow region), kết quả gần như không đổi (p95 từ 5.562 s lên 5.608 s, availability giữ nguyên 1.0000) vì độ trễ của alpha chỉ tăng thêm 5 s (median ~6.2 s), các phản hồi vẫn về trước mốc 8 s nên không bị timeout cắt ngang mà được Circuit Breaker nhận diện qua cơ chế `slow_call_s`.
   - *Timeout đúng:* Nên được thiết lập dựa trên `p99 bình thường + buffer vừa đủ`, đồng thời phải nhỏ hơn đáng kể so với user turn deadline (30 s) để chừa đủ thời gian thực hiện ít nhất 1–2 lần failover sang provider dự phòng.

3. **Bỏ lọc tool calling khỏi `ladder()`:**
   Khi bỏ điều kiện kiểm tra năng lực tool calling trong hàm `ladder()`, chạy trên băng G6:
   - Availability giảm mạnh từ **1.0000** xuống còn **0.8977**, đồng thời phát sinh **244 incompatible routes**.
   - *Góc nhìn Product Manager:* 244 khách hàng yêu cầu hoàn tiền đã bị hệ thống đẩy sang mô hình giá rẻ (`gamma-mini`) không có năng lực thực thi công cụ hoàn tiền. Kết quả là agent phản hồi ngớ ngẩn hoặc báo lỗi kỹ thuật vô căn cứ, làm gián đoạn trải nghiệm của gần 10% khách hàng trong khung giờ cao điểm và trực tiếp làm giảm uy tín thương hiệu.

---

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.00 | 81 | 128 | 81 | 0 | 1,496,700 ₫ | 1.123 |
| yours | 1.00 | 66 | 0 | 0 | 0 | 0 ₫ | 0.303 |

**Giải thích:**
- *Vì sao chỉ Idempotency key là chưa đủ?* Idempotency key chỉ bảo vệ ở phía nhà cung cấp dịch vụ bên ngoài (Payment Gateway không trừ tiền 2 lần cho cùng 1 key). Nếu worker bị crash và khởi động lại mà không có Checkpoint, worker sẽ phải chạy lại toàn bộ quy trình từ đầu, dẫn đến lãng phí chi phí gọi LLM phân loại/tóm tắt (`wasted_llm_per_crash`).
- *Vì sao chỉ Checkpoint là chưa đủ?* Nếu xảy ra sự cố mạng làm mất gói tin phản hồi (`ServiceTimeout`) hoặc worker chết ngay trước khi kịp ghi checkpoint, lượt retry bắt buộc phải gửi lại yêu cầu thanh toán. Nếu không có idempotency key, cổng thanh toán sẽ coi đó là giao dịch mới và trừ tiền lần 2.
- *Điều gì xảy ra nếu ghi checkpoint TRƯỚC khi gọi payment?* Nếu ghi checkpoint trước khi gọi hàm thanh toán, trường hợp lệnh thanh toán thất bại hoặc worker crash trước khi thanh toán hoàn tất, worker mới khởi động lại sẽ thấy checkpoint đã tồn tại và bỏ qua bước hoàn tiền. Hậu quả là khách hàng bị **mất tiền hoàn** (`lost_refunds > 0`), vi phạm nghiêm trọng cam kết tài chính.

---

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 min |
| provider_outage | page | page | page | 4 min |
| deploy_blip | none | page (False Alert) | none (Đúng) | Không kích hoạt |
| slow_leak | ticket | none (Bỏ sót) | ticket (Đúng) | 1,337 min |
| degraded_evening | page | page | page | 291 min |
| night_spike | none | page (False Alert) | none (Đúng) | Không kích hoạt |
| small_bump | none | page (False Alert) | none (Đúng) | Không kích hoạt |

**Error budget 30 ngày:**
- Tổng số requests: **25,901,755**
- Số lỗi tối đa cho phép (SLO 99.5%): **129,508.8**
- Số lỗi thực tế: **55,618** (Availability: **99.7853%**)
- Ngân sách lỗi đã tiêu thụ: **42.95%**, ngân sách còn lại: **57.05%**.
- **Quyết định:** **KHÔNG đóng băng release** (`freeze_releases = False`) vì vẫn còn hơn 57% ngân sách lỗi.
- *Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?* Một đợt lỗi ngắn 20 phút chỉ tiêu tốn chưa đến 0.2% ngân sách lỗi của cả tháng. Quy tắc đa cửa sổ (yêu cầu cả cửa sổ dài 1h và cửa sổ ngắn 5m cùng vượt ngưỡng 14.4x) giúp phân biệt giữa sự cố mang tính hủy diệt đe dọa SLO với các biến động cục bộ tạm thời, ngăn chặn triệt để tình trạng mệt mỏi vì báo động giả (*alert fatigue*).

---

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 Overload Storm](file:///d:/Du%20an/lab25/K4-Track03-NguyenHoangNam-2A202602485-Circuit-Breakers-Caching-Reliability/reports/gameday/G2_overload_storm.png)

**Tóm tắt (3 câu):**
Vào giây ảo t=120, cụm provider chính `alpha` bị mất 75% năng lực phần cứng trong vòng 180 giây. Do phiên bản `NaiveClient` cấu hình retry 3 tầng không có Circuit Breaker và không có jitter/backoff, lưu lượng retry đã bùng nổ lên gấp gần 10 lần làm nghẽn hoàn toàn hàng đợi của provider. Sự cố biến thành trạng thái lỗi bền vững (metastable failure), khiến hệ thống không thể tự phục hồi và tiếp tục sập thêm 155 giây sau khi hạ tầng phần cứng đã hồi phục 100%.

**Ảnh hưởng:**
- Tổng cộng **1,285 lượt người dùng thất bại** (availability tụt xuống **45.6%** so với mục tiêu >= 95%).
- Tổng thời gian gián đoạn kéo dài **335 giây** (180 giây sự cố phần cứng + 155 giây dư chấn retry storm).
- Sự cố này tiêu tốn hơn **62% tổng ngân sách lỗi của cả tháng** chỉ trong vòng chưa đầy 6 phút.

**Timeline (giây ảo — trích xuất từ `reports/gameday/naive_G2_overload_storm_timeseries.csv`):**

| t (s) | Sự kiện | Bằng chứng số liệu |
|---:|---|---|
| 120 | Hạ tầng `alpha` mất 75% GPU | Năng lực phục vụ giảm đột ngột; `calls_ok` giảm từ 10 req/s xuống 2 req/s ở t=121. |
| 136 | Bắt đầu bùng nổ Retry Storm | Hàng đợi đầy, 529 và timeout bắt đầu trả về; attempts bắn vào alpha vọt từ 21 lên 53 req/s. |
| 143 | Hệ thống sập hoàn toàn | Attempts đạt đỉnh **254 attempts/s** (gấp 12.7 lần giới hạn rate limit 20 rps); `calls_ok` rớt về **0 req/s**. |
| 300 | Nhà cung cấp phục hồi 100% tài nguyên | Số server của alpha quay lại 18 slot, nhưng `calls_ok` vẫn là **0 req/s**, attempts duy trì ở mức cao 115–176 req/s. |
| 372 | Đỉnh bão retry thứ hai | Lượng request retry dồn ứ đạt đỉnh mới **176 attempts/s**; toàn bộ CPU/GPU của provider bị lãng phí để xử lý các request mà client đã từ bỏ từ trước. |
| 453 | Hệ thống bắt đầu phục hồi | Bão retry cạn kiệt, hàng đợi xả hết; `calls_ok` xuất hiện trở lại ở mức 4 req/s (**155 giây sau khi provider hồi phục**). |
| 465 | Ổn định trở lại | `calls_ok` đạt lại 12 req/s, attempts ổn định ở mức 13 attempts/s. |

**Trigger vs Root Cause:**
- **Trigger:** Provider `alpha` bị giảm 75% số slot xử lý trong 180 giây do sự cố phần cứng hạ tầng.
- **Root Cause:** Thiết kế client v1 thiếu khả năng chịu lỗi nghiêm trọng: cơ chế retry 3 tầng lồng nhau không có Backoff Jitter, không có Retry Budget và không có Circuit Breaker.
- **Cơ chế Metastable Failure:** Khi hàng đợi của server bị đầy, client bị timeout sau vài giây và bỏ cuộc. Tuy nhiên, hàng đợi FIFO của server vẫn tiếp tục xử lý các request mồ côi này, gây lãng phí 100% năng lực tính toán. Đồng thời, client tiếp tục bắn thêm hàng trăm retry mới vào hệ thống, duy trì mức tải vượt xa năng lực của server kể cả sau khi server đã hồi phục hoàn toàn vào lúc t=300. Vòng lặp tự duy trì này chỉ dừng lại khi tất cả client hết lượt retry và bị drop do quá deadline.

**Điều gì giúp / Điều gì làm tệ hơn:**
- *Làm tệ hơn:* Retry ngay lập tức với delay cố định; vi phạm chỉ dẫn `retry-after`; thiếu cơ chế failover sang vùng `alpha-eu` hay `beta`.
- *Giúp giải quyết (trong `ResilientClient`):* Circuit Breaker cắt đứt dòng request khi tỷ lệ lỗi > 50%; failover ngay sang `alpha-eu`; Retry Budget khống chế lượng retry tối đa 10%; Full Jitter làm tan biến hiện tượng thundering herd.

**Action Items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai `ResilientClient` thay thế `NaiveClient` trên toàn bộ production | Prevent | On-call / Platform Team | Chạy CI test đạt amplification <= 1.2 trên kịch bản G2 |
| 2 | Cấu hình Token-bucket Retry Budget ở tầng Gateway với tỷ lệ ratio = 0.1 | Prevent | Gateway Team | Unit test xác nhận số retry không vượt quá 10% traffic khi provider lỗi 100% |
| 3 | Tích hợp Burn-rate Alerting đa cửa sổ vào hệ thống PagerDuty | Detect | SRE Team | Kiểm tra giả lập incident: page trong vòng 5 phút, 0 false alert |
| 4 | Cấu hình Header kiểm tra Deadline Propagation ở server | Mitigate | Backend Core | Server tự động hủy (drop) request trong queue nếu client đã quá thời gian timeout |

---

## 7. Một điểm yếu còn lại

**Tình huống:** Khi cụm chính `alpha` gặp sự cố kéo dài, toàn bộ tải dồn sang `beta`. Mặc dù `beta` có tính sẵn sàng cao, nhưng chi phí gọi mô hình của `beta` đắt hơn 40% (`cost_per_call = 1.4` so với `1.0` của `alpha`). Nếu sự cố kéo dài nhiều ngày, ngân sách vận hành LLM sẽ bị thâm hụt nghiêm trọng. Ngoài ra, ở G2, p95 của lượt thành công vẫn chạm mức **10.92 s** do một số request phải chờ probe ở HALF-OPEN.

**Hướng khắc phục:**
1. Áp dụng chiến lược **Cost-aware Traffic Splitting**: Khi failover, ưu tiên điều phối một phần các request chat đơn giản (không cần tool calling) sang `gamma-mini` (chi phí chỉ 0.2, rẻ hơn 7 lần) để giảm tải cho `beta`.
2. Tối ưu hóa thời gian probe của Circuit Breaker: Điều chỉnh `half_open_max_probes` linh hoạt theo tỷ lệ lưu lượng thực tế thay vì cố định bằng 2 để tăng tốc độ phát hiện hồi phục.

---

## 8. Cache (Part G)

Số liệu từ `reports/cache/*.json`:

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9120 | 0.9645 | 0.635 | 174 | 0 | 6 | 0.927 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| `similarity_threshold` | 0.92 | Ngưỡng 0.80 quá lỏng gây ra 138–181 `wrong_answers` do trả nhầm các câu hỏi khác (`other_question`). Nâng lên 0.92 loại bỏ hoàn toàn câu trả lời sai (`wrong_answers = 0`) trong khi vẫn duy trì `cache_hit_ratio = 63.5%` (> 50%). |
| `ttl_s` | 45 s | Đảm bảo tính tươi mới cho dữ liệu. Với TTL = 45 s, số câu trả lời quá hạn khi giá cước vận chuyển đổi không báo trước được khống chế ở mức 6 câu (`outdated_answers = 6` <= 15). Nếu để TTL = 300 s, con số này sẽ vọt lên 95 câu. |
| `max_stale_s` | 300 s | Cứu vãn tính sẵn sàng trong thời gian 120 s toàn bộ provider sập (t=240..360). Cho phép phục vụ 174 câu trả lời stale-if-error, đưa `faq_availability` từ 86.34% lên **96.45%** (vượt chỉ tiêu >= 93%). |

### Thí nghiệm:

1. **Ngưỡng tương đồng:**
   - Khi `similarity_threshold = 0.80`: `cache_hit_ratio = 0.647`, nhưng phát sinh tới **138 câu trả lời sai** (`wrong_answers_by_reason: {'other_question': 138}`), cost/turn = 0.971. Nguyên nhân do các câu hỏi có nghĩa khác nhau (như "Thời gian hoàn tiền" vs "Chính sách đổi trả") có vector embedding khá gần nhau (> 0.80) dẫn đến trả nhầm đáp án.
   - Khi `similarity_threshold = 0.92`: `wrong_answers = 0`, hit ratio đạt **0.635**, cost/turn tối ưu nhất ở mức **0.927**.
   - Khi `similarity_threshold = 0.97`: `wrong_answers = 0`, nhưng do quá khắt khe nên hit ratio giảm xuống **0.552**, làm cost/turn tăng lên **0.984**.
2. **TTL đối với thay đổi không báo trước:**
   - Với `ttl_s = 30`: `outdated_answers = 2`, hit ratio = 0.592.
   - Với `ttl_s = 45`: `outdated_answers = 6`, hit ratio = 0.635.
   - Với `ttl_s = 300`: `outdated_answers = 95` (vi phạm tiêu chuẩn <= 15).
   - *Giải thích:* Webhook chỉ hoạt động khi CMS chủ động phát thông báo cập nhật (như sự kiện ở t=150). Khi đối tác thứ 3 (như đơn vị vận chuyển) tự ý đổi biểu phí ở t=480 mà không gửi webhook, cache hoàn toàn không hay biết. Do đó, **TTL là cơ chế duy nhất** giới hạn thời gian sống tối đa của các thông tin cũ trước khi bắt buộc phải gửi request revalidate.
3. **Stale-if-error:**
   - Khi đặt `max_stale_s: 0` (tắt stale-if-error): Trong 120 s mọi provider ngừng hoạt động, `stale_served = 0` và `faq_availability` sụt giảm xuống còn **86.34%** (không đạt mức yêu cầu >= 93%).
   - Khi bật `max_stale_s: 300`: Hệ thống phục vụ thành công **174 câu trả lời stale**, kéo `faq_availability` lên **96.45%**.

### Trả lời câu hỏi:
- *Vì sao câu hỏi cá nhân không được cache, kể cả khi cache theo từng user?* Dữ liệu cá nhân (trạng thái đơn hàng, số dư tài khoản, địa chỉ) biến động theo thời gian thực (stateful). Nếu cache, người dùng có thể nhận thông tin đơn hàng đã cũ. Nguy hiểm hơn, nếu logic phân tách user bị lỗi (hoặc đụng độ vector), hệ thống có nguy cơ làm lộ thông tin nhạy cảm của khách hàng này cho khách hàng khác (`other_user`), vi phạm nghiêm trọng luật bảo vệ quyền riêng tư (GDPR).
- *Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?* Ta phải đặt metric cảnh báo dựa trên **tỷ lệ lỗi của Backend/Provider (Upstream Error Rate / Breaker State)** hoặc tỷ lệ phục vụ dữ liệu cũ **`stale_served_ratio`**, thay vì chỉ dựa vào `user_perceived_availability`. Khi `stale_served > 0` hoặc Circuit Breaker chuyển sang `OPEN`, hệ thống giám sát phải kích hoạt cảnh báo ngay lập tức để kỹ sư on-call can thiệp xử lý provider trước khi cửa sổ `max_stale_s` hết hạn.
