# Lab 25B — Game Day: Retry Storm, Circuit Breaker, Cache, Idempotency & Error Budget

**AICB-P2T3 · Ngày 25 · Circuit Breakers, Caching & Reliability for Production Agents**

> Bạn vào vai **on-call** của ShopAgent: bản v1 đang chạy production vừa sập vì một retry storm.
> Bạn phải thay nó bằng một client chịu lỗi, rồi chứng minh bằng số rằng bản sửa chịu được 6 "băng sự cố"
> (incident tape) dựng lại theo các kiểu sự cố thật — và viết postmortem.

**Lab độc lập.** Không cần làm hay chạy lab nào trước, kể cả các repo `K3/K4-Day25-Track3-Reliability-Agent`.
"Hệ thống đã sập" là bối cảnh có sẵn trong repo này: provider LLM, traffic và sự cố đều do trình mô phỏng
trong `src/gameday_lab/sim/` tạo ra. Không cần API key, không cần Docker. Python ≥ 3.10 + `pyyaml` + `pytest`.

---

## 1. Bối cảnh

**ShopAgent** là agent CSKH xử lý yêu cầu hoàn tiền. Mỗi lượt của user gồm 2 LLM call:

1. `tool` — model quyết định gọi tool (cần **tool calling**),
2. `chat` — model viết câu trả lời cho user.

User chờ tối đa **30 giây** cho cả lượt. Có 4 deployment:

| Deployment          | Model family          |   Tool calling   | Chi phí / call | Năng lực          | Rate limit tenant |
| ------------------- | --------------------- | :--------------: | --------------: | ------------------- | ----------------: |
| `alpha` (primary) | A-large               |       có       |             1,0 | 18 slot, ~1,2 s/req |            20 rps |
| `alpha-eu`        | A-large (cùng model) |       có       |             1,0 | 16 slot, ~1,5 s/req |            10 rps |
| `beta`            | B-large               |       có       |             1,4 | 30 slot, ~1,8 s/req |            16 rps |
| `gamma-mini`      | C-mini                | **không** |             0,2 | 40 slot, ~0,6 s/req |            30 rps |

Bản v1 đang chạy production là `NaiveClient` (`src/gameday_lab/naive_client.py`): **3 tầng retry chồng nhau**
(app × gateway × SDK = tối đa **36 attempt / call**), delay cố định, không breaker, không fallback.
Tuần trước nó đã biến một sự cố 3 phút của provider thành **5 phút sập hoàn toàn**. Nhiệm vụ của bạn: thay nó.

```bash
make baseline   # xem v1 thất bại trên cả 6 băng
```

## 2. Mục tiêu học tập

Sau lab, bạn làm được:

1. **Phân loại lỗi** (429 rate limit ≠ 429 hết quota ≠ 529 ≠ 400 ≠ timeout) và phản ứng đúng cho từng loại.
2. Retry **đúng cách**: chỉ lỗi transient, full jitter, tôn trọng `retry-after`, **retry budget**, đúng 1 tầng.
3. Viết **circuit breaker sliding-window** an toàn dưới tải đồng thời: theo tỷ lệ lỗi, bắt call chậm, giới hạn probe ở HALF-OPEN.
4. Xây **fallback ladder có kiểm tra tương thích** (không gửi tool call sang model không hỗ trợ tool), spillover khi bị rate limit.
5. Làm workflow có side effect đạt **exactly-once effect**: idempotency key + checkpoint + resume sau crash.
6. Viết **burn-rate alerting** đa cửa sổ (SRE Workbook) và tính **error budget**.
7. Viết một **postmortem blameless** có số liệu.
8. Dùng **semantic cache có kiểm soát** (scope, ngưỡng, TTL, invalidation) và **stale-if-error** để câu hỏi
   FAQ vẫn được trả lời khi mọi provider sập — mà không trả lời sai.

## 3. Cài đặt và chạy

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -e ".[dev]"                                 # hoặc tối thiểu: pip install pyyaml pytest

make test        # ở starter: phần lớn test FAIL — đó là việc của bạn
make baseline    # client naive trên 6 băng sự cố
make gameday     # client của bạn (sau khi làm Part A–C)
make compare     # naive vs của bạn, cạnh nhau
make workflows   # Part D
make slo         # Part E
make cache       # Part G: không cache vs cache của bạn trên băng C1
make plot        # hình cho postmortem (cần: pip install -e ".[plot]")
make grade       # điểm tự động (88/100) -> reports/scoreboard.json
```

Không có `make`? Chạy trực tiếp: `python scripts/run_gameday.py --client both`, `python -m pytest -q`, …

## 4. Môi trường mô phỏng (đọc kỹ — không sửa)

- **Thời gian ảo.** Simulator (`src/gameday_lab/sim/`) là discrete-event: 8–10 phút sự cố chạy trong ~0,3 giây và
  **lặp lại được** với cùng seed. Mọi hàm của bạn nhận `now` (giây ảo). **Không** dùng `time.time()` / `time.sleep()`.
- **Provider là hàng đợi.** Mỗi deployment có `servers` slot xử lý FIFO. Request phải chờ quá 10 s thì bị **529**.
  Khi client timeout, **server vẫn tiếp tục xử lý** request đó (công sức bị phí). Đây là cơ chế khiến retry storm
  biến thành **metastable failure**: provider đã hồi phục nhưng vẫn nghẹt vì chính lượng retry.
- **Rate limit** theo cửa sổ 1 giây, trả `429 rate_limit_exceeded` kèm `retry_after`.
- **Client điều khiển bằng sự kiện.** Simulator gọi `start(call, now)` cho mỗi LLM call, rồi `on_success(...)` hoặc
  `on_error(...)` cho mỗi attempt. Bạn trả về `Send(provider, timeout_s, delay_s, tag)` hoặc `GiveUp(reason)`.
  Xem `src/gameday_lab/sim/contracts.py`.

| Lỗi                         | `status` | `code`                                       | `retry_after` |
| ---------------------------- | ---------- | ---------------------------------------------- | --------------- |
| Hết rate limit              | 429        | `rate_limit_exceeded`                        | có (giây)     |
| Hết quota / credit          | 429        | `insufficient_quota`                         | không          |
| Quá tải / chết            | 529 / 503  | `overloaded_error` / `service_unavailable` | không          |
| Lỗi server                  | 500        | `api_error`                                  | không          |
| Request sai (bug của mình) | 400        | `invalid_request_error`                      | không          |
| Client hết kiên nhẫn      | 0          | `client_timeout`                             | không          |

**Metric** (in ra sau mỗi băng): `availability` (tỷ lệ lượt user thành công), `latency_p95_s` (của lượt thành công),
`amplification` (attempt / logical call), `retry_ratio` (retry *cùng provider* / call), `cost_per_good_turn`,
`recovery_s` (từ lúc provider hồi phục đến khi 95% call thành công trong 30 s), `primary_share_after_restore`,
`retry_after_violations`, `client_error_retries`, `incompatible_routes`, `unneeded_failovers`.
Băng caching (Part G) có thêm `faq_availability`, `cache_hit_ratio`, `stale_served`, `wrong_answers`
(+ `wrong_answers_by_reason`) và `outdated_answers`.

## 5. Sáu băng sự cố

| Băng                            | Chuyện gì xảy ra                                                                                    | Phải đạt                                                                                              |
| -------------------------------- | ------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| **G1** rate-limit spike    | Traffic ×2,5 trong 3 phút; tenant`alpha` chỉ được 12 rps                                       | availability ≥ 98%,**0** vi phạm `retry-after`, amplification ≤ 1,6                           |
| **G2** overload storm      | `alpha` mất 75% GPU trong 3 phút (529 + hàng đợi dài)                                          | availability ≥ 95%, amplification ≤ 1,6,**recovery ≤ 30 s**, ≥ 80% traffic quay lại `alpha` |
| **G3** brown-out           | `alpha` trả 500 cho 30% request trong 3 phút                                                       | availability ≥ 98%,**retry_ratio ≤ 0,10**                                                        |
| **G4** slow region         | `alpha` vẫn trả lời nhưng chậm thêm 5 s, **không có lỗi nào**                        | availability ≥ 97%,**p95 ≤ 10 s**, quay lại `alpha` sau sự cố                               |
| **G5** bad deploy → quota | 60% request bị malformed (bug deploy) trong 2 phút; sau đó account A**hết quota**           | **0** retry trên lỗi 400, ≤ 10 failover không cần thiết, availability ≥ 78%                 |
| **G6** provider outage     | Cả provider A sập (`alpha` 503, `alpha-eu` treo); `beta` hạ rate limit vì ai cũng dồn sang | availability ≥ 95%,**0** tool call gửi sang model không hỗ trợ tool                           |

Khi chấm, giảng viên dùng **bản băng ẩn** (lệch thời điểm, đổi cường độ) và nhiều seed — đừng hard-code thời gian.

Băng thứ bảy, **C1** (Part G), nằm riêng trong `configs/scenarios_cache.yaml` — xem Part G.

## 6. Nhiệm vụ

Thời lượng thiết kế: **4 giờ** = 2 giờ core trên lớp (A–C) + 2 giờ cho G, D, E và report F (report hoàn thiện ở nhà
nếu cần). Làm Part G ngay sau Part C — băng C1 dùng `ResilientClient` của bạn.

### Part A — Phân loại lỗi & retry (12 điểm · ~25 phút)

File: `src/gameday_lab/errors.py`, `src/gameday_lab/retry.py` · Test: `tests/test_errors_retry.py`

- `classify()`, `is_retryable()`, `counts_toward_breaker()`, `failover_helps()`.
- `FullJitterBackoff.delay()` — full jitter; nếu có `retry_after` thì **không bao giờ** retry sớm hơn.
- `RetryBudget` — token bucket: mỗi request gửi lần đầu nạp `ratio` token, mỗi retry tiêu 1 token, hồi `min_per_s`/giây.

### Part B — Circuit breaker sliding-window (12 điểm · ~40 phút)

File: `src/gameday_lab/breaker.py` · Test: `tests/test_breaker.py`

- Trip theo **tỷ lệ lỗi** trên `window_size` kết quả gần nhất, chỉ khi đã có `min_calls` kết quả.
- Call thành công nhưng chậm (≥ `slow_call_s`) là dấu hiệu xấu → trip theo `slow_rate_threshold`.
- HALF-OPEN: tối đa `half_open_max_probes` request **đang bay**; đủ `half_open_successes` → CLOSED; probe hỏng/chậm → OPEN.
- Mỗi request được cấp một `Permit`; kết quả mang `generation` cũ (về sau khi state đã đổi) thì **bỏ qua**.

### Part C — ResilientClient (30 điểm · ~50 phút)

File: `src/gameday_lab/client.py` · Test: `tests/test_client.py` (6đ) + 6 băng × 4đ

- `ladder(task)`: thứ tự fallback, **lọc theo khả năng** (tool call chỉ đi model có tool calling).
- `start` / `on_success` / `on_error` / `on_cancel`: ghép Part A + B. Gợi ý quyết định trong docstring.
- Tinh chỉnh khối `client:` trong `configs/gameday.yaml` và **giải thích** trong report.
- Chạy `make compare` cho tới khi 6/6 băng PASS.

### Part D — Workflow hoàn tiền exactly-once (12 điểm · ~30 phút)

File: `src/gameday_lab/workflows/executor.py` · Test: `tests/test_executor.py` (4đ) + `make workflows` (8đ)

- Workflow 5 bước: `classify_request → lookup_order → issue_refund → send_email → write_summary`.
- Payment có thể đã chạy nhưng response bị mất (`ServiceTimeout`), hoặc **worker chết ngay sau khi tiền đã đi** (`WorkerCrash`).
- Cần: idempotency key **ổn định** theo (session, step) + checkpoint vào `DurableStore` **sau** khi step thành công + resume.
- Đạt: 0 hoàn tiền trùng, 0 email trùng, 0 hoàn tiền bị mất, ≥ 99% hoàn tất, `wasted_llm_per_crash ≤ 0,6`.

### Part E — Burn-rate alerting & error budget (12 điểm · ~30 phút)

File: `src/gameday_lab/slo/alerting.py` · Test: `tests/test_alerting.py` (4đ) + `make slo` (4đ) + dữ liệu ẩn (4đ)

- SLO availability 99,5% / 30 ngày. Dữ liệu: `data/sli_30d.csv` (43.200 phút) + nhãn sự cố `data/sli_30d_labels.json`.
- Luật: page 14,4× (1h & 5m), page 6× (6h & 30m), ticket 1× (3d & 6h). Dùng **prefix sum** để tính nhanh.
- Đạt: page đúng mọi sự cố lớn trong thời hạn, ticket cho rò rỉ chậm (không page), **0 false page** cho các blip.
- `error_budget_report()`: budget đã tiêu bao nhiêu, có phải đóng băng release không.

### Part F — Report & postmortem (12 điểm · ~25 phút trên lớp)

Copy `reports/REPORT_TEMPLATE.md` → `reports/REPORT.md` và điền đủ, kể cả mục 8 (cache). Trọng tâm: **postmortem
blameless cho băng G2** với client naive (dùng `make plot` + CSV trong `reports/gameday/`).

### Part G — Semantic cache & stale-if-error (10 điểm · ~40 phút)

File: `src/gameday_lab/cache.py` · Test: `tests/test_cache.py` (4đ) + băng C1 (6đ) · mục 8 của report tính vào Part F

60% lượt user là câu hỏi FAQ ("Chính sách hoàn tiền thế nào?", "Phí ship đi tỉnh bao nhiêu?"). Mỗi lượt FAQ là **một**
call `chat` mang `call.request` (`text`, `user_id`, `personal`, `prompt_version`, `embedding`). Khi call thành công,
simulator đặt câu trả lời vào `call.answer` (`Answer` có `prompt_version` và `sources` — các tài liệu nguồn). Client
được trả lời từ cache bằng action mới `Serve(answer, stale)`. Xem `src/gameday_lab/sim/contracts.py`.

- `SemanticCache` — `lookup` / `store` / `invalidate`: tìm theo cosine **trong cùng scope** (`prompt_version`) với
  ngưỡng `similarity_threshold`; câu hỏi cá nhân (`request.personal`: đơn hàng, tiền hoàn, điểm của *chính* user)
  **không** nằm trong allowlist; entry còn tươi trong `ttl_s`, sau đó là cửa sổ stale `max_stale_s`; invalidate theo
  `source`; tối đa `max_entries` (LRU).
- `CachingClient` bọc `ResilientClient` của bạn: hit còn tươi → `Serve` (không gọi provider); miss → provider, rồi lưu
  `call.answer`; provider bỏ cuộc (`GiveUp`) → trả câu trả lời cũ nếu còn trong cửa sổ stale (**stale-if-error** —
  bậc 3 "cached response" của fallback ladder); webhook `on_content_update(source)` → invalidate.
- Băng **C1_faq_outage** (`make cache`): t=150 chính sách hoàn tiền đổi (**có** webhook) · t=240–360 **mọi** provider
  sập · t=420 deploy prompt v2 · t=480 bảng phí ship đổi **không báo trước**. Đạt khi:
  `wrong_answers = 0` (không trả nhầm câu hỏi khác, không lộ dữ liệu của user khác, không dùng câu trả lời của prompt cũ
  hay của tài liệu đã được báo là đổi), `outdated_answers ≤ 15` (thay đổi không báo trước — chỉ TTL giới hạn được),
  `faq_availability ≥ 93%`, `cache_hit_ratio ≥ 50%`.
- Khối `cache:` trong `configs/gameday.yaml` của starter **cố ý chưa tốt**. Chạy `make cache`, đọc
  `wrong_answers_by_reason`, chỉnh từng tham số và giải thích bằng số ở mục 8 của report.

`tests/test_cache.py` dùng client giả nên làm được ngay; băng C1 cần `ResilientClient` (Part C) đã chạy.
Nếu Part C chưa xong thì vẫn lấy được 4 điểm test của Part G.

## 7. Quy tắc

- **Không sửa**: `src/gameday_lab/sim/`, `naive_client.py`, `workflows/services.py`, `workflows/runner.py`,
  `workflows/naive_executor.py`, `slo/data.py`, `slo/scoring.py`, `configs/scenarios.yaml`,
  `configs/scenarios_cache.yaml`, `tests/`. Grader dùng bản riêng của các file này.
- **Được sửa**: các file có `TODO`, khối `client:` và `cache:` trong `configs/gameday.yaml`, thêm file/test của bạn.
- Không hard-code thời điểm sự cố hay tên băng trong client. Không đọc state nội bộ của simulator.

## 8. Nộp bài

Nộp bằng **một repo GitHub ở chế độ public** (không nộp zip).

**Quy tắc đặt tên repo:** `Track3-Day25-<MSV>-<Họ-Và-Tên>`

- `<MSV>`: mã số sinh viên của bạn.
- `<Họ-Và-Tên>`: viết **không dấu**, mỗi từ viết hoa chữ cái đầu, nối bằng dấu gạch ngang `-` (không dùng khoảng trắng).
- Ví dụ: sinh viên Nguyễn Văn An, MSV `20260123` → repo `Track3-Day25-20260123-Nguyen-Van-An`.

**Repo gồm:** code đã hoàn thành, `configs/gameday.yaml`, `reports/REPORT.md` (+ hình), `reports/scoreboard.json`
từ `make grade`. Thư mục `reports/` **không** bị gitignore — hãy commit cả nó (lưu ý `make clean` xóa hình và CSV
trong đó).

**Cách nộp:**

1. Tạo repo public đúng tên trên (hoặc fork repo này rồi đổi tên theo quy tắc).
2. Push bài làm lên nhánh `main`.
3. Ghi **commit hash** nộp bài vào dòng `Commit:` đầu `reports/REPORT.md`, và ghi **họ tên + MSV** ở dòng `Họ tên / nhóm`.
4. Nộp **link repo** (ví dụ `https://github.com/<username>/Track3-Day25-20260123-Nguyen-Van-An`) lên kênh nộp bài của lớp.

Grader chạy: `pip install -e ".[dev]" && make grade` trên máy sạch.

## 9. Rubric (100 điểm)

| Phần                       | Điểm | Chấm thế nào                                                           |
| --------------------------- | -----: | ------------------------------------------------------------------------- |
| A. Phân loại lỗi & retry |     12 | `tests/test_errors_retry.py` (tỷ lệ test pass)                        |
| B. Circuit breaker          |     12 | `tests/test_breaker.py`                                                 |
| C. ResilientClient          |     30 | `tests/test_client.py` (6) + 6 băng × 4 (băng ẩn, nhiều seed)      |
| D. Workflow exactly-once    |     12 | `tests/test_executor.py` (4) + 500 workflow có crash (8)               |
| E. Alerting & error budget  |     12 | `tests/test_alerting.py` (4) + dữ liệu public (4) + dữ liệu ẩn (4) |
| G. Cache & stale-if-error   |     10 | `tests/test_cache.py` (4) + băng C1 (6, băng ẩn, nhiều seed)        |
| F. Report & postmortem      |     12 | chấm tay — xem`docs/RUBRIC.md` (gồm 2 điểm cho mục 8, cache)      |

88 điểm tự động (`make grade`) + 12 điểm chấm tay.

## 10. Lỗi thường gặp

| Lỗi                                                               | Băng / test bắt được                                           |
| ------------------------------------------------------------------ | ------------------------------------------------------------------- |
| Breaker đếm cả lỗi 400 (`except Exception`)                  | G5 (`unneeded_failovers`), test_client                            |
| Retry lỗi 400 hoặc fail over khi gặp 400                        | G5 (`client_error_retries`)                                       |
| Bỏ qua`retry-after`, retry ngay vào provider đang rate limit  | G1, G2                                                              |
| Gửi tool call sang`gamma-mini`                                  | G6 (`incompatible_routes`)                                        |
| Không có slow-call / không có breaker                          | G4 (p95)                                                            |
| Timeout quá dài (vd. 30 s)                                       | G6 (treo 30 s mới fail over)                                       |
| HALF-OPEN không giới hạn probe                                  | test_breaker                                                        |
| Idempotency key sinh ngẫu nhiên mỗi lần gọi (`uuid4()`)     | Part D: hoàn tiền trùng                                          |
| Ghi checkpoint**trước** khi chạy step                     | Part D: hoàn tiền bị mất                                        |
| Bắt`WorkerCrash` trong executor                                 | test_executor                                                       |
| Alert theo ngưỡng 5 phút                                        | Part E: false page trên blip                                       |
| Ngưỡng similarity quá lỏng (vd. 0,80)                          | C1:`wrong_answers` (`other_question`)                           |
| Cache câu trả lời cá nhân                                     | C1:`other_user` — lộ đơn hàng của người khác; test_cache |
| Scope thiếu`prompt_version`                                     | C1:`old_prompt`                                                   |
| Bỏ qua webhook`on_content_update`                               | C1:`ignored_invalidation`                                         |
| TTL quá dài                                                      | C1:`outdated_answers`                                             |
| Không có stale-if-error, hoặc`max_stale_s` ngắn hơn sự cố | C1:`faq_availability`                                             |

## 11. FAQ

**Sao availability ở G5 chỉ cần ≥ 78%?** Trong 2 phút bad deploy, 60% request bị malformed — không client nào cứu
được request sai. Việc đúng là *không làm tệ thêm*: không retry, không fail over vô ích.

**Retry budget có được chấm trên băng không?** Gián tiếp (G3 `retry_ratio`) và qua unit test. Trong report có một
thí nghiệm bật/tắt budget — hãy giải thích kết quả.

**Lab có phần caching không?** Có — Part G (10 điểm, bắt buộc): semantic cache có scope, ngưỡng, TTL, invalidation
và stale-if-error, chấm trên băng C1. Prompt caching phía provider (giảm giá token đầu vào) không mô phỏng ở đây —
xem bài giảng Day 25.

**Sao `faq_availability` ở C1 không lên được 100%?** Trong 2 phút mọi provider sập, câu hỏi cá nhân không được trả lời
từ cache, nên luôn thất bại. Lượt không phải FAQ (cần tool call) cũng thất bại, vì vậy `availability` tổng thấp hơn
`faq_availability`.

## 12. Đọc thêm

- Brooker, *Exponential Backoff And Jitter* (AWS Architecture Blog, 2015).
- Google SRE Book, *Handling Overload*; SRE Workbook ch. 5, *Alerting on SLOs*.
- Bronson et al., *Metastable Failures in Distributed Systems* (HotOS 2021).
- Google Cloud incident report 12/06/2025 (Service Control, thiếu randomized backoff); OpenAI status write-up 03/02/2026.
- Stripe API docs, *Idempotent requests*.
- RFC 5861, *HTTP Cache-Control Extensions for Stale Content* (`stale-while-revalidate`, `stale-if-error`).
- *vCache: Verified Semantic Prompt Caching* (arXiv:2502.03771).
