# Lab 25B — Report

**Họ tên / nhóm:** Nguyễn Gia Khánh - 2A202602851  **Commit:** c8ded89  **Điểm tự động (`make grade`):** 88.0 / 88 (84/84 public test + 4/4 instructor hidden dataset)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

---

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | **PASS** |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 11.911 | 1.059 | 0.008 | 2.161 | 0.0 | **PASS** |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.738 | 1.058 | 0.034 | 2.141 | 0.0 | **PASS** |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | **PASS** |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | **PASS** |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.018 | 2.196 | 0.0 | **PASS** |

---

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Đủ lớn cho median của beta (1.8s) và alpha (1.2s), nhưng đủ nhỏ để không cạn 30s turn deadline; giúp G6 không bị treo khi alpha-eu đơ, p95 giữ ở 5.961s. |
| max_attempts_per_provider | 2 | Cho phép thử lại 1 lần trên cùng provider cho lỗi tạm thời; ngăn chặn việc cố retry nhiều lần khi provider đang chết (ở G2 naive retry 36 lần gây ampl=9.904). |
| backoff_base_s / cap_s | 0.25 / 4.0 | Base 0.25s kết hợp full jitter giúp giãn cách request tránh thundering herd; cap 4.0s đảm bảo retry không vượt quá thời gian user chờ. |
| retry_budget_ratio | 0.1 | Giới hạn retry tối đa ~10% tổng lượng request mới; giúp G3 đạt retry_ratio=0.034 <= 0.10 và dập tắt retry storm ở G2. |
| breaker.window_size / min_calls | 20 / 10 | Window 20 cuộc gọi gần nhất đủ nhạy để phát hiện sự cố nhanh nhưng không bị nhiễu bởi vài lỗi lẻ tẻ vào ban đêm (cần ít nhất 10 cuộc gọi mới tính tỷ lệ). |
| breaker.failure_rate_threshold | 0.5 | Trip sang OPEN khi 50% cuộc gọi hỏng; giúp G2 và G6 ngắt mạch kịp thời khi lỗi 529/503 tăng cao. |
| breaker.slow_call_s | 5.0 | Ngưỡng bắt các cuộc gọi chậm (bình thường alpha ~1.2s, khi chậm thêm 5s là ~6.2s); giúp G4 phát hiện vùng chậm và trip breaker, kéo p95 từ 13.764s xuống 5.562s. |
| breaker.open_duration_s | 10.0 | Giữ breaker mở 10s trước khi cho phép thăm dò ở HALF-OPEN, cho provider đủ thời gian giải phóng hàng đợi mà không bị dội tải liên tục. |
| breaker.half_open_max_probes | 2 | Chỉ cho tối đa 2 request probe đồng thời bay vào provider đang hồi phục, triệt tiêu nguy cơ thundering herd làm sập lại provider. |

Sơ đồ quyết định của `on_error`:

```
                           [Attempt Failed: ProviderError]
                                          │
                                 classify(err) -> kind
                                          │
        ┌───────────────────┬─────────────┴─────────────┬────────────────────────┐
        ▼                   ▼                           ▼                        ▼
  CLIENT_ERROR       QUOTA_EXHAUSTED              RATE_LIMITED          OVERLOADED / SERVER_ERROR
     (400)               (429)                       (429)                   / TIMEOUT (0)
        │                   │                           │                        │
  Release Permit      Release Permit              Release Permit          Record Breaker Failure
        │                   │                           │                        │
  GiveUp("client_error")  Cooldown 300s               Cooldown retry_after    tries < max_attempts AND
  [No retry, No failover]   │                           │                     budget allows AND fits deadline?
                            ▼                           ▼                        ├───────────────┬───────────────┐
                      Failover now                Failover now                   │ YES           │ NO
                            │                           │                        ▼               ▼
                       Next in ladder              Next in ladder           Retry same      Failover now
                                                                            provider with   to next in ladder
                                                                            Jitter Delay
```

---

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối:**  
   Khi giả lập breaker bị cấu hình sai (`breaker.min_calls: 1000000000`) và đặt `max_attempts_per_provider: 5`:
   - Trên **G2**: khi giữ budget, hệ thống vẫn duy trì `avail = 0.9996`, `amplification = 1.382`, `retry_ratio = 0.058`, `recovery = 0.0s`. Khi tắt budget hoàn toàn, hệ thống suy giảm nghiêm trọng: `avail = 0.9649`, `amplification = 1.672`, `retry_ratio = 0.491` (tăng gấp 8.5 lần) và `recovery = 30.0s`.
   - Trên **G3**: có budget cho `retry_ratio = 0.067`, không budget tăng lên `0.162` (vượt ngưỡng 0.10).
   - *Kết luận:* Ngay cả khi breaker hoạt động đúng, retry budget vẫn tối quan trọng vì breaker bảo vệ *từng provider riêng lẻ*, còn budget là *chốt chặn toàn cục* (global governor) giới hạn tổng tài nguyên retry của cả hệ thống, bảo vệ downstream khỏi các đợt bùng nổ tải khi nhiều provider đồng loạt chập chờn.

2. **`attempt_timeout_s: 30`:**  
   - Trên **G4**: p95 vẫn ở mức ~5.68s do alpha phản hồi trong ~6.2s và breaker bắt được slow-call.
   - Trên **G6**: khi timeout là 8s, `avail = 1.0000`, 0 give up. Khi tăng timeout lên 30s, `avail` tụt xuống **0.9488** (FAIL check min 95%), xuất hiện **108 lượt GiveUp("deadline")** và 14 lượt `no_provider_available`.
   - *Cơ chế:* Tại G6, `alpha-eu` bị treo vô thời hạn (hanging). Nếu timeout là 30s (bằng đúng deadline lượt của user), client sẽ kiên nhẫn chờ trọn 30s trên primary và cạn sạch thời gian của user, hoàn toàn mất cơ hội failover sang `beta`. Timeout đúng phải được tính dựa trên p99 latency kỳ vọng của model (~1.5–2s) cộng một biên an toàn vừa phải (8s), đủ để failover ít nhất 2–3 lần trước khi chạm turn deadline (30s).

3. **Bỏ lọc tool calling khỏi `ladder()`:**  
   - Chạy G6 khi có lọc: `availability = 1.0000`, `incompatible_routes = 0`.
   - Chạy G6 khi bỏ lọc: `availability = 0.8927`, `incompatible_routes = 256`.
   - *Góc nhìn Product Manager:* 256 khách hàng yêu cầu hoàn tiền đã nhận thông báo lỗi hệ thống và bị từ chối phục vụ một cách vô lý. Lý do là hệ thống đã "tiết kiệm sai chỗ" khi chuyển các cuộc gọi cần thực thi nghiệp vụ hoàn tiền (cần tool calling) sang model `gamma-mini` giá rẻ chỉ biết chat văn bản thuần túy. Việc thiếu kiểm tra tương thích tính năng này đã trực tiếp làm sụt giảm 11% độ tin cậy dịch vụ và gây thất thoát trải nghiệm khách hàng.

---

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid (cents) | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1,574,200 | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

**Vì sao chỉ idempotency key là chưa đủ, và chỉ checkpoint cũng chưa đủ?**
- Nếu chỉ có idempotency key mà không có checkpoint: Mỗi khi worker chết rồi khởi động lại, worker mới sẽ phải chạy lại toàn bộ từ bước 1 (classify, lookup order), gây lãng phí gọi LLM và tài nguyên (`wasted_llm_per_crash` tăng vọt).
- Nếu chỉ có checkpoint mà không có idempotency key: Khi lệnh thanh toán (`issue_refund`) đã gửi sang cổng thanh toán nhưng mạng bị rớt (`ServiceTimeout`), worker không nhận được kết quả nên chưa thể ghi checkpoint. Khi thử lại bước đó, nếu không có idempotency key, cổng thanh toán sẽ trừ tiền lần 2, gây hoàn tiền trùng (overpaid).
- **Nếu ghi checkpoint trước khi gọi payment:** Nếu worker chết ngay sau khi vừa ghi checkpoint mà chưa kịp gọi cổng payment, worker mới hồi sinh sẽ nhìn thấy checkpoint và tưởng rằng tiền đã được hoàn, lập tức chuyển sang gửi email. Hậu quả là khách hàng nhận email xác nhận nhưng tiền không bao giờ được chuyển (**lost refund** - lỗi nghiêm trọng nhất trong tài chính).

---

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | paged | page | 27 min |
| provider_outage | page | paged | page | 4 min |
| deploy_blip | none | paged (false) | none | — |
| slow_leak | ticket | not detected | ticket | 1337 min (~22.3h) |
| degraded_evening | page | paged | page | 291 min (~4.8h) |
| night_spike | none | paged (false) | none | — |
| small_bump | none | paged (false) | none | — |

**Error budget 30 ngày:**
- Tổng requests: 25,901,755
- Số lỗi cho phép (SLO 99.5%): 129,508.8
- Số lỗi thực tế: 55,618
- Availability thực tế: 99.7853%
- Budget đã tiêu: **42.95%**, còn lại: **57.05%**.
- **Quyết định:** **KHÔNG** đóng băng release (`freeze_releases = False`).
- **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?**  
  `small_bump` chỉ tiêu tốn một phần cực nhỏ (~0.16%) trong tổng error budget 30 ngày và tự kết thúc nhanh chóng trong 20 phút. Đánh thức kỹ sư on-call lúc nửa đêm hay làm gián đoạn ban ngày vì một sự cố thoáng qua không đe dọa SLO tháng sẽ gây mỏi mệt cảnh báo (alert fatigue), làm giảm độ nhạy bén khi có sự cố thảm họa thật sự. Luật đa cửa sổ (yêu cầu cả cửa sổ dài 1h và cửa sổ ngắn 5m cùng cháy) đã lọc bỏ thành công blip này.

---

## 6. Postmortem — Băng G2 với client naive (blameless)

*Dựa trên đồ thị `reports/gameday/G2_overload_storm.png` và số liệu `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):**  
Vào giây ảo t=120, deployment chính `alpha` bị mất đột ngột 75% năng lực xử lý (GPU) trong 180 giây. Do client v1 ([NaiveClient](file:///f:/VinUniversity/Schedule_Study_VinUniversity/Period_Two/Day10/K4-Track03-Day25-2A202602851-Nguyen-Gia-Khanh-Circuit-Breakers-Caching-Reliability/src/gameday_lab/naive_client.py)) sử dụng 3 tầng retry xếp chồng không có jitter, không có circuit breaker và không có retry budget, client đã gửi ồ ạt tới 163 attempts/s làm tắc nghẽn hàng đợi của provider. Sự cố này biến một lỗi suy giảm tài nguyên tạm thời thành một trạng thái hỏng hóc tự duy trì (metastable failure), khiến hệ thống mất thêm 155 giây sau khi GPU đã hồi phục mới có thể trở lại bình thường.

**Ảnh hưởng:**  
- 1,285 lượt user thất bại hoàn toàn (availability giảm xuống còn 45.60%).
- Sự cố kéo dài tổng cộng 335 giây (180s thời gian sự cố provider + 155s thời gian phục hồi).
- Tiêu tốn 34,475 lượt attempts (khuếch đại tải 9.904 lần so với mức thông thường), lãng phí $4,983 chi phí tính toán.

**Timeline (giây ảo — lấy từ CSV):**

| t (s) | Sự kiện | Bằng chứng (số liệu thực tế từ CSV) |
|---:|---|---|
| 115 | Trạng thái bình thường trước sự cố | `calls_ok = 11`, `calls_done = 11`, `attempts_alpha = 9` (tỷ lệ thành công 100%). |
| 120 | alpha mất 75% GPU (năng lực tụt từ 18 slots xuống ~4.5 slots) | `calls_ok = 7`, `attempts_alpha = 5`, hàng đợi FIFO bắt đầu tích lũy request. |
| 150 | Đỉnh điểm bùng nổ retry storm | `attempts_alpha = 163` attempts/s (gấp 18 lần bình thường), `calls_ok = 0`, 100% request hỏng do quá 10s bị trả 529 hoặc timeout. |
| 200 | Hệ thống kẹt cứng trong vòng lặp Metastable Failure | `attempts_alpha = 135` attempts/s, `calls_ok = 0`, client timeout nhưng server vẫn tiếp tục xử lý các request đã bị bỏ rơi. |
| 300 | alpha hồi phục 100% năng lực GPU phần cứng | `calls_ok = 0`, `calls_done = 5`, `attempts_alpha = 115` attempts/s. Hệ thống vẫn tê liệt hoàn toàn (0% success). |
| 455 | Hàng đợi rác dần được giải phóng | `calls_ok = 4`, `calls_done = 5`, `attempts_alpha = 13` attempts/s. Bắt đầu có phản hồi thành công đầu tiên. |
| 460 | Hệ thống chính thức phục hồi | `calls_ok = 9`, `calls_done = 12`, `attempts_alpha = 14`. Khôi phục tỷ lệ thành công >= 95% (mất 155s kể từ t=300). |

**Trigger vs Root cause:**  
- **Trigger:** Provider `alpha` mất 75% GPU phần cứng tại t=120.
- **Root cause:** Kiến trúc client v1 thiếu kiểm soát lưu lượng trầm trọng: 3 tầng retry xếp chồng cho phép tối đa 36 lần thử lại/call, không có exponential backoff với jitter, không có circuit breaker cắt tải khi lỗi, và không có retry budget.
- **Vì sao provider hồi phục mà hệ thống vẫn sập (Metastable Failure)?**  
  Server xử lý theo hàng đợi FIFO. Khi client timeout sau 8–10 giây, server *vẫn tiếp tục xử lý* request đó trong hàng đợi mà không hủy bỏ. Hàng triệu CPU/GPU cycles bị lãng phí để tính toán câu trả lời cho các request mà client đã hủy. Lượng retry liên tục dội vào với tốc độ >100 req/s khiến hàng đợi luôn đầy vượt quá 10s, kích hoạt mã 529 liên tục ngay cả khi phần cứng đã hoạt động bình thường trở lại.

**Điều gì giúp / Điều gì làm tệ hơn:**  
- *Làm tệ hơn:* Retry ngay lập tức với delay cố định; các tầng retry app × gateway × SDK nhân số lượng cuộc gọi lên gấp bội.
- *Giúp ích:* Triển khai ResilientClient với Circuit Breaker (ngắt request sang `alpha-eu` và `beta`), Retry Budget (chặn retry ở mức 10%), và Full Jitter Backoff.

**Action items:**

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế NaiveClient bằng ResilientClient có Circuit Breaker & Fallback Ladder | Prevent | Reliability Team | Chạy `make gameday` trong CI; G2 đạt `availability >= 95%` và `amplification <= 1.6`. |
| 2 | Triển khai Token-Bucket Retry Budget ở tầng client (ratio <= 0.1) | Mitigate | Client SDK Team | Unit test `test_budget_caps_retries` và `retry_ratio <= 0.05` trên Game Day G3. |
| 3 | Bổ sung cảnh báo Burn-rate đa cửa sổ phát hiện suy giảm SLO trong vòng 5 phút | Detect | SRE On-call | Chạy `make slo`; phát hiện sự cố overload storm trong < 5 phút với 0 false alarm. |

---

## 7. Một điểm yếu còn lại

**Tình huống:** Khi `alpha` và `alpha-eu` cùng gặp sự cố hoặc quá tải kéo dài, hệ thống failover dồn toàn bộ lưu lượng sang `beta`.
- **Hạn chế:** `beta` có chi phí cao hơn 40% (cost 1.4 vs 1.0) và độ trễ phục vụ chậm hơn (~1.8s vs 1.2s). Khi toàn bộ traffic đổ sang `beta`, chi phí vận hành tăng vọt và latency p95 của toàn hệ thống bị đẩy lên cao (ở G2 p95 lên tới 11.91s). Nếu cả 4 provider cùng gặp sự cố suy giảm mạng lưới diện rộng, client vẫn phải chịu độ trễ tích lũy qua các bậc fallback.
- **Giải pháp cải tiến:** 
  1. *Bulkhead & Concurrency Limiting:* Áp dụng thuật toán Vegas hoặc AIMD để giới hạn số lượng in-flight requests gửi sang `beta`, tránh làm sập luôn provider dự phòng đắt tiền.
  2. *Hedged Requests:* Khi request gửi sang `alpha` vượt quá p95 (ví dụ sau 3 giây chưa có phản hồi), chủ động gửi một request thăm dò song song sang `alpha-eu` hoặc `beta` và lấy kết quả của bên về trước, hủy bỏ bên còn lại để triệt tiêu tail latency.

---

## 8. Cache (Part G)

Số liệu từ `reports/cache/cached.json` (`make cache`):

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | **0.9124** | **0.9645** | **0.652** | **157** | **0** | **3** | **0.915** | **PASS** |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Ở ngưỡng 0.80 xuất hiện 175 lỗi `wrong_answers` do trả lời nhầm câu hỏi khác; ở ngưỡng 0.92 loại bỏ hoàn toàn lỗi nhầm (`wrong_answers = 0`) mà vẫn giữ `cache_hit_ratio = 65.2%`. |
| ttl_s | 60 | Giữ câu trả lời tươi mới; khi carrier đổi bảng phí ship không báo trước tại t=480, TTL 60s giới hạn `outdated_answers = 3` (thấp hơn nhiều so với mức tối đa cho phép là 15, so với 95 lỗi khi TTL=300s). |
| max_stale_s | 300 | Đủ dài để bao trùm đợt mất kết nối 120 giây (t=240–360s) của mọi provider; stale-if-error đã cứu được 157 câu hỏi, nâng `faq_availability` lên 96.45%. |

**Thí nghiệm:**

1. **Ngưỡng similarity:**  
   - Tại `0.80`: `hit = 0.659`, `wrong_answers = 175` (`wrong_reasons: {'other_question': 175}`), `cost/turn = 0.978`.  
   - Tại `0.92`: `hit = 0.652`, `wrong_answers = 0`, `cost/turn = 0.915`.  
   - Tại `0.97`: `hit = 0.579`, `wrong_answers = 0`, `cost/turn = 0.965`.  
   - *Phân tích:* Ở ngưỡng 0.80, các câu hỏi có embedding nằm gần nhau nhưng mang ý định hoàn toàn khác (ví dụ: "Thời hạn đổi hàng" vs "Chính sách bảo hành", hoặc "Phí giao hàng" vs "Điều kiện trả hàng") bị nhận diện nhầm, dẫn đến 175 câu trả lời sai tai hại. Ngưỡng 0.92 phân biệt chính xác ý định ngữ nghĩa trong khi vẫn gom được các câu hỏi diễn đạt khác nhau (paraphrase).

2. **TTL:**  
   - `ttl_s = 30s`: `outdated = 2`, `hit = 0.592`.  
   - `ttl_s = 60s`: `outdated = 3`, `hit = 0.652`.  
   - `ttl_s = 300s`: `outdated = 95`, `hit = 0.731` (vi phạm nghiêm trọng ngưỡng tối đa 15).  
   - *Lý do:* Sự kiện tại t=480 là `silent_updates` từ đối tác vận chuyển, hệ thống nội bộ không hề nhận được webhook. Vì không có tín hiệu chủ động để xóa cache, TTL là cơ chế an toàn duy nhất quy định thời điểm một câu trả lời cũ phải bị loại bỏ để revalidate với model.

3. **Stale-if-error:**  
   - Khi `max_stale_s = 0` (tắt stale-if-error): `faq_availability` tụt xuống **87.33%** (FAIL check min 93%), `stale_served = 0`.  
   - Khi `max_stale_s = 300`: `faq_availability` đạt **96.45%**, phục vụ thành công **157 lượt stale**.  
   - *Nhận xét:* Trong lúc toàn bộ 4 provider LLM sập hoàn toàn (t=240–360), việc trả lời người dùng một chính sách vừa lưu từ vài phút trước với cờ stale hữu ích hơn rất nhiều so với việc ném ra thông báo lỗi sập hệ thống.

**Trả lời:**

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**  
  Thứ nhất, dữ liệu cá nhân (trạng thái đơn hàng, số dư hoàn tiền, điểm tích lũy) là trạng thái biến động nhanh theo thời gian thực (real-time transactional state). Cache các câu hỏi này sẽ trả về trạng thái cũ (ví dụ đơn đã giao nhưng cache vẫn báo đang xử lý). Thứ hai và quan trọng nhất là rủi ro bảo mật thông tin (Data Leakage / PII): một sai sót nhỏ trong việc tính toán cache key hoặc prompt version có thể khiến câu trả lời chứa thông tin thẻ, địa chỉ nhận hàng của người này bị trả nhầm cho người khác (`other_user`).
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**  
  Nếu chỉ theo dõi tỷ lệ availability từ góc nhìn của user, on-call sẽ bị "ru ngủ" vì cache đang gánh tải. Để phát hiện sự cố provider kịp thời, cần alert dựa trên:
  1. **Upstream Provider Error Rate / Failure Rate:** Tỷ lệ lỗi 5xx/timeout khi cache miss gọi xuống LLM providers.
  2. **Stale Serving Ratio / Stale Responses Per Second:** Tỷ lệ phản hồi được trả về với cờ `stale=True`. Nếu tỷ lệ này vượt quá 5% tổng traffic trong 2 phút liên tiếp, hệ thống phải phát page alert ngay lập tức vì provider phía sau chắc chắn đang gặp sự cố.
